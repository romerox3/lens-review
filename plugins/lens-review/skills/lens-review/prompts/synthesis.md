# Prompt — Síntesis (Paso 5)

Consolida los hallazgos **confirmados** en un informe advisory. No es concatenar los outputs de los revisores: es dedupe + rank + veredicto recomendado + señalar dónde está el gate real.

## 1. Dedupe

- Hallazgos en el **mismo file:line** desde lentes distintas → **uno solo**, atribuido a la lente más específica; menciona la otra como contexto.
- Si dos revisores independientes coincidieron → márcalo como **alta confianza** (raro, dada la heterogeneidad; valioso).
- No fusiones hallazgos distintos del mismo archivo solo por estar cerca — cada problema real es su propia entrada.

## 2. Rank por severidad

Ordena CRÍTICO → MAYOR → MENOR → NIT. Dentro de cada nivel, por confianza (alta primero). Usa la rúbrica y etiquetas UI de [`../templates/review-report.md`](../templates/review-report.md).

## 3. Intent/decision log (Tier 2)

En Tier 2, el informe incluye (o referencia) el [`../templates/intent-decision-log.md`](../templates/intent-decision-log.md): qué se intentó, qué alternativas se rechazaron y por qué, decisiones no obvias. Si el autor no lo aportó, recógelo del PR/commits/conversación y constrúyelo — su ausencia ya es un hallazgo (el reviewer humano hereda código sin contexto).

## 4. Informe advisory

Rellena [`../templates/review-report.md`](../templates/review-report.md):

- **Sello "Revisado"**: captura `base@<sha8>...head@<sha8>` (de `git rev-parse`), tamaño del diff (`--shortstat`) y fecha-hora. **Crítico**: sin el SHA el informe caduca en silencio — un lector no sabe si los fixes ya se aplicaron. (Caso real: una review marcó "resolver H2/H14"; el autor los arregló en el working tree justo después → el informe quedó obsoleto sin forma de saberlo.)
- **TL;DR (6-8 líneas autosuficientes)**: sello Revisado, tier, revisores+modelo, línea de heterogeneidad, conteo (confirmados de brutos; refutados), veredicto recomendado.
- **Tabla de hallazgos** confirmados con TODAS las columnas: severidad · file:line · problema · fix · **confianza** · **revisor**. No recortes columnas.
- **Heterogeneidad observada**: nº de hallazgos + cuántas convergencias ★ entre revisores. Es la métrica que valida la tesis de la skill; siempre va.
- **Refutados/inciertos** (apéndice breve): qué se descartó y por qué — transparencia.
- **Cobertura y límites** (SIEMPRE): qué lentes corrieron, qué quedó fuera. **Si el diff supera ~1500 LOC** declara degradación explícita + qué archivos tuvieron cobertura profunda vs superficial (un mega-diff Tier 2 no se cubre entero con 4 revisores; decirlo es obligatorio, callarlo finge cobertura total).
- **Veredicto recomendado (NO vinculante)**: "Listo para merge" sii zero CRÍTICO + zero MAYOR confirmados.
- **Intent log + dueño humano** (Tier 2): incluir/enlazar el intent log y nombrar al dueño del merge.
- **Dónde está el gate real**: línea a [`../references/deterministic-gates.md`](../references/deterministic-gates.md) — el merge lo gatea CI/branch protection, no esta skill.

## 5. Output

```
Informe → conversación (siempre) + opcional /tmp/{feature}-lens-review.md
Hallazgos → tool ReportFindings, si está disponible (abajo)
Intent log (Tier 2) → sección del informe o /tmp/{feature}-intent-log.md
```

**Nunca** `gh pr review --approve`, `gh pr merge`, ni comentar en el PR — salvo `--comment` explícito (reglas abajo). Ver [`../methodology/sensor-not-verdict.md`](../methodology/sensor-not-verdict.md).

### ReportFindings (si la tool está disponible)

Emite los hallazgos con **una sola llamada**, rankeados por severidad (más grave primero):

- Mapeo de veredicto del validador: CONFIRMADO → `verdict: "CONFIRMED"` · INCIERTO / NO_VALIDADO → `verdict: "PLAUSIBLE"`. Los REFUTADOS **no** se emiten (van solo al apéndice de transparencia del informe).
- `category` = slug de la lente (`correctness`, `security-injection`, `test-integrity`, `architecture`); `file`/`line` del hallazgo; `summary` = el problema en una frase; `failure_scenario` = el escenario concreto que construyó el validador; `level` = effort de la review (`high`).
- Cuando la tool renderiza los hallazgos, **no repitas la tabla completa como texto** en conversación — el informe conserva TL;DR, sello Revisado, heterogeneidad, cobertura/límites, funnel y veredicto; la tabla íntegra vive en el `/tmp` scratch. Sin la tool, la tabla va en el informe como siempre.

### Publicación en el PR — SOLO con `--comment`

Tres reglas, importadas del qa-swarm de referencia (evitan los males conocidos: comentarios que parecen humanos, y summaries apilándose en cada re-run):

1. **Header bot-identifier en TODO lo posteado** (inline, summary, cualquier cosa) — los comentarios salen por la cuenta del usuario y el header es lo único que los distingue de texto humano (y lo que `review-triage` usa para clasificarlos como automatizados):

   ```markdown
   > [!NOTE]
   > 🤖 Automated comment by **lens-review** — not written by a human
   ```

2. **Inline comments como UNA review**, no N comentarios sueltos:

   ```bash
   gh api repos/{owner}/{repo}/pulls/{n}/reviews --method POST \
     -f event="COMMENT" -f commit_id="{HEAD_SHA}" \
     -f body="lens-review — hallazgos inline; summary en el comentario sticky." \
     -f 'comments[]={path: "<file>", line: <line>, body: "<header + **[<lente>]** <emoji> <SEVERIDAD> + problema/fix/escenario>"}'
   ```

   Emojis: 🔴 CRÍTICO · 🟡 MAYOR · 🟢 MENOR · 💡 NIT. Convergencias: `**[convergente: <lente1> + <lente2>]**`. `event` siempre `COMMENT` — jamás APPROVE ni REQUEST_CHANGES. Fallback si la review API se complica con muchos comments: comentarios individuales vía `/pulls/{n}/comments`.

3. **UN summary sticky por PR, upserted** — marcado `<!-- lens-review-summary -->`. Buscar el existente y actualizar in place; NUNCA apilar uno nuevo por re-run:

   ```bash
   gh api "repos/{owner}/{repo}/issues/{n}/comments" --paginate \
     --jq '[.[] | select(.body | contains("<!-- lens-review-summary -->"))][0].id'
   # existe → PATCH repos/{owner}/{repo}/issues/comments/{id} · no existe → gh pr comment {n} --body-file ...
   ```

   Estructura: marcador + header bot + `## Veredicto <emoji> (ronda <N> @ <sha8>)` + hallazgos clave de ESTA ronda + tabla de una línea por lente + `<details>` con las rondas previas colapsadas a una línea cada una (`ronda N @ sha — veredicto: disposición`).

## Checklist antes de emitir (obligatorio — no omitir ninguna)

El synthesizer tiende a producir un informe "más limpio" soltando secciones load-bearing. Antes de devolver, verifica:

- [ ] Sello **Revisado** con base/head SHA + tamaño + fecha.
- [ ] Tabla con columnas **Conf** y **Revisor** (no recortadas).
- [ ] Línea **Heterogeneidad** (hallazgos + convergencias ★).
- [ ] Sección **Cobertura y límites**; si diff >~1500 LOC, degradación declarada.
- [ ] Si **Tier 2**: **Intent log** presente + **dueño humano** nombrado.
- [ ] Conteo bruto → confirmados → refutados (el funnel visible).
- [ ] Veredicto marcado **NO vinculante** + puntero al gate de CI.

Si falta cualquiera, complétala antes de emitir. Un informe "bonito" que omite la cobertura o el SHA es peor que uno feo completo.

## Anti-patterns de síntesis

- Concatenar los 4 outputs sin dedupe ni rank. El usuario quiere UNA lista priorizada, no 4 informes.
- Esconder lo no cubierto. Si una lente no corrió o el diff era enorme, dilo en "Cobertura y límites".
- **Omitir el SHA revisado.** El informe caduca en silencio; nadie sabe si los fixes ya entraron.
- **Soltar la atribución por-revisor** "para que quede más limpio". Sin ella no se puede medir la heterogeneidad.
- Subir la severidad de un INCIERTO a MAYOR para "que se note". Inciertos van como MENOR no confirmado.
- Cerrar con "Listo para merge ✅" como si fuera un veredicto. Es recomendación; el humano + CI deciden.
