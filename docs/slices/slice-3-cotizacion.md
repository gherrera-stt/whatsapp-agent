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
- Loop de `tool_use` → `tool_result` hasta respuesta final del modelo (cap 8 iteraciones)
- Soporte de **bloques `tool_use` en paralelo** dentro de una sola respuesta (Claude puede emitir N en un turno; el runner los ejecuta concurrentemente y devuelve todos los `tool_result` en el siguiente mensaje)
- Soporte de **lookup batch** en el tool: el cliente puede preguntar por varios productos a la vez sin agotar iteraciones
- **Cache breakpoint movido a tools:** ahora el `cache_control: ephemeral` va al final del bloque `tools` (no en `system`), porque Anthropic cachea el prefijo `system + tools` y al introducir tools el cache previo se invalida
- System prompt actualizado a `system.sales.v2`: instruye usar el tool para CUALQUIER dato de catálogo
- Formato de respuesta consistente con precio + stock + descripción corta
- Snapshot de cotización con id corto que el modelo usa al crear pedido (Slice 4) para protegerse de cambios de precio entre cotizar y crear

**No incluye:** crear pedidos (Slice 4).

## Diseño técnico

### Definición del tool
```ts
{
  name: 'consultar_producto',
  description: 'Busca productos en el catálogo por nombre o SKU. Acepta uno o varios queries en una sola llamada para evitar iteraciones innecesarias.',
  input_schema: {
    type: 'object',
    properties: {
      queries: {
        type: 'array',
        minItems: 1,
        maxItems: 10,
        items: { type: 'string', description: 'Nombre o SKU. Búsqueda parcial case-insensitive.' },
        description: 'Lista de productos a buscar en una sola llamada. Si el cliente pregunta por A, B y C, manda los tres aquí en lugar de tres tool_use separados.',
      },
      limit_por_query: { type: 'integer', minimum: 1, maximum: 5, default: 3 },
    },
    required: ['queries'],
  },
}
```

> El tool acepta arrays para que "¿tienes A, B, C, D?" se resuelva en **una** iteración. El cap del loop sube a 8 (ver más abajo) para permitir aclaraciones, pero el flujo feliz nunca debería pasar de 1–2 iteraciones.

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
  "queries": [
    {
      "query": "camisa azul talla M",
      "results": [
        {
          "sku": "ABC-123",
          "nombre": "Camisa azul talla M",
          "descripcion": "Algodón 100%, manga corta",
          "precio": 19990,
          "currency": "CLP",
          "stock": 7,
          "disponible": true,
          "cotizacion_id": "QT-3F2A1B"
        }
      ],
      "match_count": 1
    }
  ]
}
```

- `currency` viene del config global (`CURRENCY`); el modelo formatea con esa moneda.
- `cotizacion_id` es un **snapshot del precio** que el backend retiene en una tabla `cotizaciones` (TTL 30 min). El modelo lo pasa al tool `crear_pedido` (Slice 4) junto con cada item; si el precio actual de DB difiere del snapshot, el backend rechaza la creación con un error claro y el modelo recotiza.
- Si `match_count = 0` el modelo debe decirle al cliente que no encontró el producto y ofrecer alternativas (en ningún caso inventa precios).

### Tabla de snapshots `cotizaciones`
```sql
CREATE TABLE cotizaciones (
  id              TEXT PRIMARY KEY,                    -- ej. QT-3F2A1B
  conversation_id BIGINT NOT NULL REFERENCES conversations(id),
  product_id      BIGINT NOT NULL REFERENCES products(id),
  precio          NUMERIC(12,2) NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at      TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '30 minutes'
);
CREATE INDEX idx_cotizaciones_conv ON cotizaciones (conversation_id, created_at DESC);
```
Sweep oportunista: en cada inserción, `DELETE FROM cotizaciones WHERE expires_at < NOW()` (barato; índice cubre la query).

### Loop de tool use
```
client → tool_use(consultar_producto, {queries: [A, B, C]})   # uno o varios bloques tool_use en paralelo
backend → ejecuta todos los lookups concurrentemente → tool_result (uno por id)
client → tool_use(otro_producto) o respuesta final
```
- Límite duro: **8 iteraciones** de tool use por turno (subido desde 3 para acomodar consultas que requieran aclaraciones encadenadas y porque ahora el tool acepta arrays). Si excede, cortar y responder fallback.
- En cada iteración la respuesta del modelo puede contener N bloques `tool_use` paralelos; el runner los ejecuta en paralelo (con el lock por conversación tomado a nivel del turno, no por iteración) y devuelve un único `messages: [{role:'user', content: [tool_result_1, tool_result_2, ...]}]`.
- Cada iteración (cada llamada al SDK) cuenta como una fila en `llm_traces` (Slice 8.5).

### System prompt v2 — adiciones
- "Para cualquier dato de precio, stock o disponibilidad, DEBES usar el tool `consultar_producto`. Nunca inventes precios."
- "Si el cliente menciona varios productos, llama el tool una vez por cada uno."
- "Si no hay resultados, ofrece alternativas en lugar de inventar."

### Errores y casos borde
- `queries` vacío: tool_result `{queries:[], error:"sin queries"}`.
- DB caída en mitad del turno: respuesta fallback al cliente, log `error`.
- Producto inactivo: no aparece en resultados (filtro `activo = TRUE`).
- Múltiples coincidencias: el modelo decide cómo desambiguar (preguntar al cliente o listar).
- `cotizacion_id` expirado al pasarlo a `crear_pedido` (Slice 4): backend devuelve error → modelo recotiza llamando al tool de nuevo.

### Nota sobre `pg_trgm` en deploy
La extensión se crea localmente sin fricción (Postgres en Docker corre como superuser). En servicios gestionados (RDS, Supabase, Neon) el usuario aplicación normalmente NO puede `CREATE EXTENSION`. Hay que ejecutarlo desde un rol privilegiado una sola vez como parte del provisioning. Documentar en el runbook del Slice 8 y, mientras tanto, el código tiene un fallback (`ORDER BY length(nombre)`) que funciona aunque la extensión falte.

## Archivos a crear / modificar
- `apps/backend/src/services/claude/tools.ts` (definiciones de tools)
- `apps/backend/src/services/claude/respond.ts` (loop tool_use con paralelo + cap 8)
- `apps/backend/src/services/catalog/search.ts`
- `apps/backend/src/services/catalog/cotizaciones.ts` (snapshot + lookup + sweep)
- `apps/backend/src/prompts/system/sales.v2.ts`
- `apps/backend/src/prompts/index.ts` (registrar v2)
- `infra/migrations/<timestamp>_pg_trgm.sql` (`CREATE EXTENSION IF NOT EXISTS pg_trgm`)
- `infra/migrations/<timestamp>_cotizaciones.sql`
- `apps/backend/tests/catalog.search.test.ts`
- `apps/backend/tests/claude.tool-loop.test.ts` (incluye caso con tool_use paralelos)

## Criterios de aceptación
- [ ] "¿cuánto cuesta la camisa azul?" devuelve precio real desde DB.
- [ ] "¿tienes stock de SKU ABC-123?" usa el SKU como lookup directo.
- [ ] Producto que no existe: el agente NO inventa precio; ofrece alternativas o pregunta.
- [ ] Producto `activo=FALSE` no aparece en resultados.
- [ ] **"¿tienes A, B, C, D?"** se resuelve en 1 sola iteración (queries array) y la respuesta menciona los cuatro.
- [ ] **Bloques `tool_use` en paralelo:** test con mock del SDK que devuelve 2 `tool_use` en una respuesta verifica que ambos se ejecutan concurrentemente y el siguiente mensaje al modelo lleva ambos `tool_result`.
- [ ] Loop de tool use limitado a **8 iteraciones**; al exceder, fallback al cliente con log `warn`.
- [ ] Cada `tool_use` exitoso inserta una fila en `cotizaciones` con `expires_at = NOW() + 30 min`.
- [ ] Sin `pg_trgm`: la búsqueda funciona aunque sea menos relevante.
- [ ] Latencia P95 < 6 s con 1 tool call (cumple §4.5 SLO de cotización).
- [ ] Con tools, segunda llamada en < 5 min reporta `cache_read_tokens > 0` (cache de prefijo `system + tools` activo).
- [ ] No se persiste `intent` en `messages` (la columna no existe; fuente de verdad = `llm_traces.tools_called` en Slice 8.5).

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
- **Loop infinito de tool_use:** mitigado con límite de 8 iteraciones + fallback.
- **Búsqueda fuzzy mala UX:** si los matches son malos, agregar `pg_trgm` y/o normalización de tildes.
- **Inyección vía nombre de producto:** sanitizar `query` (no permitir caracteres SQL extraños — se usa parametrizado, pero no asumir).
- **Cache invalidado al cambiar tools:** cualquier modificación al esquema de los tools rompe el cache. Versionar el bundle de tools junto con el system prompt y registrar el cambio en el changelog del prompt.
- **`pg_trgm` no instalable en deploy gestionado:** fallback `ORDER BY length(nombre)` mantiene la funcionalidad. Documentar en runbook.
