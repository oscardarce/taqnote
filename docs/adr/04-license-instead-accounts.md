# 04. Licencias en vez de cuentas

Estado: aceptada · Fecha: 2026-10-08

## Contexto

El plan de pago (Taqnote Cloud, Fase 5) necesita saber quién pagó y cuánto usa. Escribir autenticación propia (contraseñas, sesiones, recuperación, protección contra fuerza bruta, verificación de correo, OAuth) es un riesgo de seguridad: una brecha en una app con el contenido de reuniones mata el producto.

## Decisión

Claves de licencia, sin cuentas ni contraseñas:

1. El usuario compra en Paddle.
2. Paddle avisa al backend con un webhook firmado.
3. El backend genera la licencia y guarda solo su hash en D1.
4. La app la activa una vez. Cada petición a Taqnote Cloud la incluye, y el backend valida estado, cuota y número de dispositivos.

## Alternativas descartadas

- **Autenticación propia:** riesgo alto sin beneficio para el MVP.
- **Cuentas con Supabase Auth o Better Auth:** válidas, pero innecesarias mientras no haya sincronización ni equipos. Si llegan, se usará una de ellas, nunca autenticación escrita desde cero.

## Consecuencias

- A favor: no hay pantallas de login ni contraseñas que proteger, así que la superficie de ataque es menor.
- En contra: hace falta un flujo de recuperación de licencia y un límite de dispositivos (CLD-03).
- Se revisa si: se agrega sincronización entre dispositivos o espacios para equipos.

## Fuentes

- [Meetily: activación por licencia](https://meetily.ai/pricing)
- [Better Auth](https://noqta.tn/en/blog/better-auth-typescript-authentication-library-2026) · [Supabase: precios y límites de Auth](https://supabase.com/pricing)
- [Paddle: precios](https://paddle.com/pricing)
