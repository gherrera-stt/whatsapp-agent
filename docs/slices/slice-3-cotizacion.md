# Slice 3 — Cotización (consulta de catálogo)

## Objetivo
Permitir que el cliente pregunte por precio y disponibilidad de productos del catálogo, garantizando que los datos vienen de la base — no de alucinaciones del modelo. Se logra exponiendo a Claude el tool `consultar_producto`.

## Dependencias
- Slice 2 completo
- Catálogo cargado (Slice 0 seed)
- Variables: ninguna nueva

## Alcance
**Incluye:**
- Tool `consultar_producto` declarado en la llamada a Claude
- Búsqueda en `products` por nombre (fuzzy/ILIKE) o SKU exacto
- Loop de `tool_use` → `tool_result` hasta respuesta final del modelo
- System prompt actualizado a `system.sales.v2`: instruye usar el tool para CUALQUIER dato de catálogo
- Formato de respuesta consistente con precio + stock + descripción corta
- Persistencia del `intent='cotizacion'` en `messages` cuando aplica

**No incluye:** crear pedidos (Slice 4), cotizaciones multi-producto en una sola llamada al tool (lo hacemos con múltiples invocaciones del mismo tool).

## Diseño técnico

### Definición del tool
```ts
{
  name: 'consultar_producto',
  description: 'Busca un producto en el catálogo por nombre o SKU y devuelve precio y stock. Devuelve hasta 5 coincidencias si el nombre es ambiguo.',
  input_schema: {
    type: 'object',
    properties: {
      query: {
        type: 'string',
        description: 'Nombre del producto o SKU. Búsqueda parcial case-insensitive.',
      },
      limit: { type: 'integer', minimum: 1, maximum: 5, default: 3 },
    },
    required: ['query'],
  },
}
```

### Búsqueda en `products`
```sql
SELECT id, sku, nombre, descripcion, precio, stock, activo
FROM products
WHERE activo = TRUE
  AND (
    sku ILIKE $1                                  -- match exacto SKU
    OR LOWER(nombre) LIKE LOWER('%' || $2 || '%') -- match parcial nombre
  )
ORDER BY
  CASE WHEN sku ILIKE $1 THEN 0 ELSE 1 END,
  similarity(LOWER(nombre), LOWER($2)) DESC NULLS LAST  -- requiere pg_trgm
LIMIT $3;
```
Si `pg_trgm` no está instalada, fallback a `ORDER BY length(nombre)` para favorecer matches más cortos.

### Tool result format
```json
{
  "results": [
    {
      "sku": "ABC-123",
      "nombre": "Camisa azul talla M",
      "descripcion": "Algodón 100%, manga corta",
      "precio_clp": 19990,
      "stock": 7,
      "disponible": true
    }
  ],
  "match_count": 1
}
```
Si `match_count = 0` el modelo debe decirle al cliente que no encontró el producto y ofrecer alternativas.

### Loop de tool use
```
client → tool_use(consultar_producto, {query})
backend → query DB → tool_result
client → tool_use(otro_producto) o respuesta final
```
- Límite duro: **3 iteraciones** de tool use por turno. Si excede, cortar y responder fallback.
- Cada iteración cuenta para `llm_traces` (Slice 8.5).

### System prompt v2 — adiciones
- "Para cualquier dato de precio, stock o disponibilidad, DEBES usar el tool `consultar_producto`. Nunca inventes precios."
- "Si el cliente menciona varios productos, llama el tool una vez por cada uno."
- "Si no hay resultados, ofrece alternativas en lugar de inventar."

### Errores y casos borde
- `query` vacía: tool_result `{results:[], match_count:0, error:"query vacía"}`.
- DB caída en mitad del turno: respuesta fallback al cliente, log `error`.
- Producto inactivo: no aparece en resultados (filtro `activo = TRUE`).
- Múltiples coincidencias: el modelo decide cómo desambiguar (preguntar al cliente o listar).

## Archivos a crear / modificar
- `apps/backend/src/services/claude/tools.ts` (definiciones de tools)
- `apps/backend/src/services/claude/respond.ts` (loop tool_use)
- `apps/backend/src/services/catalog/search.ts`
- `apps/backend/src/prompts/system/sales.v2.ts`
- `apps/backend/src/prompts/index.ts` (registrar v2)
- `infra/migrations/0003_pg_trgm.sql` (`CREATE EXTENSION IF NOT EXISTS pg_trgm`)
- `apps/backend/tests/catalog.search.test.ts`
- `apps/backend/tests/claude.tool-loop.test.ts`

## Criterios de aceptación
- [ ] "¿cuánto cuesta la camisa azul?" devuelve precio real desde DB.
- [ ] "¿tienes stock de SKU ABC-123?" usa el SKU como lookup directo.
- [ ] Producto que no existe: el agente NO inventa precio; ofrece alternativas o pregunta.
- [ ] Producto `activo=FALSE` no aparece en resultados.
- [ ] Pregunta sobre 2 productos en un mismo mensaje invoca el tool 2 veces y la respuesta menciona ambos precios.
- [ ] El `intent='cotizacion'` queda registrado en `messages` saliente (heurística por presencia de tool_use).
- [ ] Loop de tool use limitado a 3 iteraciones; al exceder, fallback al cliente.
- [ ] Sin `pg_trgm`: la búsqueda funciona aunque sea menos relevante.
- [ ] Latencia P95 < 5 s con 1 tool call (más alta que Slice 2 por la ronda extra).

## Tests requeridos
- **Unit:** `catalog.search` con varios queries (exacto SKU, parcial, sin match, vacío).
- **Integración:** `claude.respond` con mock del SDK simulando `tool_use → tool_result → final`. Verifica que el tool fue invocado con los argumentos esperados.
- **E2E manual:** preguntar precios reales desde teléfono y comparar con `psql`.

## Demo
```sql
-- En psql:
INSERT INTO products (sku, nombre, descripcion, precio, stock)
VALUES ('TEST-001', 'Camisa azul talla M', 'Algodón', 19990, 7);
```
```
# Desde teléfono:
"¿cuánto cuesta la camisa azul talla M?"
# Esperado: respuesta con $19.990 y stock 7.

"¿tienes la camisa azul y la camisa roja?"
# Esperado: confirma azul, dice que la roja no la tiene.
```

## Riesgos del slice
- **Modelo inventa precios pese al system prompt:** mitigado con instrucción explícita + test manual del golden set.
- **Loop infinito de tool_use:** mitigado con límite de 3 iteraciones.
- **Búsqueda fuzzy mala UX:** si los matches son malos, agregar `pg_trgm` y/o normalización de tildes.
- **Inyección vía nombre de producto:** sanitizar `query` (no permitir caracteres SQL extraños — se usa parametrizado, pero no asumir).
