# Slice 8.5 — Endurecimiento LLM

## Objetivo
Implementar la disciplina LLM Operations descrita en §4.9 de `solucion.md`: trazabilidad completa de cada llamada a Claude, control de costo, prompt versioning con runtime tracking, y golden set de evaluación. Con esto el sistema deja de ser una caja negra cuando algo falla.

## Dependencias
- Slice 8 completo (estructura de evals existe)
- Slice 7 completo (admin existe para vistas de traces)
- Variables nuevas: `LLM_COST_DAILY_USD_THRESHOLD` (default `20`), `CONVERSATION_TOKEN_BUDGET` (default `50000`), `EVAL_JUDGE_MODEL` (default `claude-haiku-4-5`)

## Alcance
**Incluye:**
- Tabla `llm_traces` y persistencia por turno
- Wrapper único `claude.callTracked` que reemplaza llamadas crudas al SDK
- Constantes de tarifa de `claude-sonnet-4-6` y cálculo de `cost_usd` por llamada
- Estructura `apps/backend/src/prompts/` con identificadores y versiones explícitos en runtime
- Vistas en admin: drill-down de conversación a sus traces; agregados de costo (día / conversación / top 10 caras)
- Guardrail por conversación: superar `CONVERSATION_TOKEN_BUDGET` dispara `escalar_a_humano` automáticamente
- Alerta diaria: log `warn` (más adelante: webhook) si gasto del día supera `LLM_COST_DAILY_USD_THRESHOLD`
- Golden set ampliado a 30 casos con LLM-as-judge (`claude-haiku-4-5`)
- Runner `npm run eval` mejorado: corre golden set, compara contra baseline, produce HTML

**No incluye:** integración con Langfuse/Helicone (deuda explícita; migrar cuando volumen > 5k traces/día), webhook Slack para alertas (deuda futura).

## Diseño técnico

### Modelo de datos (delta)
```sql
CREATE TABLE llm_traces (
  id                       BIGSERIAL PRIMARY KEY,
  conversation_id          BIGINT REFERENCES conversations(id),
  message_id               BIGINT REFERENCES messages(id),
  call_index               INT NOT NULL,                  -- ordinal de la llamada al SDK dentro del turno (no del turno)
  prompt_id                TEXT NOT NULL,
  prompt_version           INT NOT NULL,
  model                    TEXT NOT NULL,
  input_tokens             INT NOT NULL,
  output_tokens            INT NOT NULL,
  cache_read_tokens        INT NOT NULL DEFAULT 0,
  cache_creation_tokens    INT NOT NULL DEFAULT 0,
  latency_ms               INT NOT NULL,
  tools_called             JSONB NOT NULL DEFAULT '[]'::jsonb,
  input                    JSONB NOT NULL,
  output                   JSONB NOT NULL,
  cost_usd                 NUMERIC(10,6) NOT NULL,
  outcome                  TEXT NOT NULL CHECK (outcome IN ('ok','error','timeout')),
  error_detail             TEXT,
  created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_llm_traces_conv ON llm_traces (conversation_id, created_at);
CREATE INDEX idx_llm_traces_created ON llm_traces (created_at);
CREATE INDEX idx_llm_traces_prompt ON llm_traces (prompt_id, prompt_version);

-- Tabla 1-fila para deduplicación de la alerta diaria de costo (sobrevive restarts)
CREATE TABLE llm_cost_alert_state (
  id            INT PRIMARY KEY DEFAULT 1 CHECK (id = 1),
  last_alert_on DATE
);
INSERT INTO llm_cost_alert_state (id, last_alert_on) VALUES (1, NULL) ON CONFLICT DO NOTHING;
```

> **Naming `call_index` (antes `turn_index`).** Cada llamada al SDK genera una fila en `llm_traces`. Un único turno conversacional con tool use puede generar 2–4 filas (call_index 0..N). El "turno" del cliente se identifica por `message_id`. Renombrar evita confusión.

### Constantes de tarifa (`apps/backend/src/services/claude/pricing.ts`)
```ts
export const PRICING = {
  'claude-sonnet-4-6': {
    inputPerMTok: 3.00,
    outputPerMTok: 15.00,
    cacheReadPerMTok: 0.30,
    cacheCreationPerMTok: 3.75,
  },
  'claude-haiku-4-5': {
    inputPerMTok: 1.00,
    outputPerMTok: 5.00,
    cacheReadPerMTok: 0.10,
    cacheCreationPerMTok: 1.25,
  },
} as const;

export function costUsd(model: keyof typeof PRICING, usage): number { /* ... */ }
```
**Verificar las tarifas vigentes en docs.anthropic.com al implementar; las de arriba son indicativas.**

### Wrapper `claude.callTracked`
Reemplaza llamadas directas al SDK. Recibe:
```ts
{
  conversationId, messageId, callIndex,
  promptId, promptVersion,
  model, system, messages, tools, max_tokens, temperature,
}
```
Hace:
1. `t0 = performance.now()`.
2. `client.messages.create(...)`.
3. Calcula `latency_ms`, `cost_usd`, extrae `tools_called` del bloque de respuesta (puede haber múltiples bloques `tool_use` paralelos — todos se serializan).
4. INSERT en `llm_traces` con `outcome='ok'`.
5. Si error/timeout: INSERT con `outcome='error'|'timeout'` y `error_detail`.

El `respond()` mantiene un contador local `callIndex` que arranca en 0 y aumenta en cada iteración del tool loop; se persiste en `llm_traces.call_index`.

### Prompts versionados — registry
```ts
// apps/backend/src/prompts/index.ts
import * as salesV1 from './system/sales.v1';
import * as salesV2 from './system/sales.v2';
// ...
import * as salesV5 from './system/sales.v5';

export const REGISTRY = {
  'system.sales.v1': salesV1,
  // ...
  'system.sales.v5': salesV5,
} as const;

export const ACTIVE = 'system.sales.v5';
```
El wrapper toma el `promptId` y `promptVersion` desde el registry activo (no de strings sueltas).

> **Por qué se mantienen versiones inactivas en `REGISTRY`.** Sólo `ACTIVE` se usa para llamadas nuevas al modelo, pero `llm_traces` referencia versiones históricas vía `prompt_id + prompt_version`. Mantenerlas en el registry permite re-cargar el prompt exacto que se usó para reproducir traces antiguos en el admin (drill-down) o para correr eval comparativa entre versiones. Si una versión deja de servirse hace > 6 meses y no aparece en el `llm_traces` reciente, se puede archivar.

### Guardrail de tokens por conversación
Antes de llamar a Claude:
```sql
SELECT COALESCE(SUM(
  -- excluimos cache_read_tokens porque cuestan 10× menos y no queremos
  -- castigar a una conversación larga que se beneficia bien del cache
  input_tokens - cache_read_tokens + output_tokens
), 0) AS total
FROM llm_traces WHERE conversation_id = $1
  AND created_at > NOW() - INTERVAL '24 hours';
```
Si `total > CONVERSATION_TOKEN_BUDGET`:
- No llama a Claude.
- **Llama a `escalateToHuman({source:'guardrail', motivo:'fuera_de_scope', detalle:'excedió presupuesto de tokens'})`** — la función pura compartida con el tool (Slice 6).
- Envía mensaje "Esta conversación se ha vuelto compleja, te conectaré con una persona".

> **Por qué excluir `cache_read_tokens`:** los tokens leídos del cache cuestan 10× menos que los normales. Si los contáramos al peso, una conversación que aprovecha bien el cache se vería penalizada injustamente vs una corta sin cache. Si en producción aparece abuso (cliente intenta agotar contexto rápido), se puede pasar a una métrica ponderada por costo en lugar de tokens.

### Alerta diaria de costo
- Job in-process al recibir mensaje (lazy): si `last_alert_on` < `CURRENT_DATE`, calcular gasto del día.
- Si supera umbral: log `warn` `{event:'llm_cost.threshold_exceeded', day, total_usd, threshold}` y `UPDATE llm_cost_alert_state SET last_alert_on = CURRENT_DATE`.
- **Persistencia en tabla 1-fila (`llm_cost_alert_state`)** en lugar de variable en memoria: sobrevive a restarts del proceso, evitando re-disparar la alerta el mismo día tras un deploy.
- Cálculo: `SELECT SUM(cost_usd) FROM llm_traces WHERE created_at::date = (NOW() AT TIME ZONE current_setting('app.business_tz'))::date` — con `BUSINESS_TZ` para que "el día" coincida con el día comercial del cliente.

### Vistas en admin
Páginas/secciones nuevas:
- `/conversations/:id` — añadir tab "LLM Traces" con tabla por turno: `prompt_id@version`, tokens (in/out/cache), latencia, costo, tools llamados, link a expandir input/output completos.
- `/llm/cost` — gráficos:
  - Costo por día (últimos 30)
  - Top 10 conversaciones más caras (últimos 7 días)
  - Top prompts por uso
- Endpoints: `GET /api/admin/llm/traces?conversation_id=&limit=`, `GET /api/admin/llm/cost/daily`, `GET /api/admin/llm/cost/top-conversations`.

### Eval pipeline ampliado
- `evals/cases.yaml` con 30 casos: saludo, cotización exacta, cotización ambigua, cotización producto inexistente, pedido simple, pedido multi-producto, confirmación, cancelación, escalado por solicitud, escalado por frustración, intento de fraude, fuera de scope, mensaje vacío, palabra ofensiva, instrucción de "ignora tu prompt", etc.
- Cada caso:
  ```yaml
  - id: cotizacion_ambigua
    contexto:
      - {role: user, content: "hola"}
      - {role: assistant, content: "..."}
    input: "¿cuánto vale la camisa?"
    aserciones:
      tool_esperado: consultar_producto
      regex_respuesta: "(precio|cuesta|cuánto)"
      judge: "La respuesta pide aclarar qué camisa porque hay varias."
  ```
- LLM-as-judge usa `claude-haiku-4-5` con prompt fijo que devuelve `{pass: bool, reason: string}`.
- Reporte HTML: `evals/reports/<timestamp>.html` con tabla pass/fail, latencia/costo agregados, diff vs baseline.
- Baseline en `evals/baseline.json` (commiteado); `npm run eval -- --update-baseline` lo refresca.

## Archivos a crear / modificar
- `infra/migrations/<timestamp>_llm_traces.sql` (incluye `llm_cost_alert_state`)
- `apps/backend/src/services/claude/pricing.ts`
- `apps/backend/src/services/claude/callTracked.ts`
- `apps/backend/src/services/claude/respond.ts` (refactor: usa `callTracked`, mantiene `callIndex`)
- `apps/backend/src/services/llm/traces.ts` (insert + queries agregadas)
- `apps/backend/src/services/llm/budgetGuardrail.ts` (llama a `escalateToHuman` compartida)
- `apps/backend/src/services/llm/costAlert.ts` (lee/escribe `llm_cost_alert_state`)
- `apps/backend/src/prompts/index.ts` (registry tipado)
- `apps/backend/src/routes/admin/llm.ts`
- `apps/admin/src/pages/LLMCost.tsx`
- `apps/admin/src/pages/ConversationDetail.tsx` (añadir tab Traces)
- `apps/backend/scripts/eval.ts` (rewrite con judge + baseline)
- `evals/cases.yaml` (30 casos)
- `evals/baseline.json`
- `apps/backend/tests/claude.callTracked.test.ts`
- `apps/backend/tests/budgetGuardrail.test.ts`
- `apps/backend/tests/llm.costAlert.persistence.test.ts` (verifica que tras restart no se re-dispara el mismo día)

## Criterios de aceptación
- [ ] Cada llamada a Claude inserta una fila en `llm_traces` con tokens, latencia y costo correctos. `call_index` ordinal correcto dentro del turno.
- [ ] `cost_usd` calculado coincide con tarifas vigentes (verificar con caso conocido).
- [ ] Una conversación de 3 turnos genera ≥ 3 filas en `llm_traces` (más si hubo tool use; las llamadas paralelas con N tool_use se serializan en filas distintas con `call_index` consecutivo).
- [ ] El admin muestra los traces de una conversación con drill-down a input/output completos.
- [ ] La página `/llm/cost` muestra costo del día actual y top 10 conversaciones más caras.
- [ ] **Conversación que excede `CONVERSATION_TOKEN_BUDGET` (excluyendo `cache_read_tokens`) se escala automáticamente a humano sin llamar a Claude, vía la función pura `escalateToHuman` con `source='guardrail'`.**
- [ ] Cuando el gasto diario supera umbral, aparece log `warn` con `event:llm_cost.threshold_exceeded`.
- [ ] **Restart del proceso el mismo día NO re-dispara la alerta** (test con `llm_cost_alert_state`).
- [ ] `prompt_id` y `prompt_version` registrados en cada trace coinciden con el registry activo. Versiones inactivas siguen en `REGISTRY` para replay.
- [ ] `npm run eval` corre los 30 casos del golden set, invoca al judge en los que aplica y produce reporte HTML.
- [ ] `npm run eval` falla (exit 1) si baseline regresiona en > 2 casos.
- [ ] Cambiar el system prompt de v4 a v5 y re-correr eval muestra diff con casos afectados.
- [ ] El segundo turno de una conversación reporta `cache_read_tokens > 0` (cache caliente sobre `system + tools`).

## Tests requeridos
- **Unit:** `costUsd` con casos conocidos (input only, input+output, con cache reads).
- **Unit:** `budgetGuardrail.shouldBlock` con varios totales acumulados.
- **Integración:** `callTracked` con mock del SDK — verifica insert, manejo de error, manejo de timeout.
- **Integración:** golden set runner con dos casos triviales contra mock; verifica que el judge se invoca y el reporte se genera.
- **E2E manual:** correr `npm run eval` real y revisar HTML.

## Demo
```bash
# Generar tráfico
# Desde teléfono: 5 turnos de conversación variada (saludo, cotización, pedido, confirmación, escalado)

# Inspeccionar
psql "$DATABASE_URL" -c "
  SELECT prompt_id, prompt_version, input_tokens, output_tokens, cache_read_tokens,
         latency_ms, cost_usd, jsonb_array_length(tools_called) AS n_tools
  FROM llm_traces ORDER BY created_at DESC LIMIT 10;
"

# Costo del día
psql "$DATABASE_URL" -c "
  SELECT created_at::date AS day, SUM(cost_usd) AS usd, COUNT(*) AS calls
  FROM llm_traces GROUP BY 1 ORDER BY 1 DESC LIMIT 7;
"

# Forzar guardrail (bajar el budget temporalmente)
CONVERSATION_TOKEN_BUDGET=500 npm run dev
# Mantener conversación activa hasta que escale automáticamente

# Eval
npm run eval
open evals/reports/$(ls -t evals/reports/ | head -1)
```

## Riesgos del slice
- **Tarifas hardcodeadas se desactualizan:** documentar revisión trimestral en runbook; alternativa futura: leer de un endpoint o config externa.
- **`input` y `output` jsonb crecen el tamaño de la DB rápido:** considerar política de retención (ej. truncar `input`/`output` a 10 KB y guardar resto en object storage cuando se justifique).
- **LLM-as-judge no es determinista:** usar temperature 0 en el judge; aceptar variabilidad y revisar resultados manualmente para casos críticos.
- **Guardrail demasiado agresivo escala conversaciones legítimas largas:** mitigado al excluir `cache_read_tokens` del cómputo; ajustar `CONVERSATION_TOKEN_BUDGET` con datos reales tras 1–2 semanas.
- **Costo del eval pipeline contra Claude real:** golden set de 30 con un par de turnos cada uno corre ~$0.20–$0.50 por ejecución; aceptable para uso manual pre-merge.
- **`call_index` mal incrementado en tool loop:** test que verifica que los traces de un turno tienen `call_index` consecutivo desde 0.
