# Ejemplo dogfooded — revisión multi-agente end-to-end

Recorrido completo de una revisión Tier 2 sobre un PR de pagos. Ilustra los 5 pasos con artefactos reales (ficticios pero realistas).

## Entrada

Usuario: *"revisa con varios revisores el PR #312 antes de mergear, toca el cálculo de facturación"*.

## Paso 1 — Intake gate

```bash
gh pr view 312 --json title,body,headRefName,baseRefName
gh pr diff 312 --stat   # 3 archivos, +87 −12
```

- **Intención**: del PR body → "corrige el cursor de facturación que saltaba el primer día del ciclo; ahora arranca en `cycleStart` inclusivo".
- **Tests corridos**: el body dice "17/17 pass" → ✅ hay output.
- **Diff acotado**: 99 LOC → ✅.
- **Tier 2 → intent log obligatorio**: el PR no lo trae. Se construye con el autor (alternativas: "¿reset del cursor o exclusión mutua?" → se eligió reset; invariante: "el cursor es inclusivo en `cycleStart`, exclusivo en `cycleEnd`").

Intake ✅ → continúa.

## Paso 2 — Tiering

`billing/cursor.ts` → dominio **pagos** → **Tier 2** automático. Blast-radius: `computeCursor` tiene 4 callers (worker de facturación, endpoint de preview, cron de reconciliación, test). Revisores: `lens-review:lens-correctness` + `lens-review:lens-security` + `lens-review:lens-tests` + `lens-review:lens-architecture`.

## Paso 3 — Fan-out (4 en paralelo, contexto fresco)

Cada uno recibe: intención + `base`/`head` SHA + `gh pr diff 312` + su lente. Devuelven:

- **lens-review:lens-correctness**:
  `CRÍTICO · correctness · billing/cursor.ts:44 — al cambiar a inclusivo en cycleStart, el cron de reconciliación (reconcile.ts:88) sigue asumiendo exclusivo → doble cobro del primer día · alinear reconcile.ts o documentar el invariante con un guard · confianza: alta`
- **lens-review:lens-tests**:
  `MAYOR · test-integrity · cursor.test.ts:31 — el PR reescribe 4 asserts al nuevo valor; no hay test que falle ANTES del fix reproduciendo el salto del primer día · añadir regression test del bug original · confianza: media`
- **lens-review:lens-security**: `security: cero hallazgos`
- **lens-review:lens-architecture**:
  `MENOR · architecture · billing/cursor.ts:44 — el invariante inclusivo/exclusivo vive solo en un comentario; un cambio futuro lo rompe sin que falle el build · extraer a un tipo branded o un assert compartido · confianza: alta`

## Paso 4 — Validación per-finding (Tier 2 → todo)

- Validador del CRÍTICO: lee `reconcile.ts:88`. **CONFIRMADO** — el cron usa `>` donde ahora debería ser `>=`; escenario concreto: ciclo que arranca un día 1, el cron lo cuenta otra vez. Severidad confirmada CRÍTICO.
- Validador del MAYOR (tests): **CONFIRMADO** — los asserts efectivamente se reescribieron sin regression test previo.
- Validador del MENOR (architecture): **CONFIRMADO** como MENOR.

Ninguno refutado. (Si el CRÍTICO hubiera resultado no alcanzable, se descartaría aquí.)

## Paso 5 — Síntesis → informe advisory

```markdown
# Revisión agéntica — PR #312 (cursor de facturación)

## TL;DR
- Tier: 2 (pagos · blast-radius 4 callers incl. cron de reconciliación)
- Revisores: lens-review:lens-correctness, lens-review:lens-security, lens-review:lens-tests, lens-review:lens-architecture (4 en paralelo)
- Hallazgos confirmados: 🔴 1 · 🟡 1 · 🟢 1 · 💡 0
- Veredicto recomendado (NO vinculante): Resolver antes de merge — el cron de reconciliación
  hereda el cambio de invariante y duplica el cargo del primer día. El gate es CI.

## Hallazgos
| # | Sev | file:line | Problema | Fix | Conf | Revisor |
|---|-----|-----------|----------|-----|------|---------|
| 1 | 🔴 CRÍTICO | reconcile.ts:88 | cron asume cursor exclusivo; tras el cambio cuenta 2× el día 1 → doble cobro | `>` → `>=` + guard del invariante | alta | correctness |
| 2 | 🟡 MAYOR | cursor.test.ts:31 | 4 asserts reescritos sin regression test del bug original | test que falle pre-fix | media | tests |
| 3 | 🟢 MENOR | cursor.ts:44 | invariante solo en comentario, sin guard estructural | tipo branded / assert | alta | architecture |

## Cobertura y límites
Lentes: correctness, security, test-integrity, architecture+conventions. Cross-family no activado.
e2e de facturación no incluidos en esta revisión (recomiendo correrlos).

## Intent / decision log
Adjunto: invariante inclusivo/exclusivo + alternativa "exclusión mutua" descartada por coste.

## Dónde está el gate real
El merge lo gatea CI + branch protection, no esta skill. Este informe es un sensor.
```

## Lección del ejemplo

El bug que importaba (**el cron de reconciliación heredando el cambio de invariante**) es exactamente la clase "cambio de significado → código a distancia que rompe": los tests del PR estaban verdes (cubrían el CRUD del cursor, no el cron), y solo un revisor que mide blast-radius + re-audita TODOS los usos lo caza. Heterogeneidad + validación + tiering por blast-radius trabajando juntos.
