# Slice 6 — Escalado a humano

## Objetivo
Reconocer cuando el agente no debe seguir respondiendo (cliente lo pide explícitamente, frustración detectada, o caso fuera de scope) y entregar la conversación a un humano. La conversación queda silenciada para el agente y notificada al equipo.

## Dependencias
- Slice 5 completo
- Variables nuevas (opcionales): `SLACK_WEBHOOK_URL` o `ESCALATION_EMAIL`

## Alcance
**Incluye:**
- Tool `escalar_a_humano` con `motivo` y `urgencia`
- Transición de `conversations.estado` → `humano`
- Backend deja de invocar a Claude para conversaciones en estado `humano` (ack silencioso o mensaje fijo)
- Notificación interna: log estructurado + opcionalmente webhook Slack o email
- System prompt v5: criterios para escalar
- Endpoint admin (placeholder, se completa en Slice 7) `POST /admin/conversations/:id/reactivar` para devolver a `activa`

**No incluye:** UI de bandeja de operadores (se gestiona externamente: Slack/email + admin de Slice 7).

## Diseño técnico

### Tool
```ts
{
  name: 'escalar_a_humano',
  description: 'Marca la conversación como necesitando atención humana. Úsalo cuando el cliente lo pida explícitamente, cuando detectes frustración, o cuando el caso esté fuera de tu alcance (reclamos, fraudes, dudas legales, problemas técnicos del producto).',
  input_schema: {
    type: 'object',
    properties: {
      motivo: {
        type: 'string',
        enum: ['solicitud_cliente','frustracion','fuera_de_scope','reclamo','otro'],
      },
      detalle: { type: 'string', description: 'Resumen de 1-2 frases del contexto para el operador' },
      urgencia: { type: 'string', enum: ['normal','alta'], default: 'normal' },
    },
    required: ['motivo','detalle'],
  },
}
```

### Lógica del tool
1. UPDATE `conversations SET estado='humano', last_message_at=NOW() WHERE id=$1 AND estado != 'humano'`.
2. Insertar `messages` tipo `system` con `contenido='[ESCALADO] motivo=X detalle=Y'` para trazabilidad.
3. Notificar:
   - Log estructurado `{event:'conversation.escalated', conversationId, phone, motivo, urgencia, detalle}`.
   - Si `SLACK_WEBHOOK_URL` definido: POST con bloque legible (link al admin).
   - Si `ESCALATION_EMAIL` definido: enviar email vía SMTP (configuración mínima, opcional).
4. Devolver `tool_result` `{ok:true, conversation_id}`.
5. Después de este tool, el agente envía un mensaje de cierre al cliente: "Te conectaré con una persona de nuestro equipo. Pronto te contactarán por aquí mismo."

### Comportamiento del webhook con conversación en `estado='humano'`
- Webhook recibe mensaje del cliente normalmente y lo persiste.
- **NO llama a Claude.** Salta el bloque de respuesta.
- No envía respuesta automática (silencio total) **excepto** cuando han pasado > 30 min desde el escalado: enviar mensaje fijo "Estamos revisando, te contactaremos pronto" para evitar percepción de abandono.

### Reactivación
- `POST /admin/conversations/:id/reactivar` (autenticado, Slice 7):
  - UPDATE `estado='activa'`.
  - Inserta `messages` system `[REACTIVADA por <admin>]`.

### System prompt v5 — adiciones
- Lista explícita de disparadores: solicitud directa ("hablar con humano", "operador", "persona"), frustración (palabras fuertes, repetición de queja), reclamo, mención de fraude/cargo no autorizado, problema técnico físico del producto.
- "Antes de escalar, intenta UNA aclaración. Si el cliente insiste o el tono escala, llama `escalar_a_humano` sin más vueltas."
- "Después de escalar, no respondas más en este turno excepto el mensaje de cierre."

## Archivos a crear / modificar
- `apps/backend/src/services/claude/tools.ts` (añadir tool)
- `apps/backend/src/services/conversations.ts` (función `escalateToHuman`, helper `isHumanHandled`)
- `apps/backend/src/services/notifications/slack.ts` (opcional)
- `apps/backend/src/services/notifications/email.ts` (opcional)
- `apps/backend/src/routes/webhook.ts` (rama: si conversación humano, no llamar Claude)
- `apps/backend/src/prompts/system/sales.v5.ts`
- `apps/backend/src/prompts/index.ts`
- `apps/backend/tests/conversations.escalate.test.ts`

## Criterios de aceptación
- [ ] "Quiero hablar con una persona" → `conversations.estado='humano'` y mensaje de cierre al cliente.
- [ ] Tras escalar, mensaje siguiente del cliente no provoca respuesta del agente (silencio o mensaje fijo si > 30 min).
- [ ] Mensajes posteriores del cliente sí quedan registrados en `messages` (no se pierden).
- [ ] Log estructurado `conversation.escalated` se emite con `motivo` y `detalle`.
- [ ] Si `SLACK_WEBHOOK_URL` configurado: llega notificación con phone, motivo, link admin.
- [ ] Frase neutra ("¿qué tal?") no escala.
- [ ] Frustración explícita ("esto es una basura, nadie me responde") sí escala con `motivo=frustracion`.
- [ ] Solicitud directa ("operador") escala en el primer turno.
- [ ] Reactivar la conversación desde admin la devuelve a `activa` y el agente vuelve a responder.
- [ ] El sistema escribe un mensaje system en `messages` con `[ESCALADO]`.

## Tests requeridos
- **Unit:** detector de "está en humano" — `isHumanHandled(conversation)`.
- **Integración:** `webhook` con conversación ya en humano → asserts: no se llamó a Claude; sí se persistió mensaje entrante.
- **Integración:** `escalateToHuman` cambia estado y emite log; mock de Slack recibe POST.
- **E2E manual:** "quiero hablar con un humano" → ver Slack/log; reactivar → conversación responde de nuevo.

## Demo
```
# Desde teléfono:
"Quiero hablar con un humano"
# Esperado: agente cierra, log Slack/console, estado='humano'
"hola?"
# Esperado: silencio (o mensaje fijo si > 30 min)
```
```sql
SELECT estado FROM conversations WHERE phone = '<phone>';
SELECT direccion, contenido FROM messages WHERE conversation_id=<id> ORDER BY created_at;
```

## Riesgos del slice
- **Falsos positivos de frustración:** mejor escalar de más al inicio que perder cliente; ajustar con datos reales.
- **Cliente enojado por silencio:** mensaje fijo a los 30 min mitiga.
- **Slack webhook caído rompe el flujo:** la notificación es best-effort; nunca bloquea el escalado en sí.
- **Operador olvida reactivar:** considerar timeout (deuda futura) que devuelva a `activa` tras 24 h.
