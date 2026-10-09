# 01. Tauri 2 + React para la app de escritorio

Estado: aceptada · Fecha: 2026-10-08

## Contexto

Taqnote corre durante las reuniones, al lado de Teams, Meet o Zoom, que ya consumen bastantes recursos. Necesita capturar el audio del sistema y correr modelos locales (whisper.cpp), y ambas cosas exigen código nativo. Oscar ya trabaja con React, Vite y Tailwind.

## Decisión

App de escritorio con Tauri 2: interfaz en React + Vite + Tailwind y backend en Rust. Rust se limita a lo que solo puede ser nativo (captura, VAD, inferencia local y filtro de hardware); la lógica de negocio vive en TypeScript, en `packages/core`.

## Alternativas descartadas

- **Electron:** todo en JavaScript, pero empaqueta su propio Chromium, así que pesa más en memoria y en el instalador. Además, la captura del sistema y whisper.cpp exigen código nativo de todos modos, y la ventaja de "solo JavaScript" se diluye.
- **Solo web:** el navegador limita la captura. El audio del sistema solo llega al compartir la pantalla completa en Chrome o Edge sobre Windows o ChromeOS (en macOS, desde Chrome 141 con macOS 14.2+), y el navegador no corre bien modelos grandes. Queda como cliente secundario en la Fase 4.

## Consecuencias

- A favor: app liviana y reutilización de React. En Windows, la WebView es WebView2, basada en Chromium, así que la interfaz se comporta como en Chrome.
- En contra: hay que aprender Rust básico. En macOS la WebView es WebKit y requiere más pruebas.
- Referencia: Meetily (licencia MIT) usa el mismo enfoque (Tauri, Rust y whisper.cpp) y sirve como código de consulta, con atribución.

## Fuentes

- [Tauri 2: prerrequisitos](https://v2.tauri.app/start/prerequisites/)
- [Arquitectura de Meetily](https://raw.githubusercontent.com/Zackriya-Solutions/meetily/main/docs/architecture.md)
- [getDisplayMedia y el audio del sistema](https://addpipe.com/getdisplaymedia-demo/) · [Chrome en macOS](https://blog.addpipe.com/getdisplaymedia-allows-capturing-the-screen-with-system-sounds-on-chrome-on-macos/)
