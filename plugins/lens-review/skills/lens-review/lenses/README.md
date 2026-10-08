# Lentes — revisión agéntica

Cada revisor del fan-out carga **una** lente (salvo `lens-review:lens-architecture`, que carga dos). La lente es su metodología de revisión: ejes a recorrer, preguntas guía, hallazgo típico, anti-patterns. Estructura común heredada de `ai-dlc-pr-readiness-review/lenses/`.

## Catálogo

| Lente | Archivo | Revisor | Modelo |
|---|---|---|---|
| Correctness & bugs | [`correctness-and-bugs.md`](correctness-and-bugs.md) | `lens-review:lens-correctness` | opus |
| Seguridad & injection | [`security-and-injection.md`](security-and-injection.md) | `lens-review:lens-security` | opus |
| Integridad de tests | [`test-integrity.md`](test-integrity.md) | `lens-review:lens-tests` | sonnet |
| Blast-radius & arquitectura | [`blast-radius-architecture.md`](blast-radius-architecture.md) | `lens-review:lens-architecture` | sonnet |
| Convenciones (CLAUDE.md) | [`conventions-claudemd.md`](conventions-claudemd.md) | `lens-review:lens-architecture` | sonnet |

## Selección por tier

| Tier | Lentes (revisores) |
|---|---|
| **0** | blast-radius-architecture + conventions (`lens-review:lens-architecture`) |
| **1** | + correctness-and-bugs (`lens-review:lens-correctness`) + test-integrity (`lens-review:lens-tests`) |
| **2** | + security-and-injection (`lens-review:lens-security`) [+ revisor externo cross-family si opt-in] |

## Cómo aplica cada revisor su lente

Por cada eje de la lente:
- **Cubierto**: el código lo maneja bien → anotar brevemente.
- **Hallazgo**: no lo maneja o lo maneja mal → formular en formato canónico (`SEVERIDAD · lente · file:line — problema · fix · confianza`).
- **No aplica**: irrelevante para este diff → justificar en una frase.

No omitir ejes en silencio. Pasar por todos evita "se me olvidó mirar X". Cero hallazgos en una lente es legítimo — se declara, no se infla.

## Atribución (anti-duplicado)

Un hallazgo se atribuye a **una** lente (la más específica). Si encaja en dos, la otra lo menciona como contexto, no lo repite. La síntesis hace dedupe final entre revisores.
