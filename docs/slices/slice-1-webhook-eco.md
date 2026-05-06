# Slice 1 — Webhook de WhatsApp + eco

## Objetivo
Conectar el backend a WhatsApp Cloud API: verificación inicial, recepción segura de mensajes con validación de firma, persistencia y respuesta de eco. Primer mensaje real de WhatsApp end-to-end sin Claude todavía.

## Dependencias
- Slice 0 completo
- Cuenta WhatsApp Business API (Meta) con número de pruebas y `App Secret`
- Túnel HTTPS local (`ngrok`, `cloudflared`)
- Variables nuevas: `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_ID`, `WHATSAPP_VERIFY_TOKEN`, `WHATSAPP_APP_SECRET`, `WHATSAPP_API_VERSION` (default `v21.0`)

## Alcance
**Incluye:**
- `GET /webhook` para verificación de Meta (challenge)
- `POST /webhook` con validación HMAC `X-Hub-Signature-256`
- Tablas `conversations` y `messages`
- Servicio `whatsapp` para enviar mensajes salientes (texto plano)
- Eco: el contenido recibido se devuelve idéntico al cliente
- **Persistencia del entrante ANTES del ack 200** (durabilidad — §4.7 de `solucion.md`)
- Idempotencia por `whatsapp_msg_id` con `ON CONFLICT DO NOTHING`
- **Lock por conversación** (`pg_advisory_xact_lock`) para serializar mensajes en ráfaga (§4.6 de `solucion.md`)

**No incluye:** Claude, tools, lógica conversacional, mensajes multimedia generados, plantillas proactivas.

## Diseño técnico

### Modelo de datos (delta)
```sql
CREATE TABLE conversations (
  id              BIGSERIAL PRIMARY KEY,
  phone           TEXT NOT NULL UNIQUE,
  estado          TEXT NOT NULL DEFAULT 'activa',
  -- estados: activa | humano | cerrada
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_message_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_conversations_estado ON conversations (estado);

CREATE TABLE messages (
  id              BIGSERIAL PRIMARY KEY,
  conversation_id BIGINT NOT NULL REFERENCES conversations(id),
  direccion       TEXT NOT NULL CHECK (direccion IN ('in','out')),
  contenido       TEXT NOT NULL,
  whatsapp_msg_id TEXT UNIQUE,
  tipo            TEXT NOT NULL DEFAULT 'text'
                  CHECK (tipo IN ('text','media','system')),
  -- text:   mensaje conversacional (in/out)
  -- media:  archivo entrante (imagen, audio, etc.) acusado pero no procesado
  -- system: marcador interno (ESCALADO, REACTIVADA). Se filtra de loadHistory.
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_messages_conv_created ON messages (conversation_id, created_at DESC);
```

> **Nota sobre `intent`.** Se eliminó deliberadamente la columna `intent` que aparecía en el draft inicial. La fuente de verdad sobre qué tools se invocaron en cada turno es `llm_traces.tools_called` (Slice 8.5). Mantener una columna heurística adicional crearía dos lugares con datos potencialmente discordantes. Si en el admin se necesita una etiqueta de "intención" del mensaje saliente, se calcula como vista derivada sobre `llm_traces`.

### Verificación GET
- Query: `hub.mode=subscribe`, `hub.verify_token=<token>`, `hub.challenge=<n>`.
- Si `verify_token === WHATSAPP_VERIFY_TOKEN` → responder `hub.challenge` en texto con 200.
- Si no → 403.

### Validación de firma POST
- Header: `X-Hub-Signature-256: sha256=<hex>`.
- Calcular `HMAC-SHA256(WHATSAPP_APP_SECRET, raw_body_buffer)` y comparar con `crypto.timingSafeEqual`.
- Express debe exponer el `rawBody` (middleware `express.json({verify: ...})`).
- 401 si firma inválida o ausente.

### Flujo principal POST

> Orden estricto por durabilidad: **persistir antes de hacer ack** (§4.7 de `solucion.md`). El ack a Meta debe ocurrir en < 5 s; el INSERT del entrante toma < 200 ms en condiciones normales, así que cabe holgado en la ventana.

1. **Validar firma** → si falla, 401 inmediato.
2. **Parsear `entry[].changes[].value.messages[]`** (skip si no hay mensajes; por ejemplo status updates de delivery → ack 200 vacío).
3. **TX corta de persistencia (síncrona, antes del ack):**
   - `INSERT ... ON CONFLICT (phone) DO UPDATE SET last_message_at = NOW() RETURNING id` sobre `conversations`.
   - `INSERT INTO messages (...) VALUES (...) ON CONFLICT (whatsapp_msg_id) DO NOTHING RETURNING id` sobre `messages`. Si `RETURNING` no devuelve fila → es reintento de Meta sobre un mensaje ya procesado: log `info` y skip al paso 5.
   - Commit.
4. **Responder 200 a Meta** (cuerpo vacío).
5. **Dispatch asíncrono del procesamiento** (en MVP `setImmediate`; en producción cola durable). Cada job:
   - Toma lock por conversación: `BEGIN; SELECT pg_advisory_xact_lock(hashtext($phone));` — serializa los turnos del mismo cliente.
   - Si `tipo='text'`: enviar eco vía `whatsapp.sendText(phone, contenido)`.
   - INSERT `messages` (`direccion='out'`, `whatsapp_msg_id` devuelto por Meta) dentro de la misma TX.
   - `COMMIT` libera el lock.
6. **Sweeper opcional al boot** (recomendado, no bloqueante): query `messages` de los últimos 10 min con `direccion='in'` que no tengan un `out` posterior en la misma conversación → re-encolar paso 5. Cubre el caso de crash entre ack y send.

### Servicio `whatsapp.sendText`
- POST a `https://graph.facebook.com/{version}/{phone_id}/messages`.
- Header `Authorization: Bearer ${WHATSAPP_TOKEN}`.
- Body: `{messaging_product: "whatsapp", to: phone, type: "text", text: {body}}`.
- Devuelve `messages[0].id`.
- Reintento simple: 1 reintento con backoff 500 ms si 5xx; abandona si 4xx.

### Errores y casos borde
- Payload sin `messages` (status updates de delivery): ack 200 y skip.
- Mensaje no de texto (imagen/audio/video/sticker/location/contact): persistir como `tipo='media'` con `contenido` = JSON serializado del payload original (para no perder info); responder texto fijo "Por ahora solo proceso mensajes de texto".
- Mensaje duplicado (mismo `whatsapp_msg_id`): atrapado por `ON CONFLICT DO NOTHING`; el RETURNING devuelve 0 filas y se hace skip silencioso (log `info` con `event=duplicate_inbound`).
- Fallo persistente al enviar a Meta: log `error`, no bloquear; el mensaje entrante ya quedó persistido y el sweeper lo retomará al próximo boot si fuese necesario.
- Lock contendido (turno previo aún ejecutándose): segundo job espera; si excede `STMT_TIMEOUT` (60 s), aborta con log `error` y se reintenta vía sweeper.

## Archivos a crear / modificar
- `apps/backend/src/routes/webhook.ts`
- `apps/backend/src/services/whatsapp/client.ts`
- `apps/backend/src/services/whatsapp/signature.ts`
- `apps/backend/src/services/whatsapp/types.ts`
- `apps/backend/src/services/conversations.ts`
- `apps/backend/src/services/messages.ts`
- `apps/backend/src/services/locks.ts` (helper `withConversationLock(phone, fn)`)
- `apps/backend/src/services/dispatcher.ts` (cola in-process + sweeper de boot)
- `apps/backend/src/middleware/rawBody.ts`
- `infra/migrations/<timestamp>_conversations_messages.sql`
- `apps/backend/tests/webhook.test.ts`
- `apps/backend/tests/signature.test.ts`
- `apps/backend/tests/concurrency.test.ts` (mensajes en ráfaga sobre el mismo phone)
- `.env.example` (añadir variables nuevas)

## Criterios de aceptación
- [ ] `GET /webhook?hub.mode=subscribe&hub.verify_token=<ok>&hub.challenge=123` → cuerpo `123` con 200.
- [ ] `GET /webhook` con verify_token incorrecto → 403.
- [ ] `POST /webhook` sin header `X-Hub-Signature-256` → 401.
- [ ] `POST /webhook` con firma inválida → 401.
- [ ] `POST /webhook` con firma válida y mensaje de texto → 200 en < 1 s, con el entrante ya persistido al momento del ack.
- [ ] El cliente recibe el mismo texto en su WhatsApp en < 4 s.
- [ ] Mensaje entrante queda en `messages` con `direccion='in'` y `whatsapp_msg_id` poblado.
- [ ] Mensaje saliente queda en `messages` con `direccion='out'` y `whatsapp_msg_id` del envío.
- [ ] Conversación nueva crea fila en `conversations`; existente actualiza `last_message_at`.
- [ ] Reenviar el mismo `whatsapp_msg_id` no duplica filas (`ON CONFLICT DO NOTHING`); 200 igual.
- [ ] Mensaje multimedia recibe respuesta fija sin error y queda como `tipo='media'`.
- [ ] **3 mensajes al mismo phone en < 1 s producen exactamente 3 respuestas, en orden de llegada, sin solapes** (test `concurrency.test.ts` con dispatcher real).
- [ ] **Crash simulado entre ack y send:** mensaje persistido como `in` sin `out`; al reiniciar, el sweeper lo retoma y envía la respuesta.
- [ ] No existe columna `intent` en `messages` (la "intención" se deriva de `llm_traces` en Slice 8.5).

## Tests requeridos
- **Unit:** validador de firma HMAC (vector positivo, vector negativo, header faltante).
- **Integración:** POST a `/webhook` con cuerpo y firma generados en test, contra Postgres efímero, mockeando `fetch` a Meta. Verifica filas en `messages` y llamada a `whatsapp.sendText`.
- **E2E manual:** mensaje real desde teléfono.

## Demo
```bash
docker compose up -d
npm run db:migrate
npm run dev --workspace=apps/backend
ngrok http 3000
# En Meta Developer Console: registrar URL, validar webhook, suscribir a 'messages'
# Enviar "hola" desde el teléfono al número de pruebas
psql "$DATABASE_URL" -c "SELECT direccion, contenido FROM messages ORDER BY created_at DESC LIMIT 4;"
```
Esperado: el teléfono recibe "hola"; `psql` muestra dos filas (`in` y `out`) por turno.

## Riesgos del slice
- **Comparación de firma con `===` en lugar de `timingSafeEqual`:** vulnerabilidad de timing. Test explícito con vector negativo.
- **Persistencia post-ack pierde mensajes:** mitigado moviendo el INSERT del entrante antes del ack 200 + sweeper al boot.
- **Webhook duplicado por reintentos de Meta:** `whatsapp_msg_id UNIQUE` + `ON CONFLICT DO NOTHING` da idempotencia, incluso si la primera ejecución y el reintento corren en paralelo.
- **Mensajes en ráfaga del mismo cliente producen respuestas solapadas:** mitigado con `pg_advisory_xact_lock(hashtext(phone))` en cada job del dispatcher.
- **Token de WhatsApp con TTL corto en sandbox:** documentar rotación en runbook (Slice 8).
