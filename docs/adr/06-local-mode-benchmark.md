# 06. Modo local habilitado por benchmark

Estado: aceptada · Fecha: 2026-10-08

## Contexto

Transcribir y resumir en local depende del hardware de cada usuario. Una tabla de requisitos engaña: la misma GPU rinde distinto en una laptop que en un escritorio, un driver roto produce texto basura y una laptop en batería se frena.

## Decisión

Dos filtros antes de habilitar el modo local:

1. **Mínimos duros**, que bloquean sin probar nada: Windows 10/11 de 64 bits o macOS 14.2+, 8 GB de RAM y espacio en disco para los modelos.
2. **Benchmark de un minuto** con un audio en español de transcripción conocida, que mide velocidad (RTF, tiempo de proceso entre duración del audio) y precisión (WER, porcentaje de palabras erradas):
    - RTF ≤ 0.3 y WER ≤ 15%: transcripción en vivo.
    - RTF ≤ 1.0 y WER ≤ 15%: transcripción al terminar la reunión.
    - Peor: modo local bloqueado; se ofrece key propia o Taqnote Cloud.

Transcripción y minuta eligen su motor por separado: local, key propia o Taqnote Cloud.

## Alternativas descartadas

- **Solo una tabla de requisitos:** no detecta drivers rotos ni equipos lentos.
- **Sin bloqueo:** el usuario recibe minutas malas y culpa a la app.

## Consecuencias

- A favor: experiencia honesta, y cada equipo usa el mejor modelo que soporta.
- En contra: hay que mantener un audio de prueba con su transcripción de referencia (grabación propia o Common Voice). Los umbrales son iniciales y se ajustan con datos reales, y el benchmark se repite si cambia el hardware o el driver.

## Fuentes

- [whisper.cpp: tamaño y memoria por modelo](https://huggingface.co/ggerganov/whisper.cpp)
