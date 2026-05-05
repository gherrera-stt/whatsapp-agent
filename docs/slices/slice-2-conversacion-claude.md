# Slice 2 — Conversación con Claude

## Objetivo
Reemplazar el eco por una respuesta natural generada por Claude, con contexto multi-turno y prompt caching habilitado. El cliente puede saludar y conversar; el modelo recuerda el turno anterior dentro de la misma conversación.

## Dependencias
- Slice 1 completo
- `ANTHROPIC_API_KEY` válida con cupo y acceso a `claude-sonnet-4-6`
- Variables nuevas: `ANTHROPIC_API_KEY`, `CLAUDE_MODEL` (default `claude-sonnet-4-6`), `CLAUDE_MAX_TOKENS` (default `1024`), `CLAUDE_TEMPERATURE` (default `0.3`), `HISTORY_TURNS` (default `10`)

## Alcance
**Incluye:**
- Servicio `claude` con SDK oficial `@anthropic-ai/sdk`
- System prompt versionado (`system.sales.v1`) en módulo TS
- Construcción de contexto: últimos N mensajes de la conversación + system prompt
- **Prompt caching** del system prompt (cache breakpoint en el bloque de system)
- Reemplazo del eco por la respuesta del modelo
- Persistencia de respuesta en `messages` con `direccion='out'`

**No incluye:** tools (Slice 3+), eval pipeline (Slice 8.5), traces estructurados (Slice 8.5; aquí solo log básico).

## Diseño técnico

### Estructura `apps/backend/src/prompts/`
```
prompts/
├── index.ts                # registry: id → función
├── system/
│   └── sales.v1.ts         # system prompt comercial
└── README.md               # convenciones de versionado
```

### Forma del módulo de prompt
```ts
// prompts/system/sales.v1.ts
export const id = 'system.sales';
export const version = 1;

export function build(): { system: SystemBlock[] } {
  return {
    system: [{
      type: 'text',
      text: `Eres un asistente comercial en español...`,
      cache_control: { type: 'ephemeral' },
    }],
  };
}
```

### Servicio `claude.respond`
Firma:
```ts
async function respond(args: {
  conversationId: number;
  history: Array<{role:'user'|'assistant'; content:string}>;
  userMessage: string;
}): Promise<{
  text: string;
  promptId: string;
  promptVersion: number;
  usage: { input: number; output: number; cacheRead: number; cacheCreation: number };
  latencyMs: number;
}>
```

### Construcción de contexto
1. Cargar últimos `HISTORY_TURNS` mensajes (in + out, sin marcadores `system`) ordenados ascendente.
2. Mapear a `messages: [{role: 'user'|'assistant', content}]`.
3. Añadir el `userMessage` actual al final como `role:'user'`.
4. `system` viene del prompt builder con `cache_control: ephemeral`.
5. Llamar `client.messages.create({ model, system, messages, max_tokens, temperature })`.

### Integración en el flujo del webhook
- Donde antes se hacía eco, ahora:
  1. Cargar historial.
  2. Llamar `claude.respond`.
  3. Enviar `result.text` por `whatsapp.sendText`.
  4. Persistir mensaje saliente con `intent` provisional desde el system prompt si lo etiqueta (opcional en este slice).
- Si Claude falla (timeout, 5xx tras 1 reintento, 429): enviar texto fijo "Estoy con problemas técnicos, intenta en un momento" y log `error`. No reintentar al cliente.

### Configuración de Claude
- `model`: `claude-sonnet-4-6`
- `max_tokens`: 1024
- `temperature`: 0.3
- Timeout HTTP: 15 s
- Retry: 1 con backoff 1 s solo en 5xx/timeout

### Logging mínimo (Slice 8.5 lo formaliza)
Por ahora `pino.info` con `{conversationId, promptId, promptVersion, inputTokens, outputTokens, cacheReadTokens, latencyMs}`.

## Archivos a crear / modificar
- `apps/backend/src/services/claude/client.ts`
- `apps/backend/src/services/claude/respond.ts`
- `apps/backend/src/prompts/index.ts`
- `apps/backend/src/prompts/system/sales.v1.ts`
- `apps/backend/src/prompts/README.md`
- `apps/backend/src/routes/webhook.ts` (modificado: reemplazar eco)
- `apps/backend/src/services/messages.ts` (función `loadHistory`)
- `apps/backend/tests/claude.respond.test.ts`
- `.env.example` (añadir variables Claude)

## Criterios de aceptación
- [ ] Saludo "hola" recibe respuesta coherente en español, no el eco.
- [ ] Segundo turno referencia el primero (ej. "como te dije antes" funciona).
- [ ] El historial cargado respeta `HISTORY_TURNS` (no más, no menos cuando hay suficiente).
- [ ] Conversaciones distintas (`phone` distinto) no comparten contexto.
- [ ] La respuesta queda en `messages` con `direccion='out'`.
- [ ] El `system` se envía con `cache_control: ephemeral`; segunda llamada en < 5 min reporta `cache_read_tokens > 0`.
- [ ] Latencia P95 end-to-end < 4 s con prompt cache caliente.
- [ ] Si `ANTHROPIC_API_KEY` es inválida: respuesta de fallback al cliente y log `error`.
- [ ] Si Claude tarda > 15 s: timeout y respuesta de fallback.
- [ ] El system prompt rechaza temas fuera de comercio (smoke test).

## Tests requeridos
- **Unit:** `loadHistory` (orden, límite, filtros). Builder del prompt (estructura, presencia de `cache_control`).
- **Integración:** `claude.respond` con mock del SDK Anthropic — verifica que `system`, `messages`, `model`, `temperature` son los esperados.
- **E2E manual:** dos turnos reales contra la API y verificar coherencia + cache hit en segundo turno.

## Demo
```bash
# Asegurar ANTHROPIC_API_KEY en .env
npm run dev --workspace=apps/backend
# Desde teléfono: enviar "hola"
# Esperar respuesta natural
# Enviar "¿cómo te llamas?"
# Verificar que la respuesta es contextual
psql "$DATABASE_URL" -c "
  SELECT direccion, contenido FROM messages
  WHERE conversation_id = (SELECT id FROM conversations WHERE phone = '<tu_phone>')
  ORDER BY created_at;
"
```

## Riesgos del slice
- **Costo descontrolado por falta de cache:** verificar manualmente `cache_read_tokens` en el primer test; si es 0 después del segundo turno, el cache no está bien configurado.
- **Latencia > 4 s sin streaming:** v1 sin streaming; si latencia molesta, considerar enviar mensaje "escribiendo…" antes (no en este slice).
- **Inyección de prompt vía mensaje del cliente:** el system prompt debe instruir al modelo a ignorar instrucciones del usuario que pidan cambiar comportamiento; reforzar en versiones siguientes.
- **Historial truncado mal calculado pierde contexto importante:** validar visualmente en demo.
