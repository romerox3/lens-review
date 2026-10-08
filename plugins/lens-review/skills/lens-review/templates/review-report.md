# Template — Informe advisory

Formato del informe final que emite la síntesis. Advisory: recomienda, no decide. ≤ lo necesario; TL;DR autosuficiente.

```markdown
# Revisión agéntica — {feature/PR}

## TL;DR
- **Revisado**: `{base@sha8}...{head@sha8}` · {N} archivos / +{X}−{Y} · {fecha-hora}
  — informe válido para ese HEAD; si el árbol avanzó, re-ejecutar (los fixes pueden estar ya aplicados).
- **Tier**: {0|1|2} ({razón: dominio + blast-radius})
- **Revisores**: {lista que corrió + modelo} ({N} en paralelo) · **Heterogeneidad**: {nº hallazgos, X convergencias ★}
- **Hallazgos confirmados**: 🔴 {nCRÍTICO} · 🟡 {nMAYOR} · 🟢 {nMENOR} · 💡 {nNIT} (de {bruto} brutos; {refutados} refutados)
- **Veredicto recomendado (NO vinculante)**: {Listo para merge | Resolver antes de merge}
  — el gate vinculante es CI, no esta skill.

## Hallazgos

| # | Sev | file:line | Problema | Fix sugerido | Conf | Revisor |
|---|-----|-----------|----------|--------------|------|---------|
| 1 | 🔴 CRÍTICO | pay.ts:142 | POST /payments sin idempotency key → retry duplica cargo | Idempotency-Key + tabla TTL 24h | alta | correctness |
| 2 | 🟡 MAYOR | auth.ts:88 | … | … | alta | security |
| … | | | | | | |

> Coincidencias entre revisores marcadas con ★ (alta confianza por convergencia independiente).

## Cobertura y límites
- Lentes ejecutadas: {lista}.
- Fuera de alcance / no cubierto: {p.ej. "e2e no corridos", "diff de 1800 LOC — revisión degradada, sugiero partir"}.
- Sé honesto: lo que no se miró, se declara aquí.

## Refutados / inciertos (apéndice, opcional)
- {hallazgo} → REFUTADO: {razón con file:line}.
- {hallazgo} → INCIERTO: requiere ojo humano.

## Intent / decision log (solo Tier 2)
{Incluir o enlazar templates/intent-decision-log.md — qué se intentó, alternativas rechazadas, por qué.}

## Dónde está el gate real
El merge lo gatea CI + branch protection, no esta skill. Esta revisión es un **sensor**.
Ver references/deterministic-gates.md. El humano es dueño del merge.
```

## Secciones obligatorias (NO omitibles)

El synthesizer tiende a "limpiar" el informe y soltar secciones load-bearing. No las omitas:

| Sección | Cuándo es obligatoria |
|---|---|
| Línea **Revisado** (base/head SHA + tamaño + fecha) | SIEMPRE (sin SHA el informe caduca en silencio) |
| Tabla con columnas **Conf** y **Revisor** | SIEMPRE (la atribución por-revisor sostiene la tesis de heterogeneidad) |
| **Cobertura y límites** | SIEMPRE — y si el diff supera ~1500 LOC, declarar degradación + qué archivos tuvieron cobertura profunda vs superficial |
| **Heterogeneidad observada** (en TL;DR) | SIEMPRE (nº hallazgos + convergencias ★) |
| **Intent / decision log** | Si Tier 2 |
| **Dueño humano** del merge | Si Tier 2 |

## Reglas del informe

- **TL;DR-first**: las primeras 6-8 líneas bastan para decidir. Un lector entra y sabe el estado.
- **Tabla > prosa** en hallazgos; usa TODAS las columnas (incl. Conf + Revisor) — no recortes el formato.
- **Conteos reales**, no aspiracionales. Si una lente dio 0, el conteo lo refleja.
- **Veredicto = recomendación**, nunca "✅ aprobado". El gate es CI.
- **Transparencia de lo descartado**: el apéndice de refutados construye confianza (el usuario ve que filtras, no que inventas).
- **Sin inflado**: cero NITs inventados para "parecer exhaustivo".
