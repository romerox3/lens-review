# Metodología — sensor, no veredicto

> La revisión agéntica es un **sensor**, no la decisión de merge. El humano es dueño del merge; el gate vinculante vive en CI. Este es el constraint arquitectónico más importante del sistema.

## El principio

Una skill es **probabilística**: por bien construida que esté, se la puede "convencer" (un prompt, un caso raro, un falso negativo). Un gate determinista —exit-code de un hook, un required status check en CI— **no se puede argumentar**. Osmani lo dice explícito: el gate determinista NO debe ser una skill probabilística.

Por tanto, esta skill **nunca**:
- `gh pr review --approve` ni aprueba de ningún modo.
- `gh pr merge` ni mergea.
- Comenta en el PR **salvo** flag `--comment` explícito (y aun así: comenta hallazgos, no aprueba).
- Sustituye el gate de CI / branch protection.

Y **siempre**:
- Emite hallazgos clasificados (sensor).
- Da un **veredicto recomendado** explícitamente marcado como **no vinculante**.
- Señala dónde está el gate real ([`../references/deterministic-gates.md`](../references/deterministic-gates.md)).

## Por qué importa (datos de Anthropic)

El producto hosted Code Review de Anthropic postea un check run **neutral que nunca bloquea el merge por diseño**; para gatear, parseas tú los conteos de severidad en tu CI. Resultado interno: la revisión sustantiva de PRs pasó del 16% al 54% — pero la **decisión** siguió siendo humana + CI. El sensor sube la cobertura de revisión; no toma la decisión.

## La división de responsabilidades

| Capa | Naturaleza | Rol |
|---|---|---|
| Esta skill (fan-out de revisores) | Probabilística | **Sensor**: encuentra y prioriza problemas |
| Hooks locales | Determinista pero bypassable | Feedback rápido (no es muro) |
| CI + branch protection | Determinista, externo | **Muro**: lo único que puede impedir un merge |
| Humano | Juicio | **Dueño del merge**: decide con el sensor + el muro |

Los hooks locales son bypassables (subagentes, MCP, pipe-mode) y reescribibles por el modelo si puede editarlos → **no son el muro**. El muro está en CI, fuera del alcance del modelo.

## En la práctica

- El informe cierra con "Veredicto recomendado (NO vinculante)" y una línea sobre dónde está el gate.
- Si el usuario pide explícitamente comentar (`--comment`), postea hallazgos inline; **jamás** un `--approve`.
- Si el usuario pide "aprueba y mergea": recuérdale que esta skill es sensor; la aprobación/merge la hace él (o su CI). No lo hagas tú.
