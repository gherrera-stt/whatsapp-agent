# whatsapp-agent

Agente inteligente integrado con WhatsApp Business API para automatizar la atención al cliente, realizar cotizaciones en tiempo real y gestionar pedidos de productos mediante NLP con Claude.

## Stack

- **Interfaz:** WhatsApp Cloud API (Meta)
- **Backend:** Node.js + Express (TypeScript)
- **Cerebro AI:** Claude (`claude-sonnet-4-6`)
- **Base de datos:** PostgreSQL
- **Admin:** React + Vite

## Documentación

- [Definición de la solución](docs/solucion.md)
- [Slices de implementación](docs/slices/)
- [Adversarial System Review](docs/adversarial-review.md) — proceso de revisión crítica antes y después de cada slice
- [Bitácora de revisiones](docs/reviews/)

## Estado

En desarrollo — fase de definición.

**Próximos pasos:**
1. Ejecutar `pre-flight.md` (revisión adversarial inicial). Sin Go, no se inicia construcción.
2. Slice 0 — Andamiaje (con sus reportes `pre-slice-0.md` y `post-slice-0.md`).
3. Continuar slice por slice con el gate ASR entre cada uno.
