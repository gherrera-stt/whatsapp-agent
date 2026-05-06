# Definición de la Solución — Agente Inteligente para WhatsApp

> Documento de definición del producto.
> Última actualización: 2026-05-06

---

## 1. Overview

Desarrollo de un **agente inteligente integrado con WhatsApp Business API** para automatizar la atención al cliente, realizar **cotizaciones en tiempo real** y **gestionar pedidos de productos** mediante procesamiento de lenguaje natural (NLP) utilizando **Claude** como motor de comprensión y generación de respuestas.

**Objetivo de negocio:** reducir el tiempo de respuesta al cliente, escalar la operación comercial sin sumar agentes humanos a la misma proporción y mantener una experiencia conversacional natural sobre el canal preferido del cliente final (WhatsApp).

**Valor diferencial:**
- Conversación natural en español, no flujos rígidos de menús.
- Cotizaciones consistentes con el catálogo (sin alucinaciones de precio o stock).
- Persistencia de pedidos y trazabilidad de cada conversación.
- Handoff a operador humano cuando el agente no tiene confianza suficiente.

**Componentes principales:**

| Capa | Tecnología | Rol |
|---|---|---|
| Interfaz | WhatsApp Cloud API (Meta) | Punto de entrada/salida de mensajes |
| Backend | Node.js (Express) | Webhooks, lógica de negocio, orquestación |
| Cerebro AI | Claude (Anthropic API) | Interpretación de intenciones y generación de respuestas |
| Datos | PostgreSQL | Catálogo, conversaciones, pedidos |
| Admin | React + Vite | Panel para gestionar catálogo y supervisar conversaciones |

---

## 2. Solution Narrative

### 2.1 Estructura del workspace

Monorepo simple con separación clara entre backend, frontend de administración, infraestructura local y documentación.

```
whatsapp-agent/
├── apps/
│   ├── backend/              # API Express + webhooks WhatsApp + integración Claude
│   │   ├── src/
│   │   │   ├── routes/       # /webhook, /health, /admin/*
│   │   │   ├── services/     # whatsapp, claude, catalog, orders
│   │   │   ├── domain/       # entidades: Product, Order, Conversation
│   │   │   ├── db/           # cliente PostgreSQL, migraciones, seeds
│   │   │   ├── prompts/      # system prompts y plantillas
│   │   │   └── index.ts
│   │   ├── tests/
│   │   └── package.json
│   └── admin/                # Panel React (catálogo + monitoreo)
│       ├── src/
│       └── package.json
├── packages/
│   └── shared/               # Tipos y validadores compartidos (zod)
├── infra/
│   ├── docker-compose.yml    # PostgreSQL local, opcionalmente Redis
│   └── migrations/           # Scripts SQL versionados
├── docs/
│   └── solucion.md           # Este documento
├── .env.example
├── package.json              # Workspaces (npm/pnpm)
└── README.md
```

**Decisiones clave:**
- Monorepo con workspaces (npm o pnpm) para compartir tipos entre backend y admin.
- TypeScript en todo el código aplicativo.
- Migraciones SQL versionadas en `infra/migrations`, ejecutadas por **`dbmate`** (SQL puro, idempotente vía tabla de tracking propia, naming por timestamp). Decisión final tomada para evitar la indefinición que arrastraba `node-pg-migrate` cuando se usan archivos `.sql` puros.

### 2.2 Flujo de desarrollo local — Backend

1. **Levantar dependencias de infraestructura:**
   ```
   docker compose -f infra/docker-compose.yml up -d
   ```
   Esto inicia PostgreSQL local en `localhost:5432`.

2. **Instalar dependencias y configurar entorno:**
   ```
   cp .env.example .env
   npm install
   ```
   Variables mínimas: `DATABASE_URL`, `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_ID`, `WHATSAPP_VERIFY_TOKEN`, `ANTHROPIC_API_KEY`.

3. **Aplicar migraciones y semilla de catálogo:**
   ```
   npm run db:migrate
   npm run db:seed
   ```

4. **Ejecutar el backend en modo watch:**
   ```
   npm run dev --workspace=apps/backend
   ```
   Servidor en `http://localhost:3000` con `tsx watch` o `nodemon`.

5. **Exponer webhook a Meta** mediante un túnel HTTPS local:
   - Opciones: `ngrok http 3000`, `cloudflared tunnel`, o `localtunnel`.
   - Registrar la URL pública en el dashboard de Meta y validar con `WHATSAPP_VERIFY_TOKEN`.

6. **Probar el flujo end-to-end** enviando un mensaje real desde un número de prueba de WhatsApp Business.

### 2.3 Flujo de desarrollo local — Frontend (Admin)

1. **Iniciar el panel:**
   ```
   npm run dev --workspace=apps/admin
   ```
   Vite expone `http://localhost:5173`.

2. **Proxy al backend** configurado en `vite.config.ts` para evitar CORS en desarrollo (`/api → localhost:3000`).

3. **Autenticación local:** modo dev con un usuario admin precargado vía seed; en producción se evaluará SSO o magic links.

4. **Hot reload** activo para cambios de UI; los tipos compartidos con el backend se importan desde `packages/shared`.

### 2.4 Servicios extendidos locales

| Servicio | Propósito | Cómo se levanta |
|---|---|---|
| PostgreSQL | Persistencia | `docker compose up -d postgres` |
| pgAdmin / DBeaver | Inspección manual de datos | Cliente externo apuntando a `localhost:5432` |
| ngrok / cloudflared | Túnel HTTPS para webhook de WhatsApp | `ngrok http 3000` |
| Redis (opcional, fase 2) | Cache de catálogo y rate limiting | `docker compose up -d redis` |
| Mock de WhatsApp | Pruebas sin tocar Meta (script que simula payloads de webhook) | `npm run mock:whatsapp` |

### 2.5 Estándares de calidad

**Lint y formato**
- ESLint con configuración compartida (`@whatsapp-agent/eslint-config`).
- Prettier para formato; integrado en pre-commit con `lint-staged` + `husky`.

**Tipado**
- TypeScript en modo `strict`. Sin `any` implícitos. Validación de entrada con `zod` en bordes (webhooks, API admin).

**Testing**
- Unit tests con `vitest` (servicios de dominio, parseo de mensajes, mappers).
- Integration tests del backend levantando PostgreSQL en contenedor efímero (Testcontainers).
- E2E del flujo conversacional usando el mock de WhatsApp + base de datos efímera.
- Cobertura objetivo: ≥ 70 % en servicios de dominio; los prompts no se cubren con tests unitarios sino con casos de evaluación documentados.

**Calidad de prompts (Claude)**
- Carpeta `prompts/` con system prompts versionados.
- Suite de evaluación (`npm run eval:prompts`) que ejecuta un set de mensajes representativos contra el modelo y verifica intención detectada y forma de la respuesta. No bloquea CI inicialmente; se usa como gate manual.

**CI**
- GitHub Actions: lint → typecheck → tests → build, en cada PR.
- Ramas protegidas: `main` requiere PR con check verde y una revisión.

**Observabilidad mínima**
- Logger estructurado (`pino`) con `requestId` por mensaje entrante.
- Métrica básica de latencia por llamada a Claude y a Meta.
- Errores enviados a una salida estructurada lista para integrarse con Sentry o similar más adelante.

**Seguridad**
- Verificación de firma `X-Hub-Signature-256` en cada webhook entrante.
- Secretos solo vía variables de entorno; nunca commiteados. `.env.example` con claves vacías.
- Rate limiting por número de teléfono para evitar abuso.

---

## 3. Out of Scope (versión inicial)

- **Multi-idioma.** Sólo español en la primera versión.
- **Pagos dentro del chat.** El agente entrega cotización y registra el pedido; el cobro queda fuera (link externo o handoff humano).
- **Voz y multimedia generada.** No se procesan audios entrantes ni se generan imágenes; archivos adjuntos se acusan recibo y se escalan.
- **Multi-tenant.** Una sola marca/cuenta de WhatsApp Business en esta fase.
- **CRM completo.** No se construye un CRM; sólo se registran conversaciones y pedidos. Integración con CRM externo es futura.
- **Recomendador avanzado.** El agente responde sobre el catálogo existente; no hay personalización por historial de compras en v1.
- **App móvil del panel admin.** Solo web responsive.
- **Despliegue productivo y SRE.** El alcance del documento es el producto y el dev loop local; la estrategia de despliegue se define en un documento aparte.

---

## 4. Technical Context

### 4.1 Flujo end-to-end de un mensaje

```
Cliente (WhatsApp)
   │  mensaje
   ▼
Meta Cloud API ──webhook POST──▶ Backend Express
                                    │
                                    │ 1. Verificar firma
                                    │ 2. Persistir mensaje + recuperar conversación
                                    │ 3. Construir contexto (historial + catálogo relevante)
                                    │ 4. Llamar a Claude (intención + respuesta)
                                    │ 5. Si intención = cotización/pedido → consultar/escribir en PostgreSQL
                                    │ 6. Construir respuesta final
                                    ▼
                                  Meta Cloud API ──▶ Cliente (WhatsApp)
```

### 4.2 Modelo de datos (núcleo)

- **products**: `id`, `sku`, `nombre`, `descripcion`, `precio`, `stock`, `activo`.
- **conversations**: `id`, `phone`, `estado`, `created_at`, `last_message_at`.
- **messages**: `id`, `conversation_id`, `direccion` (in/out), `contenido`, `intent` (detectado), `created_at`.
- **orders**: `id`, `conversation_id`, `phone`, `estado` (borrador, confirmado, cancelado), `total`, `created_at`.
- **order_items**: `id`, `order_id`, `product_id`, `cantidad`, `precio_unitario`.

### 4.3 Integración con Claude

- Modelo: **`claude-sonnet-4-6`** como motor único de conversación, comprensión de intención y generación de respuestas en todas las etapas (no se introduce un segundo modelo para clasificación en v1).
- **Prompt caching** para el prefijo cacheable de cada llamada. **Importante:** en la API de Anthropic el prefijo cacheable es `system + tools` (en ese orden). Cuando se introducen tools (Slice 3), el `cache_control` debe colocarse al final del bloque de `tools`, no en el `system`, para que el cache cubra ambos. Si se modifica cualquier definición de tool, el cache se invalida — versionar el conjunto de tools junto con el system prompt.
- **Tool use** para acciones estructuradas: `consultar_producto`, `crear_pedido`, `consultar_pedido`, `confirmar_pedido`, `cancelar_pedido`, `escalar_a_humano`.
- Soporte de **bloques `tool_use` en paralelo** en una sola respuesta (Claude puede emitir varios). El runner los ejecuta concurrentemente y devuelve un único bloque `tool_result` por id.
- Temperatura baja (0.2–0.4) para consistencia comercial.
- Versión del SDK `@anthropic-ai/sdk` pinneada (`~0.x.y`) en `package.json`.

### 4.4 Integración con WhatsApp Cloud API

- Webhook de mensajes entrantes (`POST /webhook`).
- Verificación inicial vía `GET /webhook` con `WHATSAPP_VERIFY_TOKEN`.
- Envío de mensajes salientes a `https://graph.facebook.com/v{version}/{phone_id}/messages`.
- Plantillas aprobadas por Meta para mensajes proactivos fuera de la ventana de 24 h.

### 4.5 Restricciones y consideraciones

- Ventana de servicio al cliente de 24 h impuesta por Meta para mensajes libres.
- **Latencia objetivo (P95 end-to-end) por tipo de turno:**
  - Saludo / charla sin tools: **< 4 s** (cache caliente).
  - Cotización con 1 tool call: **< 6 s**.
  - Pedido (consultar + crear con confirmación): **< 8 s**.
  Estos SLO son sin streaming; si los superamos consistentemente en producción, evaluar streaming + mensaje intermedio "procesando…".
- **Persistencia obligatoria del mensaje entrante antes de hacer ack 200 a Meta**, para no perder trazabilidad si el proceso muere entre el ack y el procesamiento (ver §4.8).

### 4.6 Concurrencia y consistencia conversacional

WhatsApp permite que un cliente envíe varios mensajes en ráfaga (3 mensajes en 2 s es común). Express los procesa en paralelo. Sin protección, tres handlers cargan el mismo "historial reciente" estancado, hacen tres llamadas a Claude y producen 2–3 respuestas que se solapan, además de eventuales pedidos duplicados.

**Modelo de serialización por conversación:**
- Antes de "cargar historial → llamar a Claude → persistir respuesta", cada handler toma un **`pg_advisory_xact_lock(hashtext(phone))`** dentro de una transacción.
- Alternativa más liviana en MVP local: **cola in-process por `phone`** (Map\<phone, Promise\>) que serializa los turnos.
- El lock libera al final de la transacción, no antes. Las respuestas a un mismo cliente quedan en orden de llegada, sin solapes.
- Trade-off: si un turno queda colgado en Claude (>15 s), bloquea el siguiente; el timeout de Claude (15 s) y el fallback acotan el daño.

### 4.7 Durabilidad de mensajes (outbox)

Meta exige ack 200 en menos de 5 s y reintenta sólo si no recibe ack. Si nuestro servidor responde 200 y crashea antes de persistir, el mensaje se pierde silenciosamente.

**Regla:** el mensaje entrante se persiste en `messages` (TX corta, < 200 ms) **antes** del ack 200. El procesamiento (llamar a Claude, enviar respuesta) ocurre asíncronamente *después* del ack. Si el proceso muere tras el ack pero antes de responder, al reiniciarse un sweeper detecta mensajes `in` sin `out` posterior y los re-procesa (idempotente por `whatsapp_msg_id` y por estado de conversación).

Implementación mínima:
- INSERT del entrante con `whatsapp_msg_id UNIQUE` + `ON CONFLICT DO NOTHING RETURNING id` ⇒ idempotente ante reintentos de Meta.
- El procesamiento posterior se publica a una "cola" (in-process en MVP, vía `setImmediate`) tras el commit del INSERT.
- Sweeper opcional al boot: re-encolar mensajes entrantes recientes (últimos 10 min) sin respuesta saliente.

### 4.8 Moneda y zona horaria

- **Moneda:** se manejan precios en una sola moneda configurable vía `CURRENCY` (default `CLP`). El formato de presentación (separadores de miles, símbolo) deriva de la moneda. Multi-currency queda fuera de v1.
- **Zona horaria:** `BUSINESS_TZ` (default `America/Santiago`) controla todo lo que el cliente ve con fecha (referencias `ORD-YYYYMMDD-...`, mensajes con horas). El servidor opera en UTC; sólo la presentación se convierte.

### 4.9 LLM Operations

Disciplina operativa para que el componente LLM sea construible, depurable y controlable en costo. Implementación concreta en **Slice 8.5 — Endurecimiento LLM** y referencia secundaria en §4.3.

**Prompt versioning**
- Cada prompt vive en `apps/backend/src/prompts/` como módulo TypeScript exportando una función pura que recibe el contexto y devuelve los `messages`/`system` a enviar a Claude.
- Identificador semántico estable: `<ámbito>.<propósito>.v<n>` — ej. `system.sales.v3`, `tool.consultar_producto.v1`.
- Cambio de comportamiento = nueva versión; no se sobreescribe la anterior. Git da el historial; el `prompt_id` versionado da trazabilidad en runtime.
- Cada llamada a Claude registra `prompt_id` y `prompt_version` en la tabla `llm_traces`.

**Eval pipeline**
- Golden set en `evals/cases.yaml`: 30–50 casos representativos (saludo, cotización ambigua, pedido multi-producto, intento de fraude, escalado, mensaje fuera de dominio).
- Cada caso define: `input`, `contexto_previo` (turnos anteriores), y `aserciones` (intención esperada, tool esperado, regex sobre la respuesta, criterios libres evaluados con LLM-as-judge usando `claude-haiku-4-5`).
- Runner `npm run eval` corre el set contra el prompt actual y produce reporte (pass/fail, latencia, costo) comparado contra baseline.
- Manual antes de mergear cambios de prompt; no bloquea CI inicialmente porque consume API.

**Observabilidad LLM-específica**
- Tabla `llm_traces` en PostgreSQL con todo lo que importa para depurar una conversación: identificadores (`conversation_id`, `message_id`, `prompt_id`, `prompt_version`), modelo, tokens (`input`, `output`, `cache_read`, `cache_creation`), `latency_ms`, `tools_called` (jsonb), `input` y `output` completos (jsonb), `cost_usd`, `created_at`.
- Vista en el panel admin con drill-down: conversación → turnos → trace por turno (prompt usado, tools llamados, costo).
- Sin Langfuse/Helicone en v1; migración a Langfuse cuando el volumen justifique el costo operativo (estimación: > 5k traces/día).

**Cost tracking**
- Tarifas de `claude-sonnet-4-6` codificadas como constantes (`INPUT_USD_PER_MTOK`, `OUTPUT_USD_PER_MTOK`, `CACHE_READ_USD_PER_MTOK`, `CACHE_WRITE_USD_PER_MTOK`).
- Helper calcula y persiste `llm_traces.cost_usd` por llamada.
- Vistas agregadas en admin: costo por día, por conversación, top 10 conversaciones más caras.
- **Guardrail por conversación:** si supera N tokens acumulados (configurable, default 50k), el agente cierra el turno con mensaje de cortesía y dispara `escalar_a_humano`. Evita loops infinitos costosos.
- **Alerta diaria:** si el gasto del día supera umbral (ej. USD 20), log de nivel `warn`. Más adelante: webhook a Slack.

---

## 5. Acceptance Criteria

La primera versión se considera aceptada cuando:

1. **Webhook verificado.** Meta valida correctamente el endpoint en el dashboard.
2. **Conversación básica.** Un cliente puede saludar y recibir una respuesta coherente del agente en español.
3. **Cotización.** Dado un producto del catálogo, el cliente puede preguntar precio y disponibilidad y recibir una respuesta consistente con la base de datos.
4. **Pedido.** El cliente puede solicitar uno o más productos; el sistema crea un pedido en estado `borrador` y devuelve un resumen con total y referencia.
5. **Confirmación de pedido.** El cliente puede confirmar el pedido y queda en estado `confirmado` en la base de datos.
6. **Persistencia.** Cada mensaje entrante y saliente queda registrado con `conversation_id` y `phone`.
7. **Escalado a humano.** Frases como "quiero hablar con una persona" disparan el tool de escalado y el agente deja de responder en esa conversación.
8. **Latencia.** P95 de respuesta < 4 s en pruebas locales contra Claude real.
9. **Seguridad.** Webhooks sin firma válida son rechazados con 401.
10. **Calidad.** Lint, typecheck y tests pasan en CI; cobertura ≥ 70 % en servicios de dominio.
11. **Admin.** Un administrador puede autenticarse, listar y editar productos, y ver el historial de conversaciones recientes desde el panel.

---

## 6. Risks and Assumptions

### 6.1 Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Aprobación lenta de Meta para WhatsApp Business API | Bloquea pruebas reales | Iniciar registro temprano; usar número de pruebas de Meta entretanto |
| Alucinaciones de precio o stock por el LLM | Pérdida de confianza, errores comerciales | Forzar tool use para cualquier dato de catálogo; nunca dejar que el modelo "invente" valores |
| Latencia de Claude en horas pico | Conversación se siente lenta | Prompt caching, streaming si aplica, mensaje intermedio "escribiendo…" |
| Costo de tokens superior a lo presupuestado | Margen comercial | Prompt caching agresivo del system prompt + catálogo, límite de turnos por conversación, monitoreo de tokens por conversación |
| Cambios incompatibles en WhatsApp Cloud API | Refactor inesperado | Adaptador delgado en `services/whatsapp`; pinear versión en URL |
| Datos sensibles del cliente en logs | Riesgo legal/privacidad | Redacción de PII en logger; no persistir más de lo necesario |
| Spam o abuso desde números desconocidos | Costo y ruido | Rate limiting por `phone`; lista de bloqueo |
| Mensajes en ráfaga producen respuestas solapadas o pedidos duplicados | Pérdida de confianza, doble cobro | Lock por conversación (§4.6) antes de cargar historial / llamar a Claude |
| Crash entre ack 200 y procesamiento pierde el mensaje | Cliente no recibe respuesta y queda fuera de trazabilidad | Persistir entrante antes del ack (§4.7) + sweeper opcional al boot |
| Precio cambia entre cotización y creación de pedido | Cliente reclama por precio diferente | `precio_unitario_esperado` en `crear_pedido`; rechazar si difiere |
| Stock se vende dos veces entre borrador y confirmación | Faltante físico, reclamos | Revalidar stock al confirmar (Slice 5); aceptar overselling explícito si se decide |
| Tarifas de Anthropic hardcodeadas se desactualizan | Cálculo de costo erróneo | Revisión trimestral en runbook; alerta si `cache_read` es 0 cuando debería ser positivo |

### 6.2 Supuestos

- El cliente cuenta con una cuenta verificada de WhatsApp Business y acceso a Meta for Developers.
- Existe un catálogo digital (CSV o tabla) que se puede cargar como semilla inicial.
- El equipo dispone de una `ANTHROPIC_API_KEY` con cupo suficiente para desarrollo y QA y acceso a `claude-sonnet-4-6`.
- El idioma de los clientes finales es español; no hay requerimiento de inglés en esta fase.
- El volumen inicial es bajo a medio (decenas de conversaciones simultáneas), suficiente para una sola instancia del backend.
- El handoff a humano se resuelve fuera del producto (Slack, email, teléfono); el sistema solo lo dispara y marca la conversación.

---

## 7. Slices

Cada slice es una entrega vertical funcional que se puede demostrar de punta a punta.

### Slice 0 — Andamiaje
- Monorepo, TypeScript, ESLint, Prettier, CI mínima.
- `docker-compose` con PostgreSQL.
- Migraciones iniciales y seed de productos.
- Endpoint `/health` y logger estructurado.

**Demo:** `curl /health` responde OK; `psql` muestra catálogo cargado.

### Slice 1 — Webhook de WhatsApp + eco
- Verificación `GET /webhook`.
- Recepción `POST /webhook` con validación de firma.
- Persistencia de mensaje entrante.
- Respuesta de eco enviada a Meta.

**Demo:** un mensaje real desde WhatsApp recibe el mismo texto de vuelta y queda registrado en `messages`.

### Slice 2 — Conversación con Claude
- Servicio `claude` con system prompt versionado.
- Construcción de contexto con historial reciente de la conversación.
- Respuesta generada por Claude reemplaza al eco.
- Prompt caching habilitado.

**Demo:** un saludo recibe respuesta natural en español; el modelo recuerda el turno anterior dentro de la misma conversación.

### Slice 3 — Cotización (consulta de catálogo)
- Tool `consultar_producto` expuesto a Claude.
- Búsqueda por nombre/SKU en PostgreSQL.
- Formato de respuesta con precio y disponibilidad.

**Demo:** "¿cuánto cuesta el producto X?" devuelve precio real desde la base.

### Slice 4 — Creación de pedido
- Tool `crear_pedido` que persiste `orders` + `order_items` en estado `borrador`.
- Resumen del pedido enviado al cliente con total y referencia.

**Demo:** "quiero 2 X y 1 Y" crea un pedido borrador y devuelve resumen.

### Slice 5 — Confirmación y consulta de pedido
- Tool `consultar_pedido` por referencia o por número del cliente.
- Transición de `borrador` → `confirmado` por intención del cliente.

**Demo:** "confirmo" cambia el estado y el agente responde con el pedido confirmado.

### Slice 6 — Escalado a humano
- Tool `escalar_a_humano` que marca la conversación como `humano` y silencia al agente.
- Notificación interna (log, webhook a Slack, email — según se decida).

**Demo:** "quiero hablar con una persona" detiene al agente y notifica al equipo.

### Slice 7 — Panel admin (catálogo + conversaciones)
- Login básico.
- CRUD de productos.
- Listado de conversaciones recientes con sus últimos mensajes.

**Demo:** un admin edita un producto desde la UI y la nueva información aparece en la siguiente cotización.

### Slice 8 — Endurecimiento
- Rate limiting por número.
- Métricas de latencia y costo de tokens.
- Suite de evaluación de prompts ejecutable manualmente.
- Documentación de runbook básico (errores comunes, cómo rotar tokens).

**Demo:** stress test simulado muestra rate limit activo y métricas registradas.

### Slice 8.5 — Endurecimiento LLM
Implementación concreta de la sección **4.9 LLM Operations**.
- Tabla `llm_traces` y wrapper que registra cada llamada a Claude (tokens, latencia, costo, tools, input/output).
- Estructura `apps/backend/src/prompts/` con identificadores versionados (`prompt_id` + `prompt_version`).
- Constantes de tarifa de Sonnet 4.6 y cálculo de costo por llamada.
- Golden set inicial (10 casos) en `evals/cases.yaml` y runner `npm run eval`.
- Vista en el panel admin: drill-down de conversación a sus traces; agregados de costo por día y top 10 conversaciones caras.
- Guardrail: si una conversación supera el umbral de tokens, el agente escala a humano automáticamente.

**Demo:** una conversación de prueba genera traces visibles en el admin con tokens, latencia y costo; `npm run eval` corre el golden set y produce reporte; al forzar una conversación a superar el umbral, el agente escala.

---

## Anexos

- **A. Variables de entorno.** Lista completa en `.env.example`. Globales clave: `BUSINESS_TZ` (default `America/Santiago`), `CURRENCY` (default `CLP`).
- **B. Convenciones de prompts.** A definir en `apps/backend/src/prompts/README.md`.
- **C. Política de PII y logs.** Implementación concreta en Slice 8 (redact con `pino`); este anexo recopila la política a medida que se vaya formalizando.
