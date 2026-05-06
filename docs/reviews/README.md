# Reviews — Bitácora de revisión adversarial

Carpeta donde viven todos los reportes generados por el proceso **Adversarial System Review (ASR)** definido en `docs/adversarial-review.md`.

## Convención de naming

| Tipo | Archivo | Cuándo |
|---|---|---|
| Pre-flight | `pre-flight.md` | Una sola vez, antes de Slice 0 |
| Pre-slice | `pre-slice-<N>.md` | Antes de iniciar el slice N (incluyendo `8.5`) |
| Post-slice | `post-slice-<N>.md` | Al cerrar el slice N |
| Cross-slice | `cross-<YYYY-MM-DD>.md` | Revisión periódica de invariantes |
| Incidente | `incident-<YYYY-MM-DD>-<slug>.md` | Tras outage o bug grave en producción |

Ejemplos:
- `pre-flight.md`
- `pre-slice-0.md`
- `post-slice-0.md`
- `pre-slice-1.md`
- `post-slice-1.md`
- `pre-slice-8.5.md`
- `cross-2026-07-01.md`
- `incident-2026-08-12-perdida-mensajes-meta.md`

## Plantilla

`_template.md` es la base. Copiar y completar.

```
cp docs/reviews/_template.md docs/reviews/pre-slice-1.md
```

El archivo `_template.md` tiene prefijo `_` para que el orden alfabético lo deje siempre arriba en listados.

## Flujo

1. **Pre-slice N**: revisor escribe los casos adversariales **antes** de que el constructor empiece a codear. Es contrato.
2. **Construcción**: el constructor implementa el slice. Si descubre que algún caso adversarial necesita un cambio en la spec, vuelve al `pre-slice-N.md` y lo edita con nota.
3. **Post-slice N**: el revisor (idealmente otra persona o sesión) ejecuta los casos del pre-slice + los invariantes globales + los AC del slice. Documenta verdict.
4. **Sólo si verdict = APROBADO o APROBADO CON DEUDA → se inicia Slice N+1.**

## Estado actual

Slices completados con review:

| Slice | Pre-review | Post-review | Verdict |
|---|---|---|---|
| Pre-flight | — | — | pendiente |
| 0 | pendiente | pendiente | — |
| 1..8.5 | pendiente | pendiente | — |

Esta tabla se actualiza al cerrar cada review.

## Disciplina

- Un review **nunca** se borra después de cerrado, ni siquiera si los hallazgos resultaron equivocados. Se anexa una nota correctiva.
- Si el revisor y el constructor son la misma persona (equipo de 1), ver §13 del `adversarial-review.md`: cambio de sombrero formal y/o agente IA dedicado.
- Las deudas marcadas en un post-review se arrastran al pre-review del siguiente slice como ítem explícito a verificar.
