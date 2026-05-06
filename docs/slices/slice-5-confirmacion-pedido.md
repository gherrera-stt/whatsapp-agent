# Slice 5 — Confirmación y consulta de pedido

> **Gates ASR.** `pre-slice-5.md` y `post-slice-5.md`. Casos mínimos: `docs/adversarial-review.md` §10.6. Invariantes a verificar: I-8 (revalidación stock+precio en TX), I-3 (lock).

## Objetivo
Permitir que el cliente confirme un pedido en estado `borrador` (transición a `confirmado`) y consulte el estado de pedidos previos por referencia o por su número de teléfono.

## Dependencias
- Slice 4 completo
- Variables: ninguna nueva

## Alcance
**Incluye:**
- Tool `consultar_pedido` (por referencia o por phone → últimos N)
- Tool `confirmar_pedido` con `referencia` opcional: si llega vacía, el backend resuelve "único borrador del phone" determinísticamente
- Tool `cancelar_pedido` (transición `borrador → cancelado`; sólo en estado borrador)
- **Revalidación de stock al confirmar** dentro de la TX, con lock por conversación
- System prompt v4: maneja confirmación y consulta; deja al backend resolver "el último borrador" en lugar de exigir que el modelo recuerde la referencia
- Notificación interna (log estructurado) cuando un pedido pasa a `confirmado`

**No incluye:** modificación de items de un pedido existente (cancelar y recrear), descuento real de stock al confirmar (deuda explícita: el stock se valida pero no se decrementa hasta despacho), pagos.

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
  description: 'Confirma un pedido en estado borrador. Si no entregas referencia, el backend confirma el único borrador abierto del cliente. Si hay más de un borrador y no entregas referencia, el tool devuelve error y debes preguntar al cliente cuál.',
  input_schema: {
    type: 'object',
    properties: {
      referencia: { type: 'string', description: 'Opcional. Si se omite, se resuelve por backend.', nullable: true },
    },
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
Todo dentro de **una TX con `pg_advisory_xact_lock(hashtext(phone))`**.

```
1. Resolver referencia:
   - Si llega del modelo → usarla.
   - Si no llega:
       SELECT referencia FROM orders
       WHERE phone=$1 AND estado='borrador'
       ORDER BY created_at DESC LIMIT 2;
     - 0 filas → tool_result {ok:false, motivo:'no_hay_borradores'}.
     - 1 fila → usar esa referencia.
     - 2 filas → tool_result {ok:false, motivo:'multiples_borradores', referencias:[...]} → modelo pregunta cuál.

2. Revalidar stock dentro de la TX:
     SELECT oi.product_id, oi.cantidad, p.stock, p.activo, p.precio AS precio_actual,
            oi.precio_unitario AS precio_pedido
     FROM order_items oi
     JOIN products p ON p.id = oi.product_id
     WHERE oi.order_id = (SELECT id FROM orders WHERE referencia=$ref AND phone=$phone);
   - Si algún producto tiene stock<cantidad o activo=FALSE → tool_result {ok:false, motivo:'stock_insuficiente'|'producto_inactivo', detalle:[...]}; rollback.
   - Si algún producto tiene precio_actual ≠ precio_pedido → tool_result {ok:false, motivo:'precio_cambiado', detalle:[...]}; rollback. El modelo informa al cliente y eventualmente recrea el pedido.

3. UPDATE orders SET estado='confirmado', updated_at=NOW()
   WHERE referencia=$ref AND phone=$phone AND estado='borrador'
   RETURNING id, total, referencia;
   - Si rowCount=0 (ya confirmado o cancelado en paralelo): tool_result con info actual del pedido.

4. Commit. Devolver {ok:true, referencia, total, currency, estado:'confirmado'}.
```

> **Nota sobre overselling:** la revalidación reduce la ventana, pero **stock no se decrementa**, por lo que dos confirmaciones simultáneas para el mismo último ítem aún pueden pasar ambas. El lock por conversación serializa por cliente, no entre clientes. Esto se acepta en v1 como deuda; mitigarlo requiere reservas de stock con TTL, fuera de alcance.

### Reglas
- Sólo el dueño del `phone` puede confirmar/cancelar su pedido (filtro adicional `phone = $2`).
- Pedido ya `confirmado`: tool devuelve `{ok:false, motivo:'ya estaba confirmado'}` con info actual.
- Pedido `cancelado`: tool devuelve `{ok:false, motivo:'cancelado previamente'}`.
- Referencia ambigua (cliente dice "confirma mi pedido" sin referencia y hay > 1 borrador): el modelo debe pedir cuál.
- Si hay exactamente 1 borrador en la conversación, el modelo puede asumirlo y confirmarlo.

### System prompt v4 — adiciones
- "Si el cliente dice 'confirmo', 'sí', 'dale' u otra confirmación clara tras un pedido recién creado, llama `confirmar_pedido` SIN referencia: el backend resuelve el borrador correcto."
- "Si el tool responde `multiples_borradores`, pregunta al cliente cuál quiere confirmar (lista las referencias) y reintenta con `referencia` explícita."
- "Si el tool responde `stock_insuficiente` o `producto_inactivo`, comunica el detalle al cliente y ofrece alternativas; no insistas con el mismo pedido."
- "Si el tool responde `precio_cambiado`, comunica el nuevo precio y pregunta si quiere actualizar el pedido (cancelar + recrear)."
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
- [ ] "Confirmo" tras crear UN borrador → `estado='confirmado'` en DB, agente acusa recibo. El modelo NO necesita pasar `referencia`; el backend la resuelve.
- [ ] "Confirmo" sin pedido borrador previo → tool devuelve `no_hay_borradores`; agente pide aclarar.
- [ ] "¿En qué va mi pedido?" sin referencia → lista últimos del phone.
- [ ] "¿En qué va el pedido ORD-XXXXXX?" → estado y total.
- [ ] "Cancela mi pedido" sobre borrador → `estado='cancelado'`.
- [ ] Confirmar un pedido ya confirmado → mensaje claro al cliente, no error técnico.
- [ ] Confirmar un pedido de otro `phone` → tratado como "no existe".
- [ ] Log estructurado `order.confirmed` se emite al confirmar.
- [ ] **Si hay 2 borradores activos** y el modelo llama sin referencia → tool devuelve `multiples_borradores` con la lista; el modelo pregunta cuál.
- [ ] **Stock cae a 0 entre borrador y confirmación** → tool devuelve `stock_insuficiente`; el pedido NO se confirma; agente comunica al cliente.
- [ ] **Precio se modifica entre borrador y confirmación** → tool devuelve `precio_cambiado`; agente comunica al cliente.
- [ ] Transiciones inválidas no rompen la base (UPDATE condicional).
- [ ] Lock por conversación: dos turnos consecutivos rápidos del mismo phone se serializan (no doble confirmación).

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
- **Carrera entre dos turnos consecutivos rápidos:** lock por conversación + UPDATE condicional `estado='borrador'` dan idempotencia y serialización.
- **Cliente confirma pedido de otra conversación con referencia explícita:** filtro por `phone` lo bloquea.
- **Overselling entre clientes distintos:** mitigado pero no resuelto; ver nota en lógica. Aceptado como deuda v1.
- **Modelo "olvida" la referencia del borrador recién creado:** mitigado al permitir `referencia` opcional y resolverla en backend.
- **Logs estructurados sin sink real:** aceptable; se conecta a Slack en Slice 6 o 8.
