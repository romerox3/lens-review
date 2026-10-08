# lens-review — guía rápida

Skill orquestadora de **code review multi-agente** para Claude Code. Hace fan-out a revisores heterogéneos, valida cada hallazgo, y emite un informe **advisory** (nunca aprueba ni mergea).

## Qué es y qué no es

- **Es**: un sensor que encuentra y prioriza problemas en un diff/PR/rama, con varios revisores de modelos/priors distintos.
- **No es**: un gate. No aprueba, no mergea, no comenta en el PR (salvo `--comment`). El muro vinculante vive en CI (ver [`references/deterministic-gates.md`](references/deterministic-gates.md)).

## Cómo se invoca

Por lenguaje natural (auto-trigger): *"revisa este PR con varios revisores"*, *"code review agéntico de esta rama"*, *"audita estos cambios a fondo antes de mergear"*. O explícito: `/lens-review:lens-review`.

Flags opcionales:
- `--comment` — postea los hallazgos como comentarios inline en el PR (nunca aprueba).

## El pipeline (5 pasos)

1. **Intake gate** — exige intención declarada + tests corridos + diff acotado. Sin eso, no revisa.
2. **Tiering** — clasifica por blast-radius (Tier 0 ligero / 1 estándar / 2 full) y dimensiona el fan-out.
3. **Fan-out** — lanza N revisores en paralelo, contexto fresco, modelos distintos.
4. **Validación** — un validador escéptico por hallazgo ("default refuted") filtra a alta señal.
5. **Síntesis** — dedupe + rank (CRÍTICO/MAYOR/MENOR/NIT) + intent log (Tier 2) + informe advisory.

## Los revisores (subagentes del plugin, en `agents/`)

| Revisor | Modelo | Foco |
|---|---|---|
| `lens-review:lens-correctness` | opus | bugs, lógica, edge cases, concurrencia |
| `lens-review:lens-security` | opus | injection, authz, SSRF, secrets, prompt-injection (Tier 2) |
| `lens-review:lens-tests` | sonnet | integridad del test-diff (asserts reescritos, mutation thinking) |
| `lens-review:lens-architecture` | sonnet | blast-radius, acoplamiento, convenciones CLAUDE.md |

> **Nota de carga**: tras instalar el plugin por primera vez, los agentes `lens-review:lens-*`, Claude Code los registra con un **pequeño retardo** (re-escaneo en background) — **no hace falta reiniciar**. Durante ese intervalo la skill funciona igual despachando vía `general-purpose` + override de `model` + la lente embebida (mismo resultado; ver `prompts/dispatch.md`). La **skill** sí está disponible al instante de crearla.

## Heterogeneidad (la propiedad clave)

Distintos revisores cazan bugs **disjuntos** (Osmani: 93,4% de hallazgos los caza un solo revisor). Por eso se usan modelos Y priors distintos, no N copias del mismo. Detalle: [`methodology/heterogeneity.md`](methodology/heterogeneity.md).

## Estructura

```
SKILL.md           orquestador (empezar aquí)
prompts/           intake-and-tiering · dispatch · validation · synthesis
lenses/            una metodología de revisión por revisor
tiers/             matriz de blast-radius
templates/         subagent-task · review-report · intent-decision-log
methodology/       heterogeneity · sensor-not-verdict · dont-trust-the-report
references/        deterministic-gates
examples/          ejemplo dogfooded end-to-end
evals/             prompts de disparo
```

## Relación con otras skills

- **`ai-dlc-pr-readiness-review`**: audita pre-PR y **construye el body**. Esta skill **revisa con fan-out**; aquella estructura el PR. Se complementan (revisa con esta → construye el PR con aquella).
- **`differential-review`**: revisión diferencial security-focused secuencial (1 agente, 6 fases). Esta reusa su modelado adversarial y blast-radius pero en fan-out paralelo.
- **`dispatching-parallel-agents`**: el patrón de fan-out que esta skill aplica al code review.
