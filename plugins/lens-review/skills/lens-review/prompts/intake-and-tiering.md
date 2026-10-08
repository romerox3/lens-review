# Prompt — Intake gate + Tiering (Pasos 1-2)

Prompt ejecutable para arrancar la revisión. **No lances revisores hasta cerrar el intake.**

## Paso 0 — Recopila el contexto del cambio

```bash
git status
git log <base>..HEAD --oneline          # base normalmente dev; para PR: gh pr view <N>
git diff <base>...HEAD --stat            # tamaño y archivos tocados
git diff <base>...HEAD                   # el diff completo (si cabe; si no, por archivo)
```

Para un PR: `gh pr view <N> --json title,body,headRefName,baseRefName` + `gh pr diff <N>`.

> ⚠️ **Three-dot vs dos-puntos**: `git diff <base>...HEAD` (tres puntos) compara contra el **merge-base**, lo correcto para ver "lo que esta rama añade". No uses tres puntos para copiar archivos entre worktrees (oculta commits que `dev` avanzó). Aquí, para revisar, tres puntos es lo correcto.

## Paso 1 — Intake gate

Verifica el mínimo **antes** de gastar revisores. Si falta algo, **pídelo o constrúyelo**; no arranques el fan-out a ciegas.

| Requisito | Cómo obtenerlo | Si falta |
|---|---|---|
| **Intención declarada** | PR body, ticket Linear (`mcp__linear__get_issue`), o pregunta al usuario en 1 frase: "¿qué intenta hacer y por qué?" | BLOQUEO de intake. Sin intención el revisor solo juzga estética. Pídela. |
| **Tests corridos + output** | ¿el usuario los corrió? ¿hay output? Si no, ofrécete a correrlos | Señala que el diff no se ejecutó. En Tier 2 es BLOQUEO; en Tier 0/1 es un MAYOR de intake. |
| **Diff acotado** | `--stat` total de líneas | >~1500 LOC: declara la limitación ("revisión degradada por tamaño; sugiero partir"). Revisa igual. |
| **Intent/decision log** (solo Tier 2) | [`../templates/intent-decision-log.md`](../templates/intent-decision-log.md) | BLOQUEO en Tier 2. Pídelo o constrúyelo con el usuario. |

### Estrategia para diffs grandes (>~1500 LOC)

Un mega-diff (p.ej. 78 archivos / 7500 LOC) NO se cubre entero a la misma profundidad con 4 revisores. Fingir cobertura total es el peor resultado. Adapta como `differential-review`:

| Tamaño | Estrategia |
|---|---|
| < ~1500 LOC | DEEP: todos los archivos a fondo. |
| ~1500–4000 LOC | FOCUSED: cobertura profunda en archivos de alto blast-radius (lógica de dominio, paths sensibles); superficial en el resto (renames, generados, tests mecánicos). |
| > ~4000 LOC | SURGICAL: solo critical paths a fondo; declarar explícitamente qué quedó sin cobertura profunda y sugerir partir el PR. |

En FOCUSED/SURGICAL, pasa a cada revisor la **lista priorizada de archivos** (no los 78) y **declara la degradación** en el informe (sección Cobertura y límites). El intake decide la lista; la síntesis la reporta.

**Salida del intake**:
- ✅ Intake completo → continúa a Tiering (con estrategia DEEP/FOCUSED/SURGICAL según tamaño).
- ⛔ Intake incompleto → enumera qué falta, pídelo, **para aquí**. No revises.

## Paso 2 — Tiering por blast-radius

Clasifica con la matriz de [`../tiers/blast-radius-tiers.md`](../tiers/blast-radius-tiers.md). Criterio: **¿qué rompe si está mal?**, no cuántas líneas tiene.

Procedimiento:

1. **Lista los paths tocados** y mapéalos a dominios (auth, payments, crypto, DB migration, permisos, borrado, concurrencia, config, docs, tests, UI…).
2. **Calcula blast-radius**: para los símbolos exportados que cambian, ¿cuántos callers tienen? (`grep -rn "<symbol>"`). Muchos callers + cambio de significado = sube de tier.
3. **Asigna tier**:
   - Cualquier path de la lista "alto riesgo" (auth/payments/crypto/migración/permisos/borrado/concurrencia) → **Tier 2**.
   - Lógica de dominio normal, feature/fix → **Tier 1**.
   - Solo config/docs/cosmético/rename mecánico → **Tier 0**.
   - Si dudas entre dos tiers, **sube** (conservador).
4. **Declara el tier y el set de revisores** explícitamente antes del fan-out:

```
Tier asignado: <0|1|2>
Razón: <dominio + blast-radius>
Revisores: <lista de agentType>
Intent log requerido: <sí solo en Tier 2>
```

Continúa a [`dispatch.md`](dispatch.md).
