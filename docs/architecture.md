# Taqnote: arquitectura

Última actualización: 2026-10-08 · Relacionado: [roadmap](ROADMAP.md) · [plan de implementación](plan/implement-plan.md) · [ADRs](adr/)

Taqnote es un núcleo en TypeScript con adaptadores por plataforma. La app de escritorio (Tauri 2) aporta en Rust la captura de audio, la inferencia local y el filtro de hardware; la web aporta adaptadores del navegador, y Taqnote Cloud es un Worker sin estado. Este documento describe el estado objetivo, y cada sección indica en qué fase aparece.

## Vista general

```mermaid
flowchart LR
    subgraph clientes ["Clientes"]
        DESK["apps/desktop<br/>Tauri 2"]
        WEB["apps/web<br/>PWA, Fase 4"]
        UI["packages/ui<br/>React + Tailwind"]
    end
    subgraph nucleo ["packages/core (TypeScript)"]
        ORQ["Orquestador"]
        PORTS["Puertos"]
    end
    subgraph nativo ["src-tauri (Rust)"]
        CAP["Captura de micrófono y sistema"]
        VAD["VAD y bloques de ~30 s"]
        ASR["whisper.cpp"]
        HW["Mínimos y benchmark"]
    end
    subgraph externos ["Servicios externos"]
        OLL["Ollama"]
        API["OpenAI, Groq, Claude"]
        DEST["Notion, Microsoft Graph, Google APIs"]
        CLOUD["Taqnote Cloud<br/>Workers + D1, Fases 2 y 5"]
    end
    DESK --> UI
    WEB --> UI
    UI --> ORQ
    ORQ --> PORTS
    PORTS --> CAP
    CAP --> VAD
    PORTS --> ASR
    PORTS --> HW
    PORTS --> OLL
    PORTS --> API
    PORTS --> DEST
    PORTS --> CLOUD
```

## Principios de diseño

- **Núcleo agnóstico de plataforma.** `packages/core` no importa Tauri ni APIs del navegador; todo entra por puertos.
- **Rust solo para lo nativo.** Captura, VAD, inferencia local y filtro de hardware. La orquestación, los prompts y las exportaciones viven en TypeScript.
- **Local-first.** SQLite en escritorio e IndexedDB en web; ningún servidor guarda reuniones.
- **Motor por etapa.** Transcripción y minuta eligen su motor por separado: local, key propia o Taqnote Cloud.
- **Fallar sin perder datos.** El audio se escribe en bloques cerrados y cada bloque tiene estado.
- **Integrarse, no reemplazar.** Las tareas viven en las apps del usuario; Taqnote solo guarda la referencia de lo que exportó.

## Estructura del repositorio

```
taqnote/
├── apps/
│   ├── desktop/          Tauri 2: frontend React (src/) y backend Rust (src-tauri/)
│   ├── web/              PWA (Fase 4)
│   └── cloud/            Cloudflare Worker (Fase 5)
├── packages/
│   ├── core/             dominio, puertos, orquestador, prompts y exportadores
│   └── ui/               componentes React compartidos
├── spikes/               experimentos desechables (CAP-00)
├── fixtures/audio/       audios en español con transcripción de referencia
├── docs/
└── .github/workflows/
```

## Puertos y adaptadores

| Puerto | Responsabilidad | Escritorio | Web | Fase |
| --- | --- | --- | --- | --- |
| `AudioSource` | Entregar bloques de audio por canal | Rust: cpal y wasapi, vía eventos de Tauri | getUserMedia y getDisplayMedia con AudioWorklet | 1 / 4 |
| `Transcriber` | Bloque → segmentos con tiempos | whisper.cpp en Rust; OpenAI y Groq por HTTP | OpenAI, Groq o Cloud | 1 / 2 |
| `Summarizer` | Transcript → minuta estructurada | Ollama; luego API compatible con OpenAI | API compatible con OpenAI o Cloud | 1 / 2 |
| `MeetingStore` | Persistir reuniones, segmentos y minutas | SQLite (tauri-plugin-sql) | IndexedDB | 1 / 4 |
| `DocDestination` | Publicar la minuta | Markdown y Notion; luego Google Docs | Igual, con proxy si hay CORS | 1 / 2 |
| `TaskDestination` | Crear o actualizar tareas | Notion y Microsoft To Do; luego Lists y Google Tasks | Igual | 1 / 2 |
| `CalendarDestination` | Agendar seguimientos | Google Calendar | Igual | 2 |
| `AuthProvider` | Conectar cuentas y renovar tokens | OAuth con PKCE y loopback desde Rust | OAuth con PKCE en el navegador | 1 / 4 |
| `SecretStore` | Guardar keys y tokens | Keychain del sistema | Almacenamiento del navegador | 1 / 4 |

El contrato principal, en `packages/core`:

```ts
export type Channel = 'mic' | 'system'; // mic = el usuario, system = los demás

export interface AudioChunk {
  meetingId: string;
  channel: Channel;
  seq: number;
  startMs: number; // relativo al inicio de la reunión
  endMs: number;
  uri: string;     // WAV PCM 16 kHz mono
}

export interface Segment {
  id: string;
  chunkSeq: number;
  channel: Channel;
  startMs: number;
  endMs: number;
  text: string;
}

export interface Transcriber {
  readonly id: string; // 'local-whisper' | 'groq' | 'openai' | 'cloud'
  transcribe(chunk: AudioChunk, opts: { language: string; prompt?: string }): Promise<Segment[]>;
}

export interface Summarizer {
  readonly id: string;
  summarize(input: { segments: Segment[]; template: string; language: string }): Promise<Minute>;
}

export interface ActionItem {
  id: string;
  title: string;
  notes?: string;
  owner: { name: string | null; email?: string; isMe: boolean };
  due?: { date: string; time?: string; timeZone?: string }; // ISO 8601
  refs: string[];      // ids de segmentos
  minuteUrl?: string;  // enlace a la minuta exportada
}

// Todas las operaciones son idempotentes: con `previous`, actualizan en vez de crear.
export interface DocDestination {
  readonly id: 'markdown' | 'notion' | 'google-docs';
  publish(minute: Minute, previous?: ExternalRef): Promise<ExternalRef>;
}

export interface TaskDestination {
  readonly id: 'notion' | 'ms-todo' | 'ms-lists' | 'google-tasks';
  readonly assignsOthers: boolean; // false en To Do y Google Tasks
  upsert(item: ActionItem, previous?: ExternalRef): Promise<ExternalRef>;
}

export interface CalendarDestination {
  readonly id: 'google-calendar';
  schedule(event: FollowUp, previous?: ExternalRef): Promise<ExternalRef>;
}
```

## Pipeline de una reunión

```mermaid
sequenceDiagram
    participant R as Rust (captura)
    participant O as Orquestador (core)
    participant T as Transcriber
    participant DB as MeetingStore
    participant S as Summarizer
    participant E as Destinos
    R->>R: Captura mic y sistema en canales separados
    R->>R: VAD corta bloques de ~30 s y escribe WAV
    R-->>O: evento chunk_ready
    O->>DB: guarda el bloque (recorded)
    O->>T: transcribe(bloque, glosario)
    T-->>O: segmentos con tiempos
    O->>DB: guarda segmentos (transcribed)
    Note over O: Al terminar la reunión
    O->>S: transcript por ventanas y plantilla
    S-->>O: minuta en JSON con referencias a segmentos
    O->>O: valida el esquema y las referencias
    O->>DB: guarda el borrador
    Note over O: El usuario revisa y aprueba
    O->>E: exporta con clave de idempotencia
    E-->>O: id externo
    O->>DB: guarda la exportación
```

### Captura (CAP-01 a CAP-03)

El micrófono se captura con cpal y el audio del sistema con loopback de WASAPI mediante el crate `wasapi`, que soporta esa captura de forma explícita. Cada canal se convierte a 16 kHz mono y se guarda por separado: el canal `mic` es el usuario y el canal `system`, el resto de la reunión. El loopback puede no entregar datos mientras no suena nada, así que la grabadora rellena esos huecos con silencio según el reloj para no desalinear los tiempos.

El VAD corta los bloques en silencios con un objetivo de 30 s, un mínimo de 10 s y un máximo de 40 s. El mínimo coincide con el cobro mínimo de Groq por solicitud, y 30 s es el segmento para el que Groq optimiza Whisper Turbo. Cada bloque es un WAV PCM de 16 bits (unos 0,96 MB por 30 s) que se escribe en un archivo temporal y se renombra al cerrarse, así que un cierre inesperado solo deja bloques completos. El nombre del archivo lleva canal, secuencia y tiempos (`mic-0007-210000-240000.wav`), de modo que la recuperación puede reconstruir los bloques aunque la base de datos no los tenga.

### Transcripción (TRN-01 a TRN-03)

La transcripción local usa whisper.cpp a través de whisper-rs, que hoy se mantiene en Codeberg y ofrece las features `vulkan`, `cuda` y `metal`. Una cola procesa un bloque a la vez para no competir por la GPU. El idioma por defecto es `es` y el glosario se pasa como prompt inicial. Los modelos se descargan desde Hugging Face a la carpeta de datos de la app y se verifican con el SHA publicado.

La transcripción en la nube envía el mismo bloque a OpenAI o Groq, detrás del mismo puerto `Transcriber`. En escritorio, las llamadas HTTP pasan por tauri-plugin-http para evitar las restricciones de CORS del WebView.

### Minuta (MIN-01 a MIN-03, TAR-01)

El LLM devuelve JSON validado con Zod. Cada punto lleva `refs` a ids de segmentos, y el orquestador marca o descarta los puntos cuyas referencias no existen: es la principal defensa contra acuerdos inventados.

```ts
const Minute = z.object({
  titulo: z.string(),
  asistentes: z.array(z.string()),
  temas: z.array(z.object({ titulo: z.string(), resumen: z.string(), refs: z.array(z.string()) })),
  acuerdos: z.array(z.object({
    texto: z.string(),
    responsable: z.string().nullable(),
    fecha: z.string().nullable(), // ISO 8601
    refs: z.array(z.string()),
  })),
  pendientes: z.array(z.object({ texto: z.string(), refs: z.array(z.string()) })),
});
```

El resumen trabaja por ventanas de ~10 minutos y una fusión final desde la Fase 1, porque una reunión de una hora no cabe en el contexto de un modelo local de 8–14B. Con Ollama, el tamaño de contexto (`num_ctx`) se fija de forma explícita, porque el valor por defecto es corto.

La trazabilidad apunta al segmento del transcript y su tiempo. Si el usuario conserva el audio, además permite reproducir ese momento; por defecto el audio se borra tras transcribir (PRV-01).

### Exportación (EXP-01)

El render a Markdown es el canónico: es la salida de EXP-01 y la base de lo que se publica en Notion y Google Docs. Las tareas y los eventos salen por la capa de conectores.

## Conectores (INT-01 a INT-03)

Todas las integraciones comparten el modelo `ActionItem`, un enrutamiento por responsable y el registro de exportaciones; cada app es un adaptador de `DocDestination`, `TaskDestination` o `CalendarDestination`.

**Enrutamiento.** Los compromisos del usuario (detectados en su canal de micrófono o asignados al revisar) van a To Do o a Google Tasks, según su preferencia. Las tareas de otras personas van a Notion o a Lists, porque To Do y Google Tasks no asignan tareas a terceros. Si un punto tiene fecha y hora, también se propone un evento en Google Calendar.

El usuario confirma el destino de cada tarea en la revisión; nada se envía solo.

| Conector | API | Autenticación | Cuentas | Permiso | Restricción clave |
| --- | --- | --- | --- | --- | --- |
| Notion | REST con `Notion-Version: 2025-09-03` | Token interno (F1); OAuth con broker (F2) | Cualquier workspace | Páginas compartidas con la integración | Su endpoint de tokens exige el client secret |
| Microsoft To Do | Graph: `POST /me/todo/lists/{id}/tasks` | OAuth con PKCE, cliente público | Personal y de trabajo | `Tasks.ReadWrite` delegado | No asigna tareas a otras personas |
| Microsoft Lists | Graph: `POST /sites/{site-id}/lists/{list-id}/items` | OAuth con PKCE, cliente público | Solo de trabajo o escuela | `Sites.ReadWrite.All` delegado | Puede requerir aprobación del admin |
| Google Calendar | Calendar API v3: `events.insert` | OAuth con PKCE y loopback | Personal y Workspace | Scope de Calendar (sensible) | Verificación de Google antes de abrirlo al público |
| Google Tasks | Tasks API v1: `tasks.insert` | OAuth con PKCE y loopback | Personal y Workspace | Scope de Tasks | La fecha de vencimiento no guarda la hora |
| Google Docs | Drive API v3: subida con conversión | OAuth con PKCE y loopback | Personal y Workspace | `drive.file` | Solo accede a los archivos que crea Taqnote |

**Notion.** Una base puede tener varios data sources, y en la versión 2025-09-03 las páginas se crean con un parent de tipo `data_source_id`. Taqnote usa dos data sources relacionados: Minutas y Tareas. Los límites obligan a trocear: 2000 caracteres por texto, arreglos de hasta 100 elementos y payloads de hasta 1000 bloques y 500 KB. El tráfico promedio permitido es de 3 solicitudes por segundo por integración, y un 429 se reintenta respetando `Retry-After`.

**Microsoft To Do.** Cada tarea lleva `dueDateTime`, `reminderDateTime` con `isReminderOn` y un recurso vinculado (`linkedResources`) con el enlace a la minuta.

**Microsoft Lists.** Una lista con las columnas Título, Responsable, Fecha, Estado y Reunión. Al inicio el responsable se guarda como texto, porque una columna de persona exige resolver al usuario dentro del sitio.

**Google.** Tasks guarda solo la fecha de vencimiento, sin hora, así que cuando la hora importa se crea además un evento en Calendar. Un evento con invitados les envía invitaciones, por lo que agregar a otras personas requiere confirmación explícita. La minuta se sube a Drive con conversión a Google Docs, que importa Markdown.

### OAuth en escritorio (INT-02)

El flujo es authorization code con PKCE en el navegador del sistema. Rust abre un servidor efímero en `127.0.0.1` con un puerto libre, recibe el código y lo cambia por tokens, y los refresh tokens van al keychain (PRV-02). La tabla `connections` registra proveedor, cuenta y scopes, nunca tokens.

- **Microsoft:** registro de app con la plataforma "Mobile and desktop applications", cuentas de cualquier organización y cuentas personales, sin client secret. En redirects de loopback el puerto no cuenta. Microsoft recomienda `127.0.0.1` en vez de `localhost`, pero el portal solo acepta esa forma editando el manifiesto. El permiso de Lists se pide recién cuando el usuario activa ese conector.
- **Google:** cliente de tipo escritorio con redirect `http://127.0.0.1:<puerto>` y PKCE S256. Las apps instaladas no admiten autorización incremental, así que los scopes de Google se piden juntos según los servicios que el usuario active.

### Consentimiento y verificación (INT-03)

Microsoft recomienda que los usuarios solo puedan consentir apps de publishers verificados; en esos tenants, el resto requiere aprobación de un admin. To Do pide un permiso acotado (`Tasks.ReadWrite`), mientras que Lists pide `Sites.ReadWrite.All` y lo más probable es que necesite un admin. La verificación de publisher reduce esa fricción en cuentas de trabajo.

En Google, mientras la app está en modo de prueba admite hasta 100 usuarios de prueba y las autorizaciones vencen a los siete días, así que durante el desarrollo hay que reconectar cada semana. Para producción, los scopes sensibles de Calendar exigen verificación: página pública, política de privacidad en el mismo dominio verificado, justificación de cada scope y un video del flujo. Suele tardar de 3 a 5 días hábiles; sin ella, Google muestra una advertencia y limita a 100 los usuarios nuevos.

### Broker de OAuth de Notion (EXP-03, Fase 2)

Notion autentica su endpoint de tokens con el client secret, que no puede viajar dentro de una app de escritorio. Un endpoint del Worker guarda ese secreto, hace el intercambio y la renovación de tokens, y entrega los tokens a la app con un código de un solo uso, sin conservarlos.

### Idempotencia

`exports` registra cada elemento exportado: la minuta (`item_key = 'minute'`) y cada tarea (`item_key` = id de la tarea), con su id externo en el destino. Reexportar actualiza en vez de duplicar.

## Modelo de datos (SQLite)

Cada conexión activa `PRAGMA foreign_keys = ON`. Los tiempos de segmentos y bloques son milisegundos relativos al inicio de la reunión.

```sql
CREATE TABLE meetings (
  id          TEXT PRIMARY KEY,               -- UUID v7
  title       TEXT NOT NULL,
  started_at  TEXT NOT NULL,                  -- ISO 8601 UTC
  ended_at    TEXT,
  status      TEXT NOT NULL,                  -- recording | processing | draft | reviewed | exported | failed
  language    TEXT NOT NULL DEFAULT 'es',
  template    TEXT NOT NULL DEFAULT 'base',
  asr_engine  TEXT NOT NULL,                  -- local-whisper | groq | openai | cloud
  llm_engine  TEXT NOT NULL                   -- ollama | openai-compatible | cloud
);

CREATE TABLE chunks (
  id          TEXT PRIMARY KEY,
  meeting_id  TEXT NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
  channel     TEXT NOT NULL CHECK (channel IN ('mic', 'system')),
  seq         INTEGER NOT NULL,
  start_ms    INTEGER NOT NULL,
  end_ms      INTEGER NOT NULL,
  path        TEXT,                           -- NULL cuando el audio ya se borró
  status      TEXT NOT NULL,                  -- recorded | transcribing | transcribed | failed
  attempts    INTEGER NOT NULL DEFAULT 0,
  UNIQUE (meeting_id, channel, seq)
);

CREATE TABLE segments (
  id          TEXT PRIMARY KEY,
  meeting_id  TEXT NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
  chunk_id    TEXT NOT NULL REFERENCES chunks(id) ON DELETE CASCADE,
  channel     TEXT NOT NULL,
  start_ms    INTEGER NOT NULL,
  end_ms      INTEGER NOT NULL,
  text        TEXT NOT NULL,
  speaker     TEXT                            -- Fase 3 (TRN-06)
);

CREATE TABLE markers (
  id          TEXT PRIMARY KEY,
  meeting_id  TEXT NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
  at_ms       INTEGER NOT NULL,
  kind        TEXT NOT NULL DEFAULT 'important',
  note        TEXT
);

CREATE TABLE minutes (
  id            TEXT PRIMARY KEY,
  meeting_id    TEXT NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
  version       INTEGER NOT NULL,
  content_json  TEXT NOT NULL,                -- validado con el esquema Zod
  content_md    TEXT NOT NULL,                -- render canónico
  reviewed_at   TEXT,
  UNIQUE (meeting_id, version)
);

CREATE TABLE action_items (
  id          TEXT PRIMARY KEY,
  minute_id   TEXT NOT NULL REFERENCES minutes(id) ON DELETE CASCADE,
  text        TEXT NOT NULL,
  owner       TEXT,
  due_date    TEXT,
  refs        TEXT NOT NULL,                  -- JSON con ids de segmentos
  destination TEXT,                           -- ms-todo | notion | ms-lists | google-tasks (elegido al revisar)
  status      TEXT NOT NULL DEFAULT 'open'
);

CREATE TABLE exports (
  id           TEXT PRIMARY KEY,
  meeting_id   TEXT NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
  target       TEXT NOT NULL,                 -- markdown | notion | ms-todo | ms-lists | google-tasks | google-calendar | google-docs
  item_key     TEXT NOT NULL,                 -- 'minute' o el id de la tarea
  external_id  TEXT,                          -- id en el destino: clave de idempotencia
  exported_at  TEXT NOT NULL,
  UNIQUE (meeting_id, target, item_key)
);

CREATE TABLE connections (
  id            TEXT PRIMARY KEY,
  provider      TEXT NOT NULL,                -- notion | microsoft | google
  account       TEXT NOT NULL,                -- correo o nombre del workspace
  scopes        TEXT NOT NULL,                -- JSON; los tokens viven en el keychain
  connected_at  TEXT NOT NULL,
  UNIQUE (provider, account)
);

CREATE TABLE glossary (
  term  TEXT PRIMARY KEY,
  kind  TEXT NOT NULL DEFAULT 'term'          -- person | project | term
);

CREATE TABLE benchmarks (
  id       TEXT PRIMARY KEY,
  run_at   TEXT NOT NULL,
  model    TEXT NOT NULL,
  backend  TEXT NOT NULL,                     -- cuda | vulkan | metal | cpu
  rtf      REAL NOT NULL,
  wer      REAL NOT NULL,
  mode     TEXT NOT NULL                      -- live | post | blocked
);
```

## Ciclo de vida de una reunión

```mermaid
stateDiagram-v2
    [*] --> recording
    recording --> processing: el usuario detiene la grabación
    processing --> draft: transcripción y minuta listas
    processing --> failed: error sin reintento posible
    failed --> processing: reintentar
    draft --> reviewed: el usuario aprueba
    reviewed --> exported: exportación correcta
    exported --> reviewed: se edita de nuevo
```

Cada bloque pasa por `recorded → transcribing → transcribed`, o por `failed` con reintentos y backoff. Al iniciar la app, una reunión en `recording` sin grabadora activa pasa a `processing` con los bloques que ya están en disco, y los bloques que quedaron en `transcribing` vuelven a `recorded`.

## Filtro de hardware y motor por etapa (Fase 2)

```mermaid
flowchart TD
    A["Mínimos duros<br/>Windows 10/11 64 bits o macOS 14.2+<br/>8 GB de RAM y espacio para modelos"] --> B{"¿Cumple?"}
    B -->|no| X["Bloqueado: key propia o Taqnote Cloud"]
    B -->|sí| C["Benchmark de 1 minuto<br/>audio de muestra en español"]
    C -->|"RTF ≤ 0.3 y WER ≤ 15%"| L["En vivo"]
    C -->|"RTF ≤ 1.0 y WER ≤ 15%"| P["Al terminar la reunión"]
    C -->|peor| X
```

El módulo `hw` en Rust reporta sistema, RAM y disco libre, y corre el benchmark con un audio de 60 s en español de licencia libre (grabación propia o Common Voice) y su transcripción de referencia. Prueba los modelos de mayor a menor calidad en cada backend disponible, guarda el resultado en `benchmarks` y habilita los modos permitidos. El LLM local se mide aparte, en tokens por segundo.

El instalador de Windows usará Vulkan con respaldo en CPU, que cubre GPUs de NVIDIA, AMD e Intel sin distribuir el runtime de CUDA. CUDA queda como build opcional: en las RTX 50 (Blackwell) exige CUDA Toolkit 12.8 o superior y driver R570 o superior.

## Seguridad y privacidad

- Ninguna key vive en el repo. Desde la Fase 1, keys y tokens OAuth se guardan en el keychain del sistema, y la base solo registra las conexiones.
- Cada conector pide el permiso mínimo: `Tasks.ReadWrite` para To Do y `drive.file` en Google. El permiso de Lists solo se pide al activar ese conector.
- Las capabilities de Tauri habilitan solo los permisos de plugins que se usan, con una CSP estricta y sin contenido remoto en el WebView.
- El audio de cada bloque se borra tras transcribirse, salvo que el usuario elija conservarlo.
- A la nube solo sale lo que exige el motor elegido: el bloque de audio o el texto. Desde la Fase 3, los datos personales se enmascaran antes (PRV-04).
- Los logs nunca incluyen texto de reuniones ni keys.

## Taqnote Cloud (Fases 2 y 5)

Un Cloudflare Worker sin estado. Nace en la Fase 2 como broker de OAuth de Notion y en la Fase 5 suma licencias, cuotas y el proxy hacia Groq y un LLM económico, con D1. Ni el audio ni el texto se guardan: pasan y se descartan.

| Endpoint | Función | Fase |
| --- | --- | --- |
| `GET /oauth/notion/callback` | Recibe el código de Notion, lo cambia por tokens y emite un código de un solo uso | 2 |
| `POST /oauth/notion/token` | Entrega los tokens contra el código de un solo uso, o los renueva | 2 |
| `POST /v1/licenses/activate` | Registra el dispositivo contra la licencia | 5 |
| `POST /v1/transcribe` | Reenvía un bloque a Groq y devuelve segmentos | 5 |
| `POST /v1/summarize` | Reenvía la transcripción al LLM y devuelve la minuta | 5 |
| `GET /v1/usage` | Devuelve la cuota usada y la disponible | 5 |
| `POST /webhooks/paddle` | Verifica la firma y crea o actualiza licencias | 5 |

D1 guarda `licenses` (hash de la clave, estado, plan, id de suscripción de Paddle, fin del periodo), `activations` (licencia, dispositivo, último uso) y `usage` (licencia, periodo, segundos de audio, tokens). La clave de licencia nunca se guarda en claro. Los 10 ms de CPU por invocación del plan gratuito alcanzan porque el proxy pasa casi todo el tiempo esperando la respuesta del proveedor.

## Web (Fase 4)

La PWA reutiliza `packages/core` y `packages/ui` con adaptadores propios: captura con getUserMedia y getDisplayMedia, VAD en un AudioWorklet, IndexedDB como `MeetingStore` y transcripción solo en la nube. Si la API de Notion no acepta llamadas desde el navegador por CORS, la exportación pasa por un proxy en el Worker; hay que verificarlo al llegar a esa fase.

## Decisiones

Registradas:

- [01: Tauri 2 + React para escritorio](adr/01-tauri-react-desktop.md)
- [02: Captura del audio del sistema, sin bots](adr/02-capture-audio-system-no-bots.md)
- [03: Local-first, sin servidor en el MVP](adr/03-local-first-no-server.md)
- [04: Licencias en vez de cuentas](adr/04-license-instead-accounts.md)
- [05: Workers + D1 en vez de RDS](adr/05-workers-d1-instead-rds.md)
- [06: Modo local habilitado por benchmark](adr/06-local-mode-benchmark.md)
- [07: Conectores con modelo de tarea común y OAuth con PKCE en el cliente](adr/07-connectors-oauth-pkce.md)

Pendientes, cada una con su ADR al llegar:

- Broker de OAuth de Notion en el Worker, antes de EXP-03.

- Biblioteca de VAD, tras el spike CAP-00.
- Backend de GPU por defecto del instalador (Vulkan o CUDA), tras el spike CAP-00.
- Captura en macOS: Core Audio taps (macOS 14.2+) o cpal, que documenta loopback desde macOS 14.6. Antes de DIS-07.
- Licencia del proyecto (DOC-02).

## Referencias

- [Notion: guía de la versión 2025-09-03](https://developers.notion.com/docs/upgrade-guide-2025-09-03)
- [Notion: límites de solicitud](https://developers.notion.com/reference/request-limits) · [límites de tráfico](https://developers.notion.com/reference/rate-limits) · [autorización OAuth](https://developers.notion.com/docs/authorization)
- [Microsoft Graph: crear tarea de To Do](https://learn.microsoft.com/en-us/graph/api/todotasklist-post-tasks) · [crear elemento de lista](https://learn.microsoft.com/en-us/graph/api/listitem-create)
- [Microsoft Entra: redirect URIs](https://learn.microsoft.com/entra/identity-platform/reply-url) · [consentimiento de usuarios](https://learn.microsoft.com/en-gb/entra/identity/enterprise-apps/configure-user-consent)
- [Google: OAuth para apps de escritorio](https://developers.google.com/accounts/docs/OAuth2InstalledApp) · [estado de prueba y usuarios](https://support.google.com/cloud/answer/15549945?hl=es) · [verificación de scopes sensibles](https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification)
- [Google Tasks: campo de vencimiento](https://googleapis.dev/nodejs/googleapis/latest/tasks/interfaces/Schema$Task.html) · [Google Docs: importar Markdown](https://support.google.com/docs/answer/12014036?hl=en)
- [Crate wasapi (loopback)](https://docs.rs/wasapi) · [cpal](https://github.com/RustAudio/cpal)
- [whisper-rs (se mantiene en Codeberg)](https://codeberg.org/tazz4843/whisper-rs)
- [Modelos de whisper.cpp en Hugging Face](https://huggingface.co/ggerganov/whisper.cpp)
- [Groq: Whisper Large v3 Turbo](https://console.groq.com/docs/model/whisper-large-v3-turbo)
- [NVIDIA: guía de migración a Blackwell](https://forums.developer.nvidia.com/t/software-migration-guide-for-nvidia-blackwell-rtx-gpus-a-guide-to-cuda-12-8-pytorch-tensorrt-and-llama-cpp/321330)
- [Tauri 2: prerrequisitos](https://v2.tauri.app/start/prerequisites/)
