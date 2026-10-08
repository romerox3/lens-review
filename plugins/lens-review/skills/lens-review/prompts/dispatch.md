# Prompt — Fan-out (Paso 3)

Lanza los revisores del tier **en paralelo**, cada uno con contexto fresco. Reutiliza el patrón de `dispatching-parallel-agents`: una tarea por dominio independiente, sin estado compartido, todas concurrentes.

## Regla de oro

Cada revisor es **independiente y ciego a los demás**. No heredan tu sesión ni el output de otro revisor. Tú construyes exactamente lo que cada uno necesita. Esto preserva su foco y tu contexto de coordinación — y es lo que hace que sus hallazgos sean **disjuntos** (heterogeneidad real).

## Cómo lanzar

En una **sola** respuesta, emite N llamadas a la herramienta Agent (una por revisor del tier), para que corran concurrentemente. Construye cada `prompt` desde [`../templates/subagent-task.md`](../templates/subagent-task.md).

Dos formas de despachar, equivalentes en resultado:

**A) Default robusto — funciona siempre** (úsalo si los `lens-review:lens-*` no están en la lista de agentType de la sesión): `agentType="general-purpose"` + el parámetro `model` (de aquí sale la heterogeneidad Opus/Sonnet) + la **lente embebida** en el prompt.

```
Agent(agentType="general-purpose", model="fable",  prompt=<subagent-task: lente correctness>)
Agent(agentType="general-purpose", model="fable",  prompt=<subagent-task: lente security>)        # Tier 2
Agent(agentType="general-purpose", model="opus",   prompt=<subagent-task: lente test-integrity>)
Agent(agentType="general-purpose", model="sonnet", prompt=<subagent-task: lente architecture+conventions>)
// todas corren a la vez
```

> Si el harness rechaza `model="fable"` (Claude Code antiguo), usa `"opus"` para ese agente — mismo split que el qa-swarm de referencia: el modelo más fuerte para los revisores de profundidad técnica, fallback un tier abajo.

**B) Preferido cuando estén disponibles** (los `lens-review:lens-*` del plugin ya traen system-prompt + modelo fijado):

```
Agent(agentType="lens-review:lens-correctness",  prompt=<subagent-task: lente correctness>)
Agent(agentType="lens-review:lens-tests",        prompt=<subagent-task: lente test-integrity>)
Agent(agentType="lens-review:lens-architecture", prompt=<subagent-task: lente architecture+conventions>)
```

> Comprueba la lista de agentType disponibles. Si ves `lens-review:lens-*`, usa B. Si aún no aparecen (recién creados, Claude Code los registra con un pequeño retardo — **sin reinicio**), usa A: el contenido del prompt es el mismo (el template embebe la lente), así que el resultado es equivalente.

Por tier: **Tier 0** lanza solo el revisor de arquitectura; **Tier 1** correctness + tests + architecture; **Tier 2** añade security.

## Qué va en el `prompt` de cada revisor

Rellena el template con:

- **Scope**: "Revisa SOLO desde la lente <X>. No te metas en otras dimensiones."
- **Intención del cambio**: el statement del intake (qué intenta y por qué). Sin esto, el revisor juzga estética.
- **Refs**: `base` (merge-base SHA), `head` (HEAD SHA), comando exacto para ver el diff (`git diff <base>...<head>`), lista de archivos.
- **Lente embebida o referenciada**: pásale la ruta de su archivo de lente o el contenido clave. (Los subagentes NO heredan la disciplina de esta skill — su metodología va en su propio prompt + su system prompt de agent.)
- **Formato de salida**: el bloque canónico de hallazgo (abajo), para que la síntesis pueda parsear.
- **Constraint**: "Read-only. No edites archivos. Devuelve solo hallazgos."

## Formato de hallazgo (que cada revisor debe devolver)

Cada revisor devuelve una lista de hallazgos en este formato exacto (facilita dedupe + validación):

```
SEVERIDAD · lente · file:line — <descripción concreta del problema> · <fix sugerido> · confianza: alta|media|baja
```

Ejemplo:
```
CRÍTICO · correctness · packages/backend/src/handlers/pay.ts:142 — POST /payments sin idempotency key; un retry tras timeout duplica el cargo · añadir Idempotency-Key + tabla con TTL 24h · confianza: alta
```

Si una lente no encuentra nada: el revisor devuelve `lente <X>: cero hallazgos`. Es legítimo y esperado.

## Orquestación vía Workflow — preferida en Tier 1/2 si la tool está disponible

El pipeline de esta skill es el caso canónico de la tool `Workflow`. Ventajas sobre el despacho manual: **pipeline sin barrera** (los hallazgos de un revisor entran a validación mientras los demás aún revisan — hoy el despacho manual espera a todos), **salida estructurada por schema** (adiós al parseo del bloque de texto canónico), y journal reanudable. Invocar esta skill cuenta como opt-in del usuario para Workflow. Tier 0 (1 revisor) no la amerita.

Construye `args` en el main loop (intake ya hecho): `{ intent, base, head, validateAll: <tier===2>, reviewers: [{key, model, task}] }` donde `task` es el subagent-task ya rellenado con la lente embebida, y `model` sigue la tabla (fable/fable/opus/sonnet — fallback opus si el runtime rechaza fable).

```js
export const meta = {
  name: 'lens-review',
  description: 'Fan-out heterogéneo de revisores + validación adversarial per-finding',
  phases: [{ title: 'Review' }, { title: 'Verify' }],
}
const FINDINGS = { type: 'object', required: ['findings'], properties: { findings: { type: 'array', items: {
  type: 'object', required: ['severidad', 'file', 'line', 'problema', 'fix', 'confianza'],
  properties: { severidad: { enum: ['CRITICO', 'MAYOR', 'MENOR', 'NIT'] }, file: { type: 'string' },
    line: { type: 'integer' }, problema: { type: 'string' }, fix: { type: 'string' },
    confianza: { enum: ['alta', 'media', 'baja'] } } } } } }
const VERDICT = { type: 'object', required: ['veredicto', 'razon'], properties: {
  veredicto: { enum: ['CONFIRMADO', 'REFUTADO', 'INCIERTO'] }, razon: { type: 'string' },
  severidad_ajustada: { type: 'string' }, escenario: { type: 'string' } } }

const refute = (f, r) => `Vas a INTENTAR REFUTAR un hallazgo de code review. Sesgo default: REFUTADO si hay duda.
Hallazgo: ${f.severidad} · ${r.key} · ${f.file}:${f.line} — ${f.problema} · fix propuesto: ${f.fix}
Intención del cambio: ${args.intent}
Refs: git diff ${args.base}...${args.head}
No confíes en el hallazgo: lee el código real (la función entera, callers, validaciones aguas arriba).
¿Es REAL y ALCANZABLE con inputs plausibles? Construye el escenario concreto que lo dispara; si no puedes, REFUTADO.`

const perReviewer = await pipeline(
  args.reviewers,
  r => agent(r.task, { label: `review:${r.key}`, phase: 'Review', model: r.model, schema: FINDINGS }),
  async (rev, r) => {
    const found = (rev?.findings ?? []).map(f => ({ ...f, lente: r.key }))
    const toValidate = found.filter(f => args.validateAll || ['CRITICO', 'MAYOR'].includes(f.severidad))
    const sinValidar = found.filter(f => !toValidate.includes(f))
      .map(f => ({ ...f, verdict: { veredicto: 'NO_VALIDADO', razon: 'tier 0/1: MENOR/NIT pasan marcados' } }))
    const validated = await parallel(toValidate.map(f => () =>
      agent(refute(f, r), { label: `verify:${f.file}:${f.line}`, phase: 'Verify', model: r.model, effort: 'high', schema: VERDICT })
        .then(v => ({ ...f, verdict: v }))))
    return [...validated.filter(Boolean), ...sinValidar]
  }
)
const all = perReviewer.filter(Boolean).flat()
return {
  confirmados: all.filter(f => f.verdict.veredicto === 'CONFIRMADO'),
  inciertos: all.filter(f => f.verdict.veredicto === 'INCIERTO'),
  no_validados: all.filter(f => f.verdict.veredicto === 'NO_VALIDADO'),
  refutados: all.filter(f => f.verdict.veredicto === 'REFUTADO')
    .map(f => ({ file: f.file, line: f.line, lente: f.lente, razon: f.verdict.razon })),
}
```

El resultado alimenta la síntesis directamente (dedupe + rank sobre `confirmados`/`inciertos`; `refutados` al apéndice de transparencia; `no_validados` como "no confirmado"). El funnel bruto→confirmados→refutados del informe sale de los conteos.

## Tras el fan-out

Cuando todos los revisores devuelven:

1. **Lee cada lista de hallazgos** (no las fusiones todavía).
2. **No confíes en ellas aún** — pasan por validación per-finding ([`validation.md`](validation.md)).
3. Si dos revisores **coinciden** en un file:line, márcalo: coincidencia = alta confianza (raro y valioso, dada la heterogeneidad).

Continúa a [`validation.md`](validation.md).
