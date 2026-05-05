# Slice 5 — Confirmación y consulta de pedido

## Objetivo
Permitir que el cliente confirme un pedido en estado `borrador` (transición a `confirmado`) y consulte el estado de pedidos previos por referencia o por su número de teléfono.

## Dependencias
- Slice 4 completo
- Variables: ninguna nueva

## Alcance
**Incluye:**
- Tool `consultar_pedido` (por referencia o por phone → últimos N)
- Tool `confirmar_pedido` (transición `borrador → confirmado`)
- Tool `cancelar_pedido` (transición `borrador → cancelado`; sólo en estado borrador)
- System prompt v4: maneja confirmación y consulta
- Notificación interna (log estructurado) cuando un pedido pasa a `confirmado`

**No incluye:** modificación de items de un pedido existente (cancelar y recrear), descuento real de stock al confirmar (deuda explícita), pagos.

## Diseño técnico

### Definiciones de tools
```ts
{
  name: 'consultar_pedido',
  description: 'Devuelve estado y resumen de un pedido. Si no se da referencia, devuelve los últimos pedidos del cliente.',
  input_schema: {
    type: 'object',
    properties: {
      referencia: { type: 'string', nullable: true },
      limit: { type: 'integer', minimum: 1, maximum: 5, default: 3 },
    },
  },
}

{
  name: 'confirmar_pedido',
  description: 'Confirma un pedido en estado borrador. La referencia debe haberse creado en la misma conversación o explícitamente proporcionada por el cliente.',
  input_schema: {
    type: 'object',
    properties: { referencia: { type: 'string' } },
    required: ['referencia'],
  },
}

{
  name: 'cancelar_pedido',
  description: 'Cancela un pedido en estado borrador.',
  input_schema: {
    type: 'object',
    properties: { referencia: { type: 'string' } },
    required: ['referencia'],
  },
}
```

### Lógica de transición de estado
```sql
-- confirmar_pedido
UPDATE orders SET estado='confirmado', updated_at=NOW()
WHERE referencia=$1 AND estado='borrador'
RETURNING id, total, referencia;
-- Si rowCount=0: el pedido no existe o no está en borrador
```

### Reglas
- Sólo el dueño del `phone` puede confirmar/cancelar su pedido (filtro adicional `phone = $2`).
- Pedido ya `confirmado`: tool devuelve `{ok:false, motivo:'ya estaba confirmado'}` con info actual.
- Pedido `cancelado`: tool devuelve `{ok:false, motivo:'cancelado previamente'}`.
- Referencia ambigua (cliente dice "confirma mi pedido" sin referencia y hay > 1 borrador): el modelo debe pedir cuál.
- Si hay exactamente 1 borrador en la conversación, el modelo puede asumirlo y confirmarlo.

### System prompt v4 — adiciones
- "Si el cliente dice 'confirmo', 'sí', 'dale' u otra confirmación, llama `confirmar_pedido` con la referencia del último borrador creado en esta conversación."
- "Si hay varios borradores, pregunta cuál quiere confirmar antes de actuar."
- "Para 'mi pedido', 'el pedido que hice', usa `consultar_pedido` sin referencia para listar los últimos."
- "Tras confirmar, agradece y dice qué sigue (ej. 'lo procesaremos pronto')."

### Notificación de confirmación
- Al confirmar, log `info` estructurado con `{event:'order.confirmed', referencia, total, phone}`.
- En Slice 6 se conecta a webhook/Slack (esto es solo log por ahora).

## Archivos a crear / modificar
- `apps/backend/src/services/claude/tools.ts` (añadir 3 tools)
- `apps/backend/src/services/orders/confirm.ts`
- `apps/backend/src/services/orders/cancel.ts`
- `apps/backend/src/services/orders/find.ts`
- `apps/backend/src/prompts/system/sales.v4.ts`
- `apps/backend/src/prompts/index.ts`
- `apps/backend/tests/orders.confirm.test.ts`

## Criterios de aceptación
- [ ] "Confirmo" tras crear un borrador → `estado='confirmado'` en DB, agente acusa recibo.
- [ ] "Confirmo" sin pedido borrador previo → agente pide aclarar, no falla.
- [ ] "¿En qué va mi pedido?" sin referencia → lista últimos del phone.
- [ ] "¿En qué va el pedido ORD-XXXX?" → estado y total.
- [ ] "Cancela mi pedido" sobre borrador → `estado='cancelado'`.
- [ ] Confirmar un pedido ya confirmado → mensaje claro al cliente, no error técnico.
- [ ] Confirmar un pedido de otro `phone` → tratado como "no existe".
- [ ] Log estructurado `order.confirmed` se emite al confirmar.
- [ ] Si hay 2 borradores activos, el modelo pregunta cuál confirmar (no asume).
- [ ] Transiciones inválidas no rompen la base (UPDATE condicional).

## Tests requeridos
- **Unit:** matriz de transiciones de estado (borrador→confirmado, confirmado→confirmado, cancelado→confirmado, etc.).
- **Integración:** `orders.confirm` con conditional UPDATE; verifica idempotencia.
- **Integración:** `claude.respond` con mock simulando "confirmo" con 1 borrador previo.
- **E2E manual:** crear → confirmar → consultar.

## Demo
```
# Desde teléfono:
"Quiero 1 camisa azul"
# (agente confirma, crea borrador con referencia)
"Confirmo"
# Esperado: estado pasa a confirmado, agente acusa
"¿Cuál es el estado de mi pedido?"
# Esperado: lista el pedido confirmado con su referencia
```
```sql
SELECT referencia, estado, total FROM orders WHERE phone = '<phone>' ORDER BY created_at DESC;
```

## Riesgos del slice
- **Confirmación implícita ambigua:** "ok" puede ser confirmación o no; system prompt instruye a confirmar verbalmente antes de llamar el tool si hay duda.
- **Carrera entre dos turnos consecutivos rápidos:** UPDATE condicional `estado='borrador'` da idempotencia.
- **Cliente confirma pedido de otra conversación con referencia explícita:** filtro por `phone` lo bloquea.
- **Logs estructurados sin sink real:** aceptable; se conecta a Slack en Slice 6 o 8.
