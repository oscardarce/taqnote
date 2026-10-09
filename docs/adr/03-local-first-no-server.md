# 03. Local-first, sin servidor en el MVP

Estado: aceptada · Fecha: 2026-10-08

## Contexto

Taqnote debe ser gratis por defecto y respetar la confidencialidad de las reuniones. Un servidor que transcribe para otros cuesta más con cada usuario (GPU o APIs pagadas), mientras que las donaciones no crecen al mismo ritmo.

## Decisión

Los datos viven en el equipo del usuario: SQLite en escritorio e IndexedDB en la web. El MVP no tiene backend: el modo local y el modo con key propia funcionan sin servidor. Un servidor aparece solo para el broker de OAuth de Notion (Fase 2) y para Taqnote Cloud (Fase 5), y nunca guarda audio ni transcripciones.

## Alternativas descartadas

- **App web con transcripción en el servidor:** costo variable por usuario, y el audio de reuniones confidenciales sale del equipo.
- **Base de datos en la nube desde el inicio:** costo fijo y datos sensibles fuera del equipo, sin un beneficio que el MVP necesite.

## Consecuencias

- A favor: costo fijo cero y privacidad por defecto.
- En contra: no hay sincronización entre dispositivos (queda en espera, con cifrado de extremo a extremo), el respaldo depende del usuario y el rendimiento depende del hardware de cada uno (ver ADR 06).
- Se revisa si: los usuarios necesitan sincronizar entre dispositivos.

## Fuentes

- `docs/ROADMAP.md`, principios 1, 2, 3 y 7.
