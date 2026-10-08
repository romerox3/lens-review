# Dónde vive el muro determinista

> Esta skill es un **sensor** (probabilística, "convencible"). El **muro** que de verdad impide un merge es determinista y vive en CI, no aquí. Este documento explica la división y qué hay hoy en Teros. Ver también [`../methodology/sensor-not-verdict.md`](../methodology/sensor-not-verdict.md).

## Por qué el gate NO puede ser esta skill

Una skill es un LLM con instrucciones: por buena que sea, un caso raro o un falso negativo la pasa. Un **required status check** en CI o un **exit-code 2** de un hook **no se argumenta** — fallan o pasan por construcción. Osmani es explícito: el gate determinista no debe ser una skill probabilística.

## Las capas (de menos a más vinculante)

| Capa | Determinista | Bypassable | Rol |
|---|---|---|---|
| `lens-review` (esta skill) | No | — | **Sensor**: encuentra y prioriza |
| Hooks locales de Claude Code | Sí | Sí (subagentes, MCP, pipe-mode; reescribibles por el modelo) | Feedback rápido, no muro |
| Git hooks (`core.hooksPath`) | Sí | Sí | En Teros están **desactivados** (`core.hooksPath=/dev/null`) |
| **CI + branch protection** | Sí | No (fuera del alcance del modelo) | **El muro**: lo único que impide un merge |
| Humano | Juicio | — | **Dueño del merge** |

## Estado actual en Teros (verificado)

El proyecto delega los gates a CI, no a hooks locales (git hooks off). Los workflows en `teros-private/.github/workflows/`:

- **`ci.yml`** — corre de verdad: `bun test` (unit + integration del backend), `tsc` (typecheck frontend + backend), `test:render` (Vitest), y `e2e` (Playwright) **solo en push** (red de seguridad post-merge, no gate por-commit).
- **`security.yml`** — `gitleaks` (secrets, falla el PR) + `osv-pr-diff` (vulns nuevas en deps). `sbom` informativo.
- **`deploy-prod.yml`** — health checks **post-deploy** (no gatean pre-merge).

> El comentario histórico "ci.yml no corre casi nada" del CLAUDE.md está **desactualizado**: hoy ci.yml sí corre tests/tsc/e2e.

## El gap (recomendación, no la toca esta skill)

**No hay branch protection rules** configuradas en GitHub. Es decir: los jobs de `ci.yml`/`security.yml` corren, pero **nada impide mergear con ellos en rojo** — el muro existe pero no está cableado a la puerta. Para cerrar el gap (fuera del alcance de esta skill, requiere acción en GitHub):

1. En `Settings → Branches → Branch protection rules` para `dev` y `main`:
   - Required status checks: `ci / test`, `ci / frontend`, `ci / backend-types`, `security / gitleaks`, `security / osv-pr-diff`.
   - Require PR before merging + al menos 1 review humano.
2. Opcional: parsear los conteos de severidad de un informe de revisión (el de esta skill o el producto hosted Code Review) en un job de CI que falle si hay CRÍTICO/MAYOR — convirtiendo el sensor en input de un gate determinista, sin que el sensor decida.

## Cómo encaja esta skill con el muro

- La skill produce el **informe advisory** (sensor): qué está mal, priorizado.
- El **humano** lee el informe y decide.
- El **CI** (cuando branch protection esté cableado) impide el merge si los checks fallan.
- La skill **nunca** sustituye, simula ni "aprueba por" el muro. Si alguien pide que la skill apruebe/mergee, la respuesta es: el gate es CI, la decisión es del humano.
