---
name: lens-review
effort: high
user-invocable: true
argument-hint: "[PR# | rama | (vacío = diff actual)] [--comment]"
description: |
  Orquesta una revisión de código multi-agente sobre un diff/PR/rama: hace fan-out a revisores heterogéneos (correctness, seguridad, integridad de tests, blast-radius/arquitectura) en contextos aislados y modelos distintos, valida cada hallazgo adversarialmente, y emite un informe advisory severity-ranked (CRÍTICO/MAYOR/MENOR/NIT) — NUNCA aprueba ni mergea. Tiered por blast-radius (config→ligero · payments/auth/crypto/migración→full). Use cuando el usuario diga "revisa este PR con varios revisores", "code review agéntico", "revisión multi-agente", "audita estos cambios a fondo", "revisa antes de mergear algo de alto riesgo", o pida una segunda/tercera opinión heterogénea sobre un diff.
compatibility: |
  Designed for Claude Code. Orchestrator skill — fan-out to multiple specialized review subagents via the Agent/Task tool. Spanish content with English technical terms. Requires git + gh CLI. Advisory-only (sensor, not gate).
metadata:
  category: workflow-preference
  version: "0.2.0"
  author: "Antonio Romero"
  framework: "AI-DLC 2026"
---

# lens-review

Orquestador de revisión de código **multi-agente**. No revisas tú solo: haces **fan-out** a revisores heterogéneos (contextos aislados, modelos distintos, priors distintos), **validas** cada hallazgo adversarialmente, y **sintetizas** un informe advisory. La heterogeneidad es la propiedad que hace que esto funcione — distintos revisores cazan conjuntos de bugs **disjuntos** (ver [`methodology/heterogeneity.md`](methodology/heterogeneity.md)).

Esta skill es un **sensor, no un veredicto**: nunca aprueba, comenta en el PR (sin flag explícito) ni mergea. El muro determinista vive en CI, no aquí (ver [`methodology/sensor-not-verdict.md`](methodology/sensor-not-verdict.md) y [`references/deterministic-gates.md`](references/deterministic-gates.md)).

## Cuándo aplicar

Cuando hay un diff/PR/rama y se quiere una revisión profunda y heterogénea antes de mergear. Tres entradas típicas:

1. **Rama local con commits** → revisar `git diff <base>...HEAD` (base normalmente `dev`).
2. **PR abierto** → `gh pr diff <N>` + contexto del PR.
3. **Diff suelto / staged** → `git diff` o `git diff --staged`.

## Cuándo NO aplicar

- **Cambio cosmético/mecánico** (formateo masivo, rename, mover archivos): no necesita fan-out; una pasada de `pr-and-linear-hygiene` de `ai-dlc-pr-readiness-review` basta. Si dudas, Tier 0 lo resuelve.
- **Aún no hay código** (solo idea/plan): usa `ai-dlc-plan-design-review`.
- **Quieres construir el body del PR**, no revisar: usa `ai-dlc-pr-readiness-review`. (Esta skill audita; aquella estructura el PR. Se complementan.)

## Composición con el toolkit de PR ops

- **`stamp-check`** → sus `ESCALATE` de tier alto son la entrada típica de esta skill.
- **`review-triage`** → dispone lo que esta skill encuentra: con `--comment` los hallazgos se vuelven threads del PR que triage clasifica y ejecuta (fix claros / nits / defer). Es el loop del artículo de PostHog: swarm → triage → iterar hasta que no quede nada accionable.
- **`babysit-prs`** → orquesta el ciclo en bucle desatendido y sugiere re-correr esta skill cuando hay commits sustantivos nuevos.
- **`pr-evidence`** → la otra mitad de la verificación: esta skill razona sobre el diff; aquella lo ejecuta y observa. En Tier 2, pedir ambas.

## Principio rector — sensor, no veredicto

| La skill SÍ hace | La skill NUNCA hace |
|---|---|
| Emitir hallazgos clasificados por severidad | `gh pr review --approve` / aprobar |
| Recomendar bloquear o seguir | `gh pr merge` / mergear |
| Escribir el informe a conversación + `/tmp/` | Comentar en el PR salvo flag `--comment` explícito |
| Señalar dónde vive el gate vinculante (CI) | Sustituir el gate de CI / branch protection |

El humano es dueño del merge. Detalle en [`methodology/sensor-not-verdict.md`](methodology/sensor-not-verdict.md).

---

## El pipeline (5 pasos)

```
1. INTAKE GATE   → ¿hay intención declarada? ¿tests corridos? ¿diff acotado?  (si no → pedir, no revisar)
2. TIERING       → blast-radius → Tier 0 / 1 / 2 → set de revisores
3. FAN-OUT       → Agent()×N en paralelo, contexto fresco por revisor
4. VALIDACIÓN    → un validador escéptico por hallazgo ("default refuted")
5. SÍNTESIS      → dedupe + rank (CRÍTICO/MAYOR/MENOR/NIT) + intent log + informe advisory
```

### Paso 1 — Intake gate

Antes de gastar revisores, exige el mínimo. Detalle en [`prompts/intake-and-tiering.md`](prompts/intake-and-tiering.md).

- **Intención declarada**: ¿qué intenta hacer este cambio y por qué? (del PR body, del ticket Linear, o pregúntalo). Sin intención, el revisor no puede juzgar si el código hace lo correcto — solo si "se ve bien".
- **Tests corridos + output**: ¿se ejecutaron los tests? ¿con qué resultado? Un diff que nunca se corrió no se revisa a fondo (se señala como bloqueo de intake).
- **Diff acotado**: si el diff es enorme (>~1500 LOC), señálalo — la calidad de revisión cae con el tamaño. Sugiere partir; revisa de todas formas pero declara la limitación.

Si falta algo: **NO arranques el fan-out**. Pide el artefacto que falta (o constrúyelo: `git diff`, correr tests). En Tier 2 (alto blast-radius) el **intent/decision log es obligatorio** (ver Paso 2 y [`templates/intent-decision-log.md`](templates/intent-decision-log.md)).

### Paso 2 — Tiering por blast-radius

Clasifica el cambio y dimensiona el fan-out. Matriz completa en [`tiers/blast-radius-tiers.md`](tiers/blast-radius-tiers.md).

| Tier | El cambio toca… | Revisores | Extra |
|---|---|---|---|
| **0** Ligero | config, docs, cosmético, rename mecánico | 1 — `lens-review:lens-architecture` (convenciones) | — |
| **1** Estándar | feature/fix normal, lógica de dominio | 3 — `lens-review:lens-correctness` + `lens-review:lens-tests` + `lens-review:lens-architecture` | validación per-finding |
| **2** Full | payments, auth, crypto, migración DB, concurrencia, permisos, borrado de datos | 4-5 — añade `lens-review:lens-security` | **intent log obligatorio** + validación adversarial + dueño humano nombrado |

Determina el tier por **lo que el cambio puede romper si está mal**, no por su tamaño en líneas. Un one-liner en el path de pagos es Tier 2. Calcula blast-radius real (callers de lo que cambia) — ver lente [`lenses/blast-radius-architecture.md`](lenses/blast-radius-architecture.md).

### Paso 3 — Fan-out

Lanza los revisores del tier **en paralelo** con la herramienta Agent, cada uno con **contexto fresco** (nunca heredan tu sesión). Detalle del dispatch en [`prompts/dispatch.md`](prompts/dispatch.md); reutiliza el patrón de `dispatching-parallel-agents`.

Cada revisor recibe una tarea construida desde [`templates/subagent-task.md`](templates/subagent-task.md): scope exacto, contexto (intención + base/head SHA + archivos), su lente, y formato de salida.

**Cómo despachar cada revisor** (la heterogeneidad de modelo la da el parámetro `model` del Agent tool; la de prior, la lente embebida):

- **Default robusto (funciona siempre)**: `agentType="general-purpose"` + `model: <opus|sonnet>` + la **lente embebida** en el prompt (vía `templates/subagent-task.md`).
- **Preferido cuando esté disponible**: `agentType="lens-review:lens-<dim>"` (subagentes del plugin, que ya traen el system prompt + el modelo fijado). Nota: al crearlos por primera vez, Claude Code los registra con un **pequeño retardo** (re-escaneo en background) — **no hace falta reiniciar**. Durante ese intervalo de segundos usa el default robusto; el resultado es equivalente. Comprueba la lista de agentType: si ves los `lens-review:lens-*`, úsalos.

| Revisor | `model` | `agentType` (preferido / fallback) | Lente |
|---|---|---|---|
| Correctness | **fable** (fallback opus) | `lens-review:lens-correctness` / `general-purpose` | [`lenses/correctness-and-bugs.md`](lenses/correctness-and-bugs.md) |
| Seguridad | **fable** (fallback opus) | `lens-review:lens-security` / `general-purpose` | [`lenses/security-and-injection.md`](lenses/security-and-injection.md) |
| Integridad de tests | opus | `lens-review:lens-tests` / `general-purpose` | [`lenses/test-integrity.md`](lenses/test-integrity.md) |
| Arquitectura/convenciones | sonnet | `lens-review:lens-architecture` / `general-purpose` | [`lenses/blast-radius-architecture.md`](lenses/blast-radius-architecture.md) + [`lenses/conventions-claudemd.md`](lenses/conventions-claudemd.md) |

**Heterogeneidad**: 2×Fable + Opus + Sonnet — tres tiers de modelo con lentes/system-prompts distintos = blind spots distintos (mismo split que el qa-swarm de referencia: el modelo más fuerte para los revisores de profundidad técnica). Si el harness rechaza `fable` (Claude Code antiguo), cae a `opus` para ese agente. NO uses el mismo modelo+prompt para todos: eso es un revisor con N copias, no N revisores.

**Orquestación preferida en Tier 1/2**: si la tool `Workflow` está disponible, úsala en lugar del despacho manual — pipeline `review → validate` sin barrera (los hallazgos de un revisor entran a validación mientras los demás aún revisan) + salida estructurada por schema (sin parsear texto). Ver "Orquestación vía Workflow" en [`prompts/dispatch.md`](prompts/dispatch.md). El despacho con Agent es el fallback universal; Tier 0 (1 revisor) no amerita Workflow.

### Paso 4 — Validación per-finding

Cada hallazgo de cada revisor se valida con un subagente **escéptico independiente** (prompt: "intenta refutarlo; default = refutado si hay duda"). Detalle en [`prompts/validation.md`](prompts/validation.md) + [`methodology/dont-trust-the-report.md`](methodology/dont-trust-the-report.md).

- El validador **lee el código real**, no confía en el reporte del revisor.
- Hallazgo que el validador no puede confirmar → **se filtra** (no llega al informe).
- Los falsos positivos erosionan la confianza más rápido que un falso negativo. Solo pasa lo de **alta señal**.
- Optimización: en Tier 0/1 puedes validar solo CRÍTICO/MAYOR; en Tier 2, todo.

### Paso 5 — Síntesis

Consolida y emite. Detalle en [`prompts/synthesis.md`](prompts/synthesis.md).

1. **Dedupe** entre revisores (un hallazgo en el mismo file:line desde dos lentes = uno, atribuido a la lente más específica). Nota: que dos revisores coincidan es señal de alta confianza; que casi nunca coincidan es lo esperado (heterogeneidad).
2. **Rank** por severidad (rúbrica abajo).
3. **Intent/decision log**: en Tier 2, exige/genera el log ([`templates/intent-decision-log.md`](templates/intent-decision-log.md)) — qué se intentó, alternativas rechazadas, por qué.
4. **Informe advisory** con [`templates/review-report.md`](templates/review-report.md): TL;DR + tabla de hallazgos + veredicto recomendado (no vinculante) + dónde está el gate real.

---

## Rúbrica de severidad

Misma rúbrica que `ai-dlc-pr-readiness-review` / `ai-dlc-plan-design-review`:

| Severidad | Etiqueta UI | Recomienda bloquear |
|---|---|---|
| CRÍTICO | 🔴 blocking | Sí |
| MAYOR | 🟡 important | Sí |
| MENOR | 🟢 nit | No |
| NIT | 💡 suggestion | No |

**Veredicto recomendado** (no vinculante): "Listo para merge" = zero CRÍTICO + zero MAYOR confirmados. La decisión es del humano; el gate es CI.

---

## Forma del output

```
Informe → conversación (siempre) + opcional /tmp/{feature}-lens-review.md (scratch, NO commiteable)
Hallazgos confirmados → tool ReportFindings si está disponible (typed, la UI los renderiza) — ver synthesis.md
Intent log (Tier 2) → /tmp/{feature}-intent-log.md (o sección del informe)
Comentarios en PR → SOLO con flag --comment explícito, con las reglas de publicación de synthesis.md
                    (bot-header en todo + inline como UNA review + summary sticky upserted)
```

`/tmp/` y no `docs/`: el informe es scratch para auditar pre-merge, no documentación histórica.

---

## Índice de archivos

| Archivo | Cuándo leer |
|---|---|
| [`prompts/intake-and-tiering.md`](prompts/intake-and-tiering.md) | Paso 1-2: intake gate + clasificación de tier |
| [`prompts/dispatch.md`](prompts/dispatch.md) | Paso 3: cómo lanzar el fan-out |
| [`prompts/validation.md`](prompts/validation.md) | Paso 4: validación adversarial per-finding |
| [`prompts/synthesis.md`](prompts/synthesis.md) | Paso 5: dedupe + rank + informe |
| [`tiers/blast-radius-tiers.md`](tiers/blast-radius-tiers.md) | Matriz de tiering por blast-radius |
| [`lenses/README.md`](lenses/README.md) | Selección de lentes por revisor |
| [`lenses/correctness-and-bugs.md`](lenses/correctness-and-bugs.md) | Lente de `lens-review:lens-correctness` |
| [`lenses/security-and-injection.md`](lenses/security-and-injection.md) | Lente de `lens-review:lens-security` |
| [`lenses/test-integrity.md`](lenses/test-integrity.md) | Lente de `lens-review:lens-tests` |
| [`lenses/blast-radius-architecture.md`](lenses/blast-radius-architecture.md) | Lente de `lens-review:lens-architecture` (arquitectura) |
| [`lenses/conventions-claudemd.md`](lenses/conventions-claudemd.md) | Lente de `lens-review:lens-architecture` (convenciones) |
| [`templates/subagent-task.md`](templates/subagent-task.md) | Construir la tarea de cada revisor |
| [`templates/review-report.md`](templates/review-report.md) | Formato del informe advisory |
| [`templates/intent-decision-log.md`](templates/intent-decision-log.md) | Intent/decision log (Tier 2) |
| [`methodology/heterogeneity.md`](methodology/heterogeneity.md) | Por qué priors/modelos distintos |
| [`methodology/sensor-not-verdict.md`](methodology/sensor-not-verdict.md) | Advisory-only; humano dueño del merge |
| [`methodology/dont-trust-the-report.md`](methodology/dont-trust-the-report.md) | Verificación independiente per-finding |
| [`references/deterministic-gates.md`](references/deterministic-gates.md) | Dónde vive el muro (CI + branch protection) |
| [`examples/full-pr-multiagent-review.md`](examples/full-pr-multiagent-review.md) | Ejemplo dogfooded end-to-end |

---

## Anti-patterns

- **Mismo modelo+prompt para todos los revisores**. Eso no es heterogéneo; son copias del mismo blind spot. Varía modelo Y prior.
- **Saltarse el intake gate "porque tengo prisa"**. Revisar sin intención = juzgar estética, no corrección.
- **Pasar todos los hallazgos sin validar**. Los falsos positivos matan la confianza. Filtra a alta señal.
- **Que un revisor herede tu contexto de sesión**. Construye exactamente lo que necesita; nada más.
- **Aprobar / mergear / comentar en el PR**. Esta skill es sensor. El gate es CI. Sin flag `--comment`, ni un comentario.
- **Tratar tamaño = riesgo**. Un one-liner en auth es Tier 2; un refactor de 800 líneas de tests puede ser Tier 1.
- **Inflar el informe**. Si una lente encontró 0 hallazgos, dilo. Mejor "lente X: cero hallazgos" que inventar nits.
