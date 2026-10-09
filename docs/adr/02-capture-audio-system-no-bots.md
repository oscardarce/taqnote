# 02. Captura del audio del sistema, sin bots

Estado: aceptada · Fecha: 2026-10-08

## Contexto

Las reuniones ocurren en Teams, Google Meet y Zoom. Para transcribirlas hace falta el audio de todos los participantes, no solo el del micrófono: si el usuario usa audífonos, el micrófono no capta a los demás.

## Decisión

Capturar en el equipo dos canales separados: el micrófono (el usuario) y el audio del sistema (los demás). En Windows, el audio del sistema se captura con loopback de WASAPI mediante el crate `wasapi`, y el micrófono con `cpal`. Nadie extra entra a la reunión.

## Alternativas descartadas

- **Bots que se unen a la llamada:** cuestan un servidor por reunión, dependen de los permisos de cada plataforma y muchas empresas los bloquean.
- **Transcripciones de Teams vía Microsoft Graph:** solo cubren Teams con la transcripción activada, no son en vivo, exigen el permiso `OnlineMeetingTranscript.Read.All` y la reunión no debe haber expirado. Queda como idea para después de 1.0.

## Consecuencias

- A favor: funciona con cualquier plataforma, y los canales separados dan "tú vs. el resto" sin diarización.
- En contra: hay código nativo por sistema operativo (macOS usará Core Audio taps, disponibles desde macOS 14.2). Las reuniones presenciales con un solo micrófono son más difíciles, y grabar a otros exige un aviso (PRV-03).
- Se revisa si: alguna plataforma ofrece audio en vivo por API sin necesidad de bots.

## Fuentes

- [Crate wasapi (loopback)](https://docs.rs/wasapi) · [cpal](https://github.com/RustAudio/cpal)
- [Microsoft Graph: transcripciones de reuniones](https://learn.microsoft.com/en-us/graph/api/onlinemeeting-list-transcripts)
- [Core Audio taps desde macOS 14.2](https://blog.addpipe.com/getdisplaymedia-allows-capturing-the-screen-with-system-sounds-on-chrome-on-macos/)
