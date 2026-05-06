# Slice 4 — Creación de pedido

## Objetivo
Permitir que el cliente solicite uno o más productos por chat y que el sistema cree un pedido en estado `borrador` con sus ítems y total. El agente devuelve un resumen con referencia legible.

## Dependencias
- Slice 3 completo (búsqueda de catálogo)
- Variables: ninguna nueva

## Alcance
**Incluye:**
- Tablas `orders` y `order_items`
- Tool `crear_pedido` con lista de ítems, cada item incluye `cotizacion_id` (snapshot de Slice 3) y `precio_unitario_esperado`
- Validación atómica dentro de la TX: producto existe, está activo, stock suficiente, **precio actual coincide con `precio_unitario_esperado`** (ventana cotización ↔ creación)
- Cálculo de total al momento del borrador, en moneda configurada (`CURRENCY`)
- Generación de referencia legible **`ORD-YYYYMMDD-XXXXXX`** (6 hex chars, día calculado en `BUSINESS_TZ`)
- System prompt v3: instruye al modelo a confirmar items y total con el cliente antes de crear el borrador, y a re-cotizar si el backend rechaza por precio cambiado

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
  description: 'Crea un pedido en estado borrador. Antes de llamar este tool DEBES haber confirmado con el cliente los productos exactos, cantidades y total. Cada item lleva el cotizacion_id devuelto por consultar_producto y el precio_unitario_esperado que se le mostró al cliente.',
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
            cotizacion_id: { type: 'string', description: 'Id devuelto por consultar_producto (Slice 3). Anclaje contra cambios de precio.' },
            precio_unitario_esperado: { type: 'number', description: 'Precio que se le mostró al cliente. Si difiere del precio actual, el backend rechaza.' },
          },
          required: ['sku', 'cantidad', 'cotizacion_id', 'precio_unitario_esperado'],
        },
      },
      notas: { type: 'string', description: 'Comentarios del cliente sobre el pedido', nullable: true },
    },
    required: ['items'],
  },
}
```

### Flujo del tool
Todo dentro de **una sola transacción** que abre con `pg_advisory_xact_lock(hashtext($phone))` (lock por conversación, §4.6).

1. Validar cada item:
   - Producto con ese `sku` existe y `activo=TRUE`.
   - `stock >= cantidad`.
   - `cotizacion_id` existe, no está expirado y corresponde al mismo `product_id` y `conversation_id`.
   - **`products.precio` actual === `precio_unitario_esperado`** (anclaje de precio). Si difiere → tool_result error con `motivo='precio_cambiado'`, `precio_actual`, y el modelo recotiza.
   - Si alguna validación falla: devolver `tool_result` con detalle y NO crear pedido. La TX hace rollback.
2. `precio_unitario := precio_unitario_esperado` (ya validado), `subtotal = precio_unitario * cantidad`.
3. Generar `referencia = ORD-YYYYMMDD-XXXXXX`:
   - `YYYYMMDD` calculado con `BUSINESS_TZ` (día comercial, no UTC).
   - `XXXXXX` = 6 hex chars (16⁶ = 16 777 216 valores/día; colisión ~50 % a ~5 100 pedidos/día).
   - Reintento ante colisión: hasta 5 intentos, luego abortar con error.
4. Insertar `orders` y `order_items` en la misma TX.
5. Commit. Devolver `tool_result` con `referencia`, `total`, `currency`, `items` resueltos.

### Tool result format (éxito)
```json
{
  "ok": true,
  "referencia": "ORD-20260506-7A3F1B",
  "estado": "borrador",
  "currency": "CLP",
  "total": 39980,
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
    {"sku":"XYZ-999","motivo":"no_encontrado"},
    {"sku":"ABC-123","motivo":"stock_insuficiente","stock_disponible":1,"cantidad_pedida":5},
    {"sku":"DEF-456","motivo":"precio_cambiado","precio_esperado":15000,"precio_actual":17500},
    {"sku":"GHI-789","motivo":"cotizacion_expirada"}
  ]
}
```
Ante `precio_cambiado` o `cotizacion_expirada`, el modelo debe recotizar (`consultar_producto`) y volver a confirmar con el cliente antes de reintentar `crear_pedido`.

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
- "Si el tool devuelve `precio_cambiado` o `cotizacion_expirada`, recotiza con `consultar_producto`, comunica el nuevo precio al cliente, y sólo reintenta `crear_pedido` si el cliente acepta."
- "Si el tool devuelve error de stock, ofrece la cantidad disponible o productos similares."
- "Cuando recibas el `cotizacion_id` y precio en `consultar_producto`, debes guardarlos mentalmente y pasarlos a `crear_pedido` exactamente como vinieron."

## Archivos a crear / modificar
- `apps/backend/src/services/claude/tools.ts` (añadir `crear_pedido`)
- `apps/backend/src/services/orders/create.ts` (TX con lock, validación de cotización + precio)
- `apps/backend/src/services/orders/reference.ts` (generador `ORD-YYYYMMDD-XXXXXX` con `BUSINESS_TZ`)
- `apps/backend/src/prompts/system/sales.v3.ts`
- `apps/backend/src/prompts/index.ts`
- `infra/migrations/<timestamp>_orders.sql`
- `apps/backend/tests/orders.create.test.ts` (incluye casos: precio_cambiado, cotizacion_expirada, colisión de referencia)

## Criterios de aceptación
- [ ] "Quiero 2 camisas azules" → el agente confirma items y total antes de crear.
- [ ] Tras confirmar, se crea fila en `orders` (`estado='borrador'`) y filas en `order_items`.
- [ ] La respuesta al cliente incluye referencia tipo `ORD-YYYYMMDD-XXXXXX` (6 hex chars).
- [ ] El día en la referencia corresponde a `BUSINESS_TZ` (test: simular reloj en 23:30 hora local; la referencia debe usar el día local).
- [ ] El total persistido coincide con la suma de `subtotal` de items y la `currency` del config.
- [ ] SKU inexistente → tool error, agente pide aclarar; NO se crea pedido.
- [ ] Stock insuficiente → tool error con cantidad disponible; agente ofrece ajustar.
- [ ] Producto inactivo → tratado como inexistente.
- [ ] **Precio cambiado entre cotización y creación:** admin edita precio en medio del flujo → tool error `precio_cambiado`; modelo recotiza y pregunta al cliente antes de reintentar.
- [ ] **`cotizacion_id` expirado o de otra conversación → error `cotizacion_expirada`.**
- [ ] Referencia es única (UNIQUE constraint atrapa colisiones; reintenta hasta 5 veces; test que fuerza colisión vía mock de RNG).
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
- **Race condition: dos turnos paralelos crean dos pedidos:** mitigado por `pg_advisory_xact_lock(hashtext(phone))` introducido en Slice 1, ahora reutilizado dentro de la TX de creación.
- **Precio se mueve entre cotización y creación:** mitigado con `cotizacion_id` + `precio_unitario_esperado`; si el admin edita precio en medio del flujo, el cliente nunca paga distinto a lo cotizado.
- **Stock no se reserva entre borrador y confirmación:** Slice 5 revalida stock al confirmar; aceptable en v1.
- **Colisión de referencia:** UNIQUE atrapa; reintento con nueva semilla hasta 5 veces. Test que cubra el reintento.
- **TZ del servidor distinta a `BUSINESS_TZ`:** mitigado calculando el día con `Intl.DateTimeFormat(BUSINESS_TZ)`; test cubre el caso 23:30 local cerca del cambio de día UTC.
