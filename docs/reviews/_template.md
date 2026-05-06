# <Pre-flight | Pre-slice N | Post-slice N | Cross | Incident> Review — <YYYY-MM-DD>

> Plantilla. Copiar a `docs/reviews/<naming>.md` y completar. Ver `docs/adversarial-review.md` §11.

**Revisor:** <nombre humano o agente IA — debe ser distinto del constructor>
**Constructor:** <nombre>
**Slice / Alcance:** <Slice N — Título | Pre-flight | etc.>
**Fecha de inicio:** YYYY-MM-DD
**Fecha de cierre:** YYYY-MM-DD
**Estado:** EN CURSO | APROBADO | APROBADO CON DEUDA | RECHAZADO

---

## 1. Contexto

Una o dos frases: qué se revisa y por qué ahora. Referenciar el commit / branch / PR específicos.

- Commit base: `<hash>`
- Rama: `<branch>`
- Documentación que rige: `docs/slices/slice-N-*.md` (versión a la fecha del review)

---

## 2. Casos adversariales

> En `pre-slice-N.md` se **declaran** los casos que el slice debe sobrevivir. En `post-slice-N.md` se **ejecutan** y se reporta el resultado.

| # | Caso | Origen (§ del adversarial-review.md o riesgo del slice) | Resultado | Severidad si falla | Evidencia |
|---|---|---|---|---|---|
| 1 | Ej. *"Mismo `whatsapp_msg_id` enviado 5 veces"* | §10.2 + I-2 | PASS / FAIL / N/A | CRÍTICO / ALTO / MEDIO / BAJO | query psql, log, screenshot |
| 2 | ... | ... | ... | ... | ... |

**Casos adicionales detectados durante la revisión** (no estaban en el pre-slice):
- ...

---

## 3. Invariantes verificados

> Repasar §9 del adversarial-review.md y marcar uno por uno los invariantes que aplican al slice.

| # | Invariante | Estado | Evidencia |
|---|---|---|---|
| I-1 | Persistir entrante antes del ack 200 | OK / ROTO / NO APLICA | ... |
| I-2 | Idempotencia por `whatsapp_msg_id` | ... | ... |
| I-3 | Lock por conversación al procesar turno | ... | ... |
| I-4 | `loadHistory` excluye system/media + handoff | ... | ... |
| I-5 | Datos de catálogo sólo vía tool de DB | ... | ... |
| I-6 | `cache_control` en posición correcta (system+tools) | ... | ... |
| I-7 | `crear_pedido` rechaza precio cambiado | ... | ... |
| I-8 | `confirmar_pedido` revalida stock+precio en TX | ... | ... |
| I-9 | Conversación humano no llama a Claude | ... | ... |
| I-10 | `escalateToHuman` idempotente | ... | ... |
| I-11 | `/api/admin/*` rechaza sin cookie | ... | ... |
| I-12 | Cookie `Secure` condicional a `NODE_ENV` | ... | ... |
| I-13 | Logs redactan `phone` y `contenido` | ... | ... |
| I-14 | Cada SDK call → fila en `llm_traces` | ... | ... |
| I-15 | `cache_read_tokens > 0` en 2º turno | ... | ... |
| I-16 | Guardrail de tokens dispara escalateToHuman | ... | ... |
| I-17 | Alerta diaria no se duplica tras restart | ... | ... |
| I-18 | Migraciones idempotentes | ... | ... |
| I-19 | Queries parametrizadas (sin concatenación) | ... | ... |
| I-20 | Verificación HMAC con `timingSafeEqual` | ... | ... |

---

## 4. Categorías de ataque revisadas

> Marcar las categorías del §8 que se exploraron y un resumen de hallazgos.

- [ ] §8.1 Corrupción de estado — *resumen*
- [ ] §8.2 Race conditions — ...
- [ ] §8.3 Crashes y restarts — ...
- [ ] §8.4 Costo descontrolado — ...
- [ ] §8.5 Seguridad — ...
- [ ] §8.6 Gaps de observabilidad — ...
- [ ] §8.7 Drift entre slices — ...
- [ ] §8.8 UX y consistencia comercial — ...
- [ ] §8.9 Datos en zonas grises — ...

---

## 5. Hallazgos

> Cada hallazgo: descripción, repro, severidad, fix propuesto.

### 5.1 CRÍTICO (bloqueantes)
- (ninguno) o:
  - **Título.** Descripción.
    - **Repro:** pasos exactos.
    - **Impacto:** qué se rompe / quién lo nota.
    - **Fix mínimo:** qué se debe cambiar.

### 5.2 ALTO (cerrar antes del próximo slice)
- ...

### 5.3 MEDIO (deuda con dueño y fecha)
- ...

### 5.4 BAJO (observación)
- ...

---

## 6. Verificación cross-slice

> Aplica solo a post-slice N ≥ 1. Confirmar que el slice anterior sigue funcionando.

- [ ] Demo del Slice N-1 ejecutado contra el código actual: pasa / falla.
- [ ] Migraciones desde DB virgen hasta el commit actual: corren limpio.
- [ ] AC del Slice N-1 aún se cumplen tras los cambios del Slice N.

---

## 7. Auditoría de logs y métricas (post-slice)

- [ ] Logs del demo no contienen PII en claro.
- [ ] `requestId` se propaga en todo el ciclo de un mensaje.
- [ ] Métricas Prometheus (si aplica) reflejan el demo.
- [ ] `llm_traces` (si aplica) muestra `cache_read_tokens > 0` en 2º turno.

---

## 8. Decisión

**Verdict:** APROBADO | APROBADO CON DEUDA | RECHAZADO

### Justificación
1–3 frases.

### Si APROBADO CON DEUDA
| Hallazgo | Severidad | Dueño | Fecha máxima |
|---|---|---|---|
| ... | MEDIO | <nombre> | YYYY-MM-DD |

### Si RECHAZADO
Lista exacta de qué se requiere para volver a someter:
- ...

---

## 9. Próximos pasos

- Ítems para el constructor.
- Ítems para el siguiente pre-slice (qué carry-over de deudas).
- Si el review descubrió un nuevo invariante o categoría de ataque que vale la pena formalizar: nota para actualizar `docs/adversarial-review.md`.
