# Taqnote: plan de implementación

Última actualización: 2026-10-08 · Relacionado: [roadmap](../roadmap.md) · [arquitectura](../arquitectura.md)

Este plan detalla paso a paso las fases 0 y 1, que son las que se construyen ahora. Las fases 2 a 5 quedan a nivel de hitos y se detallan al empezar cada una: un plan fino a meses de distancia envejece antes de usarse. Cada paso es un pull request y referencia los IDs del roadmap.

## Cómo se trabaja

- **Un paso, una rama, un PR.** Rama `feat/CAP-01-loopback`; commits en formato Conventional Commits con el ID: `feat(captura): CAP-01 loopback de WASAPI`.
- **Definición de listo:** CI en verde, pruebas del paso, docs actualizadas e Issue cerrado desde el PR con `Closes #<número>`.
- **Decisiones:** si un paso obliga a elegir entre alternativas, se escribe un ADR antes de mergear.
- **Tamaño:** S, M o L es esfuerzo relativo, no una fecha. Las fechas viven en los milestones de GitHub.

## Prerrequisitos (Windows)

| Herramienta | Para qué | Nota |
| --- | --- | --- |
| Git y GitHub CLI (`gh`) | Repo, issues y PRs | |
| Node.js LTS y pnpm | Monorepo y frontend | pnpm se activa con `corepack enable` |
| Rust con rustup | Backend de Tauri | Toolchain MSVC: `rustup default stable-msvc` |
| Microsoft C++ Build Tools | Compilar Rust y whisper.cpp | Workload "Desktop development with C++" |
| CMake | Compilar whisper.cpp | |
| Vulkan SDK | Backend de GPU por defecto | |
| CUDA Toolkit 12.8 o superior | Backend CUDA en la RTX 5070 (opcional) | Blackwell exige 12.8+ y driver R570+ |
| LLVM (libclang) | Bindings de whisper-rs | Solo si el build lo pide |
| Ollama | LLM local | Un modelo de 8 a 14B parámetros |
| VS Code con rust-analyzer | Editor | |

## Ruta de aprendizaje mínima

Antes del spike (paso 0.5):

- **Rust:** capítulos 1 a 10 y 16 de *The Rust Programming Language*: ownership, manejo de errores y concurrencia con hilos y canales.
- **Tauri 2:** comandos, eventos y capabilities.
- **Audio digital:** frecuencia de muestreo, canales, PCM y remuestreo.

## Fase 0: Fundamentos

### 0.1 Documentación base · DOC-01 · S

Crear `docs/` con su `README.md` (índice), `roadmap.md`, `arquitectura.md`, este plan y los ADRs 0001 a 0006 que enlaza la arquitectura.

**Listo cuando:** el README del repo enlaza a `docs/` y cada ADR tiene contexto, decisión y consecuencias.

### 0.2 Licencia · DOC-02 · S

Elegir entre MIT o Apache-2.0, que maximizan la adopción, y AGPL-3.0, que obliga a publicar los cambios a quien ofrezca Taqnote como servicio. Registrar la decisión en un ADR y agregar `LICENSE`.

**Listo cuando:** `LICENSE` está en la raíz y el README indica la licencia.

### 0.3 Monorepo · DIS-01 · M

1. En la raíz: `corepack enable` y `pnpm init`. En `package.json`, fijar `packageManager` con la versión de pnpm instalada, para que el CI use la misma.
2. Crear `pnpm-workspace.yaml`:

    ```yaml
    packages:
      - "apps/*"
      - "packages/*"
    ```

3. Dentro de `apps/`, ejecutar `pnpm create tauri-app` con nombre `desktop`, frontend React con TypeScript y pnpm como gestor.
4. Crear `packages/core` (`@taqnote/core`): TypeScript estricto y Vitest, con los tipos de dominio y los puertos de la arquitectura, sin dependencias de Tauri ni del navegador.
5. Crear `packages/ui` (`@taqnote/ui`): componentes React con Tailwind. `apps/desktop` consume ambos con `"workspace:*"`.
6. Calidad: ESLint y Prettier para TypeScript, `cargo fmt` y `cargo clippy` para Rust, `.editorconfig`, y un `.gitignore` con `node_modules`, `target`, `dist`, `.env*` y las carpetas de grabaciones de desarrollo. Los audios de `fixtures/` sí se versionan.

**Listo cuando:** `pnpm --filter desktop tauri dev` abre la ventana con un componente de `@taqnote/ui` que usa un tipo de `@taqnote/core`.

### 0.4 CI · DIS-02 · S

Crear `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v7
      - uses: pnpm/action-setup@v6          # toma la versión de packageManager
      - uses: actions/setup-node@v7
        with:
          node-version: lts/*
          cache: pnpm
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - uses: Swatinem/rust-cache@v2
        with:
          workspaces: apps/desktop/src-tauri
      - run: pnpm install --frozen-lockfile
      - run: pnpm -r --if-present run lint
      - run: pnpm -r --if-present run test
      - run: cargo fmt --check --manifest-path apps/desktop/src-tauri/Cargo.toml
      - run: cargo clippy --manifest-path apps/desktop/src-tauri/Cargo.toml -- -D warnings
      - run: cargo test --manifest-path apps/desktop/src-tauri/Cargo.toml
```

El build del instalador entra en la Fase 2 (DIS-03); por ahora el CI solo verifica.

**Listo cuando:** un PR muestra el check en verde y la rama `main` exige que pase antes de mergear. En repos públicos, la protección de rama es gratuita.

### 0.5 Spike de captura y transcripción · CAP-00 · L

En `spikes/cap-00/`, un binario de Rust independiente y desechable:

1. Listar dispositivos y grabar 30 s del micrófono a WAV con cpal y hound.
2. Grabar 30 s del audio del sistema con loopback de WASAPI (crate `wasapi`) mientras suena una reunión de prueba.
3. Grabar ambos a la vez en archivos separados, con tiempos de un reloj monotónico, y rellenar con silencio los huecos del loopback.
4. Convertir a 16 kHz mono con rubato y comparar la duración del archivo contra el reloj.
5. Transcribir con whisper-rs y large-v3-turbo usando Vulkan (y CUDA, si se instaló), imprimir los segmentos con tiempos y medir el RTF.
6. Probar 10 minutos de una reunión real de Teams, Meet o Zoom.

Los hallazgos van a `docs/spikes/cap-00.md`, y las decisiones de VAD y backend de GPU, a sus ADRs.

**Listo cuando:** 10 minutos de reunión real producen dos WAV cuya duración difiere del reloj menos de 1% y un transcript legible, con el RTF medido.

## Fase 1: MVP personal (Windows)

Los pasos 1.1 a 1.4 producen una grabadora con transcript; los pasos 1.5 a 1.8 la convierten en minutas, y el 1.9 es el uso real.

### 1.1 Modelo de datos y almacenamiento · PRV-01 · M

- Migraciones de SQLite con tauri-plugin-sql según el esquema de la arquitectura, con `PRAGMA foreign_keys = ON`.
- Puerto `MeetingStore` en `@taqnote/core` y adaptador SQLite en `apps/desktop`.
- Un `MeetingStore` en memoria para las pruebas del orquestador.

**Listo cuando:** se crea, lista y borra una reunión con sus bloques y segmentos desde una pantalla de desarrollo.

### 1.2 Grabadora · CAP-01, CAP-02, CAP-03 · L

- Pasar el código del spike a `src-tauri/src/audio/`: un hilo de captura por canal y un canal de mensajes hacia el chunker.
- VAD con cortes en silencio: objetivo de 30 s, mínimo de 10 s y máximo de 40 s. Cada bloque se escribe en un archivo temporal y se renombra al cerrarse.
- Comandos de Tauri `start_recording` y `stop_recording`, y eventos `chunk_ready` y `audio_level`.
- Recuperación al iniciar: una reunión en `recording` sin grabadora activa pasa a `processing`, y sus bloques se reconstruyen desde los nombres de archivo.
- Pruebas en Rust del chunker con audios de `fixtures/`.

**Listo cuando:** al matar el proceso a mitad de una grabación de 20 minutos y reabrir la app, la reunión conserva todos los bloques cerrados.

### 1.3 Transcripción local · TRN-01, TRN-03 · L

- Módulo `src-tauri/src/asr/` con whisper-rs. Las features `vulkan` y `cuda` son features opcionales de Cargo, y el CI compila solo para CPU.
- Gestor de modelos: descarga desde Hugging Face a la carpeta de datos de la app, con verificación del SHA.
- Cola de un trabajo a la vez para la GPU; idioma `es` por defecto y glosario como prompt inicial.
- Comando `transcribe_chunk` que devuelve segmentos, y adaptador `Transcriber` local en TypeScript que lo invoca.
- Si la retención de audio está desactivada, el WAV se borra al transcribirse (PRV-01).

**Listo cuando:** una reunión de 30 minutos queda transcrita en local y el glosario corrige al menos un nombre propio que antes salía mal.

### 1.4 Transcript en vivo y marcadores · TRN-02, UX-01 · M

- Vista de la reunión que muestra los segmentos a medida que llegan, con las etiquetas "Yo" (micrófono) y "Otros" (sistema).
- Atajo global con tauri-plugin-global-shortcut que guarda un marcador con el tiempo actual.

**Listo cuando:** en una reunión real el texto aparece con unos 30 a 40 s de retraso y los marcadores quedan en la línea de tiempo.

### 1.5 Minuta con LLM local · MIN-01, MIN-03, TAR-01 · L

- Adaptador `Summarizer` para Ollama con salida JSON y el tamaño de contexto (`num_ctx`) fijado de forma explícita.
- Prompt en español y esquema Zod de la minuta: temas, acuerdos con responsable y fecha, y pendientes, cada punto con `refs` a segmentos.
- Resumen por ventanas de ~10 minutos y fusión final, para que una reunión de una hora quepa en el contexto del modelo local.
- Validación: se marcan los puntos cuyas `refs` no existen, y se reintenta si el JSON no cumple el esquema.
- Pruebas con Vitest del armado de ventanas, del parseo y de la validación de referencias.

**Listo cuando:** una reunión de una hora produce una minuta válida en local y cada acuerdo enlaza a su segmento.

### 1.6 Revisión · MIN-02, TAR-01 · M

- Editor de la minuta y tabla editable de acuerdos, con responsable y fecha.
- Cada punto muestra su fragmento del transcript al pasar el cursor.
- El botón "Aprobar" pasa la reunión a `reviewed`.

**Listo cuando:** se corrige una minuta completa sin tocar la base de datos a mano.

### 1.7 Exportar a Markdown · EXP-01 · S

- Render canónico de la minuta a Markdown, el mismo que alimenta a Notion, guardado en una carpeta elegida por el usuario.

**Listo cuando:** el archivo se ve en VS Code igual que en la app.

### 1.8 Exportar a Notion · EXP-02 · M

- Integración interna de Notion con acceso a dos data sources relacionados: Minutas y Tareas.
- Cliente con `Notion-Version: 2025-09-03` que crea las páginas con un parent de tipo `data_source_id`.
- Troceo en textos de hasta 2000 caracteres y lotes de hasta 100 bloques; ante un 429, reintento respetando `Retry-After`.
- Idempotencia: la tabla `exports` guarda el id de la página, y reexportar la actualiza en vez de duplicarla.
- El token vive en la carpeta de configuración de la app, nunca en el repo.

**Listo cuando:** exportar dos veces la misma reunión deja una sola página con sus tareas enlazadas.

### 1.9 Uso real · salida de la Fase 1 · M

- Usar Taqnote en todas las reuniones durante dos semanas y abrir un Issue por cada fallo.
- Anotar por reunión el tiempo hasta la minuta, las ediciones necesarias y el audio perdido.

**Listo cuando:** se cumple la salida del roadmap: 10 reuniones reales, minutas en Notion y cero audio perdido.

## Fases 2 a 5: hitos

### Fase 2: Beta pública

1. Mínimos y benchmark (HW-01, HW-02) con un audio de muestra propio o de Common Voice.
2. Motor por etapa y keys propias en el keychain (HW-03, TRN-05, MIN-04, PRV-02).
3. Parakeet v3 para CPU (TRN-04), medido con los mismos fixtures.
4. Onboarding y aviso de grabación (UX-02, PRV-03).
5. Conexión con Notion vía OAuth (EXP-03); antes, verificar si el intercambio de tokens necesita un endpoint en Workers.
6. Instalador firmado con auto-actualización (DIS-03), prueba de WER en el CI (DIS-04), landing (DIS-05) y GitHub Sponsors (DIS-06).

### Fase 3: Diferenciadores

Primero TAR-02 y TAR-03, que usan datos que ya existen. Después MIN-05 a MIN-07, TRN-06 y TRN-07, UX-03 a UX-05, EXP-04 a EXP-07, PRV-04 e HIS-01 e HIS-02. La versión de macOS (DIS-07) entra cuando haya presupuesto para Apple Developer y una Mac de pruebas.

### Fase 4: Web

Reutiliza `@taqnote/core` y `@taqnote/ui` con adaptadores del navegador (WEB-01 a WEB-04). La extensión de Chrome (WEB-05) va al final.

### Fase 5: Taqnote Cloud

Workers y D1 (CLD-01), después Paddle y licencias (CLD-02, CLD-03), luego el proxy y las cuotas (CLD-04, CLD-05) y, al final, los documentos legales y los precios (CLD-06). Puede avanzar en paralelo con la Fase 4.

## Riesgos

| Riesgo | Mitigación |
| --- | --- |
| El loopback de WASAPI deja huecos o se desfasa | El spike CAP-00 lo valida antes de construir encima |
| whisper.cpp no compila en la máquina de desarrollo | Prerrequisitos listados; Vulkan como alternativa a CUDA |
| El LLM local inventa acuerdos | Referencias obligatorias a segmentos y revisión antes de exportar |
| La API de Notion cambia | Versión fijada en el header y pruebas del cliente con respuestas grabadas |
| Agotamiento en un proyecto personal largo | Fases cortas con salida verificable y uso real desde la Fase 1 |

## Referencias

- [Tauri 2: prerrequisitos](https://v2.tauri.app/start/prerequisites/)
- [NVIDIA: guía de migración a Blackwell](https://forums.developer.nvidia.com/t/software-migration-guide-for-nvidia-blackwell-rtx-gpus-a-guide-to-cuda-12-8-pytorch-tensorrt-and-llama-cpp/321330)
- [actions/checkout](https://github.com/actions/checkout) · [actions/setup-node](https://github.com/actions/setup-node) · [pnpm/action-setup](https://github.com/pnpm/action-setup) · [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache)
- [Notion: guía de la versión 2025-09-03](https://developers.notion.com/docs/upgrade-guide-2025-09-03)
