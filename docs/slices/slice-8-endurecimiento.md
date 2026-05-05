# Slice 8 — Endurecimiento

## Objetivo
Llevar el sistema de "funciona en demo" a "funciona bajo condiciones adversas": rate limiting, métricas operativas básicas, runbook y suite de evaluación inicial. Es el endurecimiento operacional general; el específico de LLM es Slice 8.5.

## Dependencias
- Slice 7 completo
- Variables nuevas: `RATE_LIMIT_PER_PHONE_PER_MIN` (default `20`), `METRICS_ENABLED` (default `true`)

## Alcance
**Incluye:**
- Rate limiting por `phone` (sliding window simple) — 429 si excede
- Lista de bloqueo (`blocklist`) por phone
- Métricas Prometheus básicas: latencia `/webhook`, latencia Claude, conteo de mensajes por dirección, conteo de errores
- Endpoint `/metrics` autenticado (token simple, no JWT admin)
- Suite de evaluación de prompts ejecutable manualmente (estructura — el set rico viene en 8.5)
- Runbook básico en `docs/runbook.md`: errores comunes, cómo rotar tokens, cómo restablecer admin
- Migraciones idempotentes verificadas

**No incluye:** observabilidad LLM detallada (Slice 8.5), Sentry/APM completo (deuda futura), HA / despliegue (out of scope).

## Diseño técnico

### Modelo de datos (delta)
```sql
CREATE TABLE blocklist (
  phone      TEXT PRIMARY KEY,
  motivo     TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE rate_limit_events (
  phone      TEXT NOT NULL,
  ts         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_rate_limit_phone_ts ON rate_limit_events (phone, ts);
```
Alternativa: si Redis está habilitado (Slice 2.4 mencionado como opcional), usar `INCR` con TTL; si no, tabla con `DELETE FROM rate_limit_events WHERE ts < NOW() - INTERVAL '1 minute'` periódico.

### Rate limit
Antes de procesar mensaje en webhook:
1. ¿Está en `blocklist`? → 200 a Meta (no podemos rechazar) pero NO procesamos.
2. Contar eventos del último minuto para `phone`. Si > `RATE_LIMIT_PER_PHONE_PER_MIN`:
   - No llamar a Claude.
   - Enviar mensaje fijo "Estás enviando muchos mensajes, espera un momento" (con cooldown de 30 s para no inundar).
3. Insertar evento.

### Métricas (Prometheus client)
```
http_request_duration_seconds{route="/webhook",method="POST"}  histogram
http_requests_total{route,status}                              counter
claude_call_duration_seconds                                   histogram
claude_calls_total{outcome="ok|error|timeout"}                 counter
whatsapp_send_total{outcome}                                   counter
messages_in_total                                              counter
messages_out_total                                             counter
errors_total{component}                                        counter
rate_limit_hits_total{phone_hash}                              counter
```
Endpoint `GET /metrics` con header `Authorization: Bearer ${METRICS_TOKEN}`.

### Suite de evaluación (estructura)
- Carpeta `evals/` con `cases.yaml` (5 casos seed; expansión a 30–50 en 8.5).
- `apps/backend/scripts/eval.ts` — ejecuta los casos contra `claude.respond` real.
- Salida: tabla `pass/fail` por caso + reporte JSON guardado en `evals/reports/<timestamp>.json`.
- `package.json`: `"eval": "tsx apps/backend/scripts/eval.ts"`.
- Manual; no en CI.

### Runbook (`docs/runbook.md`)
Secciones:
- **Cómo rotar `WHATSAPP_TOKEN`** (System User Token en Meta Business)
- **Cómo rotar `ANTHROPIC_API_KEY`**
- **Cómo restablecer credenciales de admin** (re-seed)
- **Errores comunes y diagnóstico:**
  - 401 webhook → `WHATSAPP_APP_SECRET` desincronizado
  - 5xx Claude → estrategia de fallback
  - DB caída → `/health` reporta degraded; verificar `docker ps`
  - Rate limit pegando a clientes legítimos → ajustar `RATE_LIMIT_PER_PHONE_PER_MIN`
- **Backups** mínimos: `pg_dump` manual; deuda: backups automáticos.

## Archivos a crear / modificar
- `infra/migrations/0006_blocklist_ratelimit.sql`
- `apps/backend/src/middleware/rateLimit.ts`
- `apps/backend/src/services/blocklist.ts`
- `apps/backend/src/middleware/metricsAuth.ts`
- `apps/backend/src/routes/metrics.ts`
- `apps/backend/src/observability/metrics.ts` (registry + métricas)
- `apps/backend/scripts/eval.ts`
- `evals/cases.yaml` (5 casos seed)
- `docs/runbook.md`
- `apps/backend/tests/rateLimit.test.ts`

## Criterios de aceptación
- [ ] 21 mensajes en < 60 s desde el mismo phone → mensaje fijo de cooldown, no se llama a Claude.
- [ ] Tras 60 s, el siguiente mensaje pasa normalmente.
- [ ] Phone en `blocklist` → mensaje no se procesa (sin llamada a Claude, sin respuesta saliente).
- [ ] `/metrics` con token correcto → 200 con cuerpo en formato Prometheus.
- [ ] `/metrics` sin token o token incorrecto → 401.
- [ ] Métricas reflejan tráfico real: enviar mensajes y ver contadores subir.
- [ ] `npm run eval` corre los 5 casos seed y produce reporte legible.
- [ ] `docs/runbook.md` cubre las 4 secciones listadas con pasos accionables.
- [ ] Migraciones 0001..0006 corren limpio en una DB nueva.

## Tests requeridos
- **Unit:** ventana de rate limit (límite, justo abajo, justo arriba).
- **Integración:** webhook con phone bloqueado/rate-limited; verifica que Claude no fue llamado.
- **Integración:** `/metrics` con/sin token.
- **Manual:** correr `npm run eval` y revisar reporte.

## Demo
```bash
# Stress: 25 mensajes rápidos al webhook
for i in {1..25}; do curl -X POST .../webhook ... ; done
curl -H "Authorization: Bearer $METRICS_TOKEN" http://localhost:3000/metrics | grep rate_limit
npm run eval
cat evals/reports/$(ls -t evals/reports/ | head -1)
```

## Riesgos del slice
- **Rate limit con tabla SQL en alta carga es costoso:** v1 acepta el costo; documentar migración a Redis si volumen lo justifica.
- **Métricas sin alerting:** las métricas existen pero no avisan; la integración con un Alertmanager/Grafana queda fuera del documento.
- **Eval set chico (5 casos) da falsa sensación de cobertura:** Slice 8.5 expande a 30–50.
