# Slice 6 — Escalado a humano

## Objetivo
Reconocer cuando el agente no debe seguir respondiendo (cliente lo pide explícitamente, frustración detectada, o caso fuera de scope) y entregar la conversación a un humano. La conversación queda silenciada para el agente y notificada al equipo.

## Dependencias
- Slice 5 completo
- Variables nuevas (opcionales): `SLACK_WEBHOOK_URL` o `ESCALATION_EMAIL`

## Alcance
**Incluye:**
- Tool `escalar_a_humano` con `motivo` y `urgencia`
- **Función pura compartida `escalateToHuman(conversationId, motivo, detalle, urgencia)`** que el handler del tool y otros disparadores (guardrail de tokens en Slice 8.5, eventual reintento manual) invocan para no duplicar lógica
- Transición de `conversations.estado` → `humano`
- Backend deja de invocar a Claude para conversaciones en estado `humano`
- **Lazy check de "30 min sin respuesta"** al recibir mensaje del cliente: si el último `out` es de hace > 30 min y la conversación está en `humano`, enviar mensaje fijo "Estamos revisando, te contactaremos pronto"
- Marcador system `[HUMAN HANDOFF inicio]` al escalar y `[HUMAN HANDOFF fin]` al reactivar, para que `loadHistory` (Slice 2) no le envíe a Claude el "vacío" entre escalado y reactivación como si fueran turnos del cliente sin respuesta
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
El handler del tool delega en la función pura `escalateToHuman`:

```ts
async function escalateToHuman(args: {
  conversationId: number;
  phone: string;
  motivo: 'solicitud_cliente'|'frustracion'|'fuera_de_scope'|'reclamo'|'otro';
  detalle: string;
  urgencia?: 'normal'|'alta';
  source: 'tool'|'guardrail'|'admin';
}): Promise<{ok: true, alreadyEscalated: boolean}>
```

1. UPDATE `conversations SET estado='humano', last_message_at=NOW() WHERE id=$1 AND estado != 'humano' RETURNING id`. Si rowCount=0 → ya estaba en humano (`alreadyEscalated:true`); devolver sin notificar de nuevo.
2. INSERT `messages` (`tipo='system'`, `contenido='[HUMAN HANDOFF inicio] motivo=X detalle=Y source=Z'`) para trazabilidad y para que `loadHistory` ignore correctamente el segmento.
3. Notificar (best-effort, no bloquea):
   - Log estructurado `{event:'conversation.escalated', conversationId, phone, motivo, urgencia, detalle, source}`.
   - Si `SLACK_WEBHOOK_URL` definido: POST con bloque legible (link al admin). Errores se loguean pero no propagan.
   - Si `ESCALATION_EMAIL` definido: enviar email vía SMTP. Errores no propagan.
4. Devolver `{ok:true, alreadyEscalated:false}`.

El handler del tool, sobre el resultado, hace que el modelo emita un mensaje de cierre: *"Te conectaré con una persona de nuestro equipo. Pronto te contactarán por aquí mismo."* La función pura es reutilizada por:
- Slice 8.5 — guardrail de tokens (cuando una conversación excede `CONVERSATION_TOKEN_BUDGET`).
- Endpoint admin `POST /admin/conversations/:id/escalar` (futuro, no en este slice).

### Comportamiento del webhook con conversación en `estado='humano'`
- Webhook recibe mensaje del cliente normalmente y lo persiste (`messages` con `direccion='in'`, `tipo='text'`).
- **NO llama a Claude.** Salta el bloque de respuesta.
- **Lazy check de 30 min:** al recibir el mensaje, consultar `MAX(created_at) FILTER (WHERE direccion='out')` de la conversación. Si > 30 min atrás → enviar mensaje fijo "Estamos revisando, te contactaremos pronto" (y persistirlo como `out` para no spamear: el siguiente mensaje del cliente vuelve a calcular contra este). Esto evita un timer/cron y se computa solo cuando se necesita.
- Si el último `out` es < 30 min, silencio total.

### Reactivación
- `POST /admin/conversations/:id/reactivar` (autenticado, Slice 7):
  - UPDATE `estado='activa'`.
  - Inserta `messages` system `[HUMAN HANDOFF fin] reactivada_por=<admin_email>`.
- **Importante:** los mensajes del cliente acumulados durante `estado='humano'` quedan en `messages` pero `loadHistory` (Slice 2) ya los excluye **porque están entre los marcadores `[HUMAN HANDOFF inicio]` y `[HUMAN HANDOFF fin]`**. Sin esto, al primer mensaje post-reactivación el modelo recibiría 5–10 turnos del cliente sin respuestas y trataría de "ponerse al día" extrañamente.
- Implementación en `loadHistory`: además de excluir `tipo='system'`, omitir mensajes cuyo rango `[created_at_start, created_at_end]` esté dentro de un par `[HANDOFF inicio]..[HANDOFF fin]` aún abierto o cerrado. Equivalentemente: tomar los `tipo='text'` cuya posición no esté entre marcadores.

### System prompt v5 — adiciones
- Lista explícita de disparadores: solicitud directa ("hablar con humano", "operador", "persona"), frustración (palabras fuertes, repetición de queja), reclamo, mención de fraude/cargo no autorizado, problema técnico físico del producto.
- "Antes de escalar, intenta UNA aclaración. Si el cliente insiste o el tono escala, llama `escalar_a_humano` sin más vueltas."
- "Después de escalar, no respondas más en este turno excepto el mensaje de cierre."

## Archivos a crear / modificar
- `apps/backend/src/services/claude/tools.ts` (añadir tool)
- `apps/backend/src/services/conversations/escalate.ts` (función pura `escalateToHuman` reutilizable)
- `apps/backend/src/services/conversations/state.ts` (helper `isHumanHandled`)
- `apps/backend/src/services/messages.ts` (modificar `loadHistory` para respetar marcadores HANDOFF)
- `apps/backend/src/services/notifications/slack.ts` (opcional)
- `apps/backend/src/services/notifications/email.ts` (opcional)
- `apps/backend/src/services/dispatcher.ts` (rama: si conversación en humano, lazy-check 30min y skip Claude)
- `apps/backend/src/prompts/system/sales.v5.ts`
- `apps/backend/src/prompts/index.ts`
- `apps/backend/tests/conversations.escalate.test.ts`
- `apps/backend/tests/messages.loadHistory.handoff.test.ts`

## Criterios de aceptación
- [ ] "Quiero hablar con una persona" → `conversations.estado='humano'` y mensaje de cierre al cliente.
- [ ] Tras escalar, mensaje siguiente del cliente no provoca respuesta del agente (silencio si último out < 30 min).
- [ ] Si el último `out` es de hace > 30 min, el siguiente mensaje del cliente recibe el mensaje fijo (una sola vez por ventana de 30 min).
- [ ] Mensajes posteriores del cliente sí quedan registrados en `messages` (no se pierden).
- [ ] Log estructurado `conversation.escalated` se emite con `motivo` y `detalle`.
- [ ] Si `SLACK_WEBHOOK_URL` configurado: llega notificación con phone, motivo, link admin. Slack caído NO bloquea el escalado.
- [ ] Frase neutra ("¿qué tal?") no escala.
- [ ] Frustración explícita ("esto es una basura, nadie me responde") sí escala con `motivo=frustracion`.
- [ ] Solicitud directa ("operador") escala en el primer turno.
- [ ] Reactivar la conversación desde admin la devuelve a `activa`, inserta `[HUMAN HANDOFF fin]` y el agente vuelve a responder en el siguiente turno **sin "ponerse al día" sobre los mensajes silenciados**.
- [ ] El sistema escribe `[HUMAN HANDOFF inicio]` (`tipo='system'`) al escalar y `[HUMAN HANDOFF fin]` al reactivar.
- [ ] `loadHistory` no envía a Claude los mensajes entre marcadores HANDOFF (test unitario).
- [ ] Función pura `escalateToHuman` invocada desde `source='guardrail'` produce el mismo efecto que desde `source='tool'` (test).

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
- **Mensajes acumulados durante humano confunden al modelo al reactivar:** mitigado con marcadores `[HUMAN HANDOFF inicio/fin]` que `loadHistory` filtra.
- **Escalado disparado dos veces (tool + guardrail) en el mismo turno:** la función pura es idempotente (UPDATE condicional `estado != 'humano'` + `alreadyEscalated:true`).
