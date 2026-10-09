# Progreso

<!-- Se lee al inicio de cada sesión: máximo ~40 líneas. Se reemplaza, no se acumula (el historial vive en git). Lo actualiza /cierre-sesion. -->

Última actualización: 2026-10-09

## Dónde estamos

- Fase 0 (Fundamentos), paso 0.1 Documentación base (DOC-01): en curso.
- Hecho en esta sesión: revisamos el ADR 01 (Tauri 2 + React) con preguntas; cumple el criterio del paso (contexto, decisión y consecuencias). Al plan se agregó el riesgo de la WebView minimizada (tabla de Riesgos y "Listo cuando" del paso 1.3), y al glosario, la entrada de Tauri.
- Falta para cerrar 0.1: revisar los ADRs 02 a 07, crear `docs/README.md` (no existe) y escribir el `README.md` de la raíz (existe vacío, sin versionar).

## Siguiente acción

1. Abrir con la pregunta de repaso (abajo).
2. Revisar con Oscar `docs/adr/02-capture-audio-system-no-bots.md`: explicar la decisión, preguntas de comprobación y glosario. Luego del 03 al 07, uno por uno.
3. Oscar escribe `docs/README.md` y el `README.md` de la raíz, con guía del agente.

## Decisiones recientes

- Nombres de archivo en inglés y documentos en español.
- Frontera Rust/TypeScript (lectura del ADR 01): va en Rust lo que necesita el sistema operativo o el hardware, o lo que trabaja sobre el audio crudo, que nace en Rust; las decisiones de producto van en TypeScript. No requiere ADR nuevo.
- Riesgo de throttling: con la ventana minimizada, la WebView puede frenarse o suspenderse, y en Windows Tauri no permite desactivarlo (docs.rs, tauri-utils 2.10.1, campo `background_throttling`). Mitigación: el disco es la fuente de verdad y el orquestador procesa los bloques pendientes al arrancar y al volver a mostrarse. Si en el paso 1.3 el atraso es inaceptable, se escribe un ADR (por ejemplo, para mover la cola de transcripción a Rust).

## Bloqueos y preguntas abiertas

- Licencia por decidir en el paso 0.2 (DOC-02): MIT o Apache-2.0 frente a AGPL-3.0. `LICENSE.md` existe vacío y sin versionar.
- No verificado: que Meetily sea MIT (verificar antes de consultar su código) y el audio del sistema en Chrome 141 con macOS 14.2+ (verificar en la Fase 4).

## Pregunta de repaso para abrir la próxima sesión

La reunión dura 60 minutos y la vista de Taqnote sale de memoria en el minuto 10. Al restaurar la ventana, ¿qué tiene que hacer el orquestador para que la minuta no tenga huecos, y qué dato del nombre de los WAV se lo permite? Pista: `docs/architecture.md`, sección "Captura".

## Conceptos vistos

Ver `docs/learning/glossary.md`. Pendientes de escribir con palabras de Oscar: WebView, IPC, throttling y fuente de verdad.
