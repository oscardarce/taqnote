# 07. Conectores con modelo de tarea común y OAuth con PKCE en el cliente

Estado: aceptada · Fecha: 2026-10-08

## Contexto

Las minutas y tareas deben llegar a Notion, Microsoft To Do y Lists, y Google Calendar, Tasks y Docs. Cada API impone sus propias reglas:

- To Do y Google Tasks son listas personales: no asignan tareas a otras personas.
- Lists exige cuenta de trabajo o escuela y el permiso `Sites.ReadWrite.All`, que suele requerir la aprobación de un admin.
- Google Tasks guarda la fecha de vencimiento sin hora, y los scopes de Calendar son sensibles, así que Google exige verificar la app.
- El endpoint de tokens de Notion exige el client secret.
- Microsoft Loop no tiene API pública de escritura.

## Decisión

- Un modelo común (`ActionItem`) y tres puertos: `DocDestination`, `TaskDestination` y `CalendarDestination`. Cada app es un adaptador.
- Enrutamiento por responsable: los compromisos del usuario van a To Do o Google Tasks, y las tareas de otros, a Notion o Lists. Un punto con fecha y hora propone además un evento en Calendar. El usuario confirma cada destino en la revisión.
- Exportaciones idempotentes: la tabla `exports` guarda el id externo, y reexportar actualiza en vez de duplicar.
- OAuth de escritorio: authorization code con PKCE en el navegador del sistema, redirección loopback a `127.0.0.1`, cliente público sin secret y tokens en el keychain del sistema.
- Notion: token interno en la Fase 1. En la Fase 2, OAuth a través de un broker en el Worker, que guarda el client secret.

## Alternativas descartadas

- **Integraciones sueltas por app:** duplican la lógica de tareas, reintentos e idempotencia.
- **Client secret dentro de la app:** una app instalada no puede guardar secretos.
- **Guardar los tokens en un backend:** contradice el ADR 03.
- **Microsoft Loop:** no tiene API de escritura.

## Consecuencias

- A favor: un conector nuevo es un adaptador nuevo, y cada conector pide el permiso mínimo (`Tasks.ReadWrite`, `drive.file`).
- En contra: la verificación de Google tarda de 3 a 5 días hábiles y exige página pública, política de privacidad en un dominio verificado y un video. En modo de prueba, las autorizaciones vencen a los 7 días y hay un máximo de 100 usuarios de prueba. Algunos tenants de Microsoft solo permiten consentir apps de publishers verificados.
- Se revisa si: Microsoft publica una API de escritura para Loop.

## Fuentes

- [Graph: crear tarea de To Do](https://learn.microsoft.com/en-us/graph/api/todotasklist-post-tasks) · [crear elemento de lista](https://learn.microsoft.com/en-us/graph/api/listitem-create)
- [Entra: consentimiento de usuarios](https://learn.microsoft.com/en-gb/entra/identity/enterprise-apps/configure-user-consent) · [redirect URIs](https://learn.microsoft.com/entra/identity-platform/reply-url)
- [Google: OAuth para apps de escritorio](https://developers.google.com/accounts/docs/OAuth2InstalledApp) · [estado de prueba](https://support.google.com/cloud/answer/15549945?hl=es) · [verificación de scopes sensibles](https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification)
- [Google Tasks: campo de vencimiento](https://googleapis.dev/nodejs/googleapis/latest/tasks/interfaces/Schema$Task.html)
- [Notion: autorización](https://developers.notion.com/docs/authorization)
- [Microsoft Loop sin API pública](https://learn.microsoft.com/en-us/answers/questions/5432288/is-there-an-api-for-microsoft-loop-integration-wit)
