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
- Persistencia de mensajes entrante y saliente
- Idempotencia por `whatsapp_msg_id`

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
  intent          TEXT,
  whatsapp_msg_id TEXT UNIQUE,
  tipo            TEXT NOT NULL DEFAULT 'text',
  -- tipos: text | media | system
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_messages_conv_created ON messages (conversation_id, created_at DESC);
```

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
1. Validar firma → si falla, 401 inmediato.
2. **Responder 200 a Meta inmediatamente** (Meta corta a 5 s y reintenta).
3. Encolar procesamiento (in-process por ahora, vía `setImmediate` o cola simple):
   - Parsear `entry[].changes[].value.messages[]`.
   - Por cada mensaje: upsert `conversations` por `phone`, insertar `messages` (in) con `whatsapp_msg_id`.
   - Si `tipo='text'`: enviar eco vía `whatsapp.sendText(phone, contenido)`.
   - Insertar `messages` (out) con `whatsapp_msg_id` devuelto por Meta.

### Servicio `whatsapp.sendText`
- POST a `https://graph.facebook.com/{version}/{phone_id}/messages`.
- Header `Authorization: Bearer ${WHATSAPP_TOKEN}`.
- Body: `{messaging_product: "whatsapp", to: phone, type: "text", text: {body}}`.
- Devuelve `messages[0].id`.
- Reintento simple: 1 reintento con backoff 500 ms si 5xx; abandona si 4xx.

### Errores y casos borde
- Payload sin `messages` (status updates de delivery): ack 200 y skip.
- Mensaje no de texto (imagen/audio/video): persistir como `tipo='media'` con placeholder; responder texto fijo "Por ahora solo proceso mensajes de texto".
- Mensaje duplicado (mismo `whatsapp_msg_id`): UNIQUE constraint atrapa; log `info` y skip.
- Fallo persistente al enviar a Meta: log `error`, no bloquear; el mensaje entrante ya quedó persistido.

## Archivos a crear / modificar
- `apps/backend/src/routes/webhook.ts`
- `apps/backend/src/services/whatsapp/client.ts`
- `apps/backend/src/services/whatsapp/signature.ts`
- `apps/backend/src/services/whatsapp/types.ts`
- `apps/backend/src/services/conversations.ts`
- `apps/backend/src/services/messages.ts`
- `apps/backend/src/middleware/rawBody.ts`
- `infra/migrations/0002_conversations_messages.sql`
- `apps/backend/tests/webhook.test.ts`
- `apps/backend/tests/signature.test.ts`
- `.env.example` (añadir variables nuevas)

## Criterios de aceptación
- [ ] `GET /webhook?hub.mode=subscribe&hub.verify_token=<ok>&hub.challenge=123` → cuerpo `123` con 200.
- [ ] `GET /webhook` con verify_token incorrecto → 403.
- [ ] `POST /webhook` sin header `X-Hub-Signature-256` → 401.
- [ ] `POST /webhook` con firma inválida → 401.
- [ ] `POST /webhook` con firma válida y mensaje de texto → 200 en < 1 s.
- [ ] El cliente recibe el mismo texto en su WhatsApp en < 4 s.
- [ ] Mensaje entrante queda en `messages` con `direccion='in'` y `whatsapp_msg_id` poblado.
- [ ] Mensaje saliente queda en `messages` con `direccion='out'` y `whatsapp_msg_id` del envío.
- [ ] Conversación nueva crea fila en `conversations`; existente actualiza `last_message_at`.
- [ ] Reenviar el mismo `whatsapp_msg_id` no duplica filas.
- [ ] Mensaje multimedia recibe respuesta fija sin error.

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
- **Bloquear el ack a Meta:** Meta corta a 5 s y reintenta; siempre responder 200 antes de procesar.
- **Webhook duplicado por reintentos de Meta:** `whatsapp_msg_id UNIQUE` da idempotencia.
- **Token de WhatsApp con TTL corto en sandbox:** documentar rotación en runbook (Slice 8).
