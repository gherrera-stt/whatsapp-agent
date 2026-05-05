# Slice 4 — Creación de pedido

## Objetivo
Permitir que el cliente solicite uno o más productos por chat y que el sistema cree un pedido en estado `borrador` con sus ítems y total. El agente devuelve un resumen con referencia legible.

## Dependencias
- Slice 3 completo (búsqueda de catálogo)
- Variables: ninguna nueva

## Alcance
**Incluye:**
- Tablas `orders` y `order_items`
- Tool `crear_pedido` con lista de ítems
- Validación: producto existe, está activo, stock suficiente
- Cálculo de total al momento del borrador
- Generación de referencia legible (`ORD-YYYYMMDD-XXXX`)
- System prompt v3: instruye al modelo a confirmar items y total con el cliente antes de crear el borrador
- Persistencia del `intent='pedido'` en `messages`

**No incluye:** confirmación del pedido (Slice 5), descuento de stock (sólo se valida disponibilidad; no se reserva).

## Diseño técnico

### Modelo de datos (delta)
```sql
CREATE TABLE orders (
  id              BIGSERIAL PRIMARY KEY,
  conversation_id BIGINT NOT NULL REFERENCES conversations(id),
  phone           TEXT NOT NULL,
  referencia      TEXT NOT NULL UNIQUE,
  estado          TEXT NOT NULL DEFAULT 'borrador',
  -- estados: borrador | confirmado | cancelado
  total           NUMERIC(12,2) NOT NULL CHECK (total >= 0),
  notas           TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_orders_phone_estado ON orders (phone, estado);
CREATE INDEX idx_orders_conv ON orders (conversation_id);

CREATE TABLE order_items (
  id              BIGSERIAL PRIMARY KEY,
  order_id        BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id      BIGINT NOT NULL REFERENCES products(id),
  cantidad        INT NOT NULL CHECK (cantidad > 0),
  precio_unitario NUMERIC(12,2) NOT NULL CHECK (precio_unitario >= 0),
  subtotal        NUMERIC(12,2) NOT NULL CHECK (subtotal >= 0)
);
CREATE INDEX idx_order_items_order ON order_items (order_id);
```

### Definición del tool
```ts
{
  name: 'crear_pedido',
  description: 'Crea un pedido en estado borrador. Antes de llamar este tool DEBES haber confirmado con el cliente los productos exactos, cantidades y total.',
  input_schema: {
    type: 'object',
    properties: {
      items: {
        type: 'array',
        minItems: 1,
        items: {
          type: 'object',
          properties: {
            sku: { type: 'string' },
            cantidad: { type: 'integer', minimum: 1 },
          },
          required: ['sku', 'cantidad'],
        },
      },
      notas: { type: 'string', description: 'Comentarios del cliente sobre el pedido', nullable: true },
    },
    required: ['items'],
  },
}
```

### Flujo del tool
1. Validar cada `sku`: existe, `activo=TRUE`, `stock >= cantidad`.
   - Si alguno falla: devolver `tool_result` con `error` y detalle por SKU; NO crear pedido.
2. Calcular `precio_unitario` desde `products.precio` actual y `subtotal = precio * cantidad`.
3. Generar `referencia = ORD-YYYYMMDD-XXXX` donde XXXX = hex aleatorio 4 chars.
4. Insertar `orders` y `order_items` en una transacción.
5. Devolver `tool_result` con `referencia`, `total`, `items` resueltos.

### Tool result format (éxito)
```json
{
  "ok": true,
  "referencia": "ORD-20260505-7A3F",
  "estado": "borrador",
  "total_clp": 39980,
  "items": [
    {"sku":"ABC-123","nombre":"Camisa azul","cantidad":2,"precio_unitario":19990,"subtotal":39980}
  ]
}
```

### Tool result format (error de validación)
```json
{
  "ok": false,
  "errores": [
    {"sku":"XYZ-999","motivo":"no encontrado"},
    {"sku":"ABC-123","motivo":"stock insuficiente","stock_disponible":1,"cantidad_pedida":5}
  ]
}
```

### Mensaje que el agente envía al cliente
Después del tool exitoso, el modelo redacta algo como:
> "Listo, dejé tu pedido como borrador. Referencia ORD-20260505-7A3F.
> • 2 × Camisa azul talla M — $39.980
> Total: $39.980
> ¿Confirmas para procesarlo?"

### System prompt v3 — adiciones
- "Antes de llamar `crear_pedido`, repite al cliente los productos exactos con cantidades y total, y espera su confirmación."
- "Si el cliente menciona un producto sin cantidad, asume 1 pero confírmalo."
- "Después de crear el borrador, indícale la referencia y pídele que confirme con 'confirmo' o 'sí'."
- "Si el tool devuelve error de stock, ofrece la cantidad disponible o productos similares."

## Archivos a crear / modificar
- `apps/backend/src/services/claude/tools.ts` (añadir `crear_pedido`)
- `apps/backend/src/services/orders/create.ts`
- `apps/backend/src/services/orders/reference.ts` (generador de referencia)
- `apps/backend/src/prompts/system/sales.v3.ts`
- `apps/backend/src/prompts/index.ts`
- `infra/migrations/0004_orders.sql`
- `apps/backend/tests/orders.create.test.ts`

## Criterios de aceptación
- [ ] "Quiero 2 camisas azules" → el agente confirma items y total antes de crear.
- [ ] Tras confirmar, se crea fila en `orders` (`estado='borrador'`) y filas en `order_items`.
- [ ] La respuesta al cliente incluye referencia tipo `ORD-YYYYMMDD-XXXX`.
- [ ] El total persistido coincide con la suma de `subtotal` de items.
- [ ] SKU inexistente → tool error, agente pide aclarar; NO se crea pedido.
- [ ] Stock insuficiente → tool error con cantidad disponible; agente ofrece ajustar.
- [ ] Producto inactivo → tratado como inexistente.
- [ ] Referencia es única (UNIQUE constraint atrapa colisiones; reintenta generación).
- [ ] Inserts orders + items son transaccionales (si falla items, no queda order huérfano).
- [ ] Pedidos múltiples por la misma conversación coexisten en estado `borrador`.

## Tests requeridos
- **Unit:** `generateReference` (formato, randomness). `validateItems` (varios casos: ok, no existe, stock insuficiente, inactivo).
- **Integración:** `orders.create` contra Postgres efímero — verificar transaccionalidad forzando un fallo en items.
- **Integración:** `claude.respond` con mock del SDK simulando flow `consultar_producto → crear_pedido`.
- **E2E manual:** flujo completo desde teléfono con dos productos distintos.

## Demo
```
# Desde teléfono:
"Quiero 2 camisas azules talla M y 1 camisa roja"
# Esperado: agente confirma items y total, pregunta "¿confirmas?"
"sí"
# Esperado: agente devuelve referencia ORD-... con total
```
```sql
SELECT o.referencia, o.estado, o.total, oi.cantidad, p.nombre
FROM orders o JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.phone = '<phone>' ORDER BY o.created_at DESC;
```

## Riesgos del slice
- **Modelo crea pedido sin confirmar con el cliente:** mitigado en system prompt + verificable con eval (Slice 8.5).
- **Race condition: dos turnos paralelos crean dos pedidos:** improbable en WhatsApp 1-a-1, pero documentar.
- **Precio se mueve entre cotización y creación:** el `precio_unitario` se congela al crear; es el contrato comercial.
- **Stock no se reserva:** dos clientes pueden crear pedidos del mismo último ítem; aceptable en v1, documentar como deuda.
- **Colisión de referencia:** UNIQUE atrapa; reintento con nueva semilla. Test que cubra el reintento.
