# 05. Cloudflare Workers + D1 en vez de RDS

Estado: aceptada · Fecha: 2026-10-08

## Contexto

Taqnote necesita un backend pequeño: el broker de OAuth de Notion (Fase 2) y, en la Fase 5, licencias, cuotas y un proxy sin estado hacia los proveedores de IA. La meta es costo fijo cero y nada que administrar.

## Decisión

Cloudflare Workers para los endpoints y D1 (SQLite administrado) para licencias, activaciones y uso.

## Alternativas descartadas

- **AWS RDS:** no tiene capa siempre gratuita. En cuentas creadas desde el 15 de julio de 2025, el plan gratuito funciona con créditos y la cuenta se cierra a los 6 meses o al agotarlos. Además exige administrar VPC, security groups y respaldos.
- **Supabase Free:** el proyecto se pausa tras 7 días sin actividad y se restaura a mano; quitar la pausa cuesta $25 al mes (plan Pro). Es inaceptable con clientes pagando.

## Consecuencias

- A favor: el plan gratuito de Workers da 100.000 peticiones al día, y D1 incluye 5 GB, 5 millones de filas leídas y 100.000 escritas por día. D1 es SQLite, el mismo motor del escritorio.
- En contra: el plan gratuito limita a 10 ms de CPU por invocación. Alcanzan para un proxy, porque la espera al proveedor no cuenta como CPU, pero obligan a mantener el Worker liviano y sin estado.
- Se revisa si: el uso supera los límites diarios del plan gratuito.

## Fuentes

- [Cloudflare Workers: precios](https://developers.cloudflare.com/workers/platform/pricing/) · [D1: preguntas frecuentes](https://developers.cloudflare.com/d1/reference/faq)
- [AWS Free Tier en 2026](https://spot.rackspace.com/blog/aws-free-tier) · [RDS sin capa siempre gratuita](https://infratally.com/articles/aws-free-tier-2026/)
- [Supabase: precios](https://supabase.com/pricing)
