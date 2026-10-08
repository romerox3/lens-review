# Template — Intent / decision log (Tier 2)

> Obligatorio en Tier 2. Fuerza a explicitar **qué se intentó, qué se descartó y por qué**, para que el reviewer (humano o agéntico) no herede código de alto riesgo sin contexto. Ataca el problema "el reviewer es el primer humano-equivalente que ve este código".

## Cuándo

- **Tier 2** (alto blast-radius): el intake lo exige. Si el autor no lo aportó, recógelo del PR/commits/conversación y constrúyelo con él. Su **ausencia ya es un hallazgo**.
- Tier 0/1: opcional. Útil si hay una decisión no obvia.

## Formato

```markdown
# Intent / decision log — {feature/PR}

## Intención
{1-3 frases: qué intenta hacer este cambio y por qué. El problema que resuelve, no la implementación.}

## Enfoque elegido
{El approach que se implementó, en 2-4 frases.}

## Alternativas consideradas y descartadas
| Alternativa | Por qué se descartó |
|---|---|
| {opción A} | {razón concreta: coste, riesgo, no encaja con X} |
| {opción B} | {razón} |

## Decisiones no obvias
- {Decisión}: {por qué se tomó. Algo que un reviewer preguntaría "¿por qué así y no asá?"}
- {Trade-off aceptado conscientemente y su justificación}

## Invariantes que el cambio asume / debe preservar
- {p.ej. "el id de job es per-usuario; el executor DEBE filtrar por (userId, id)"}
- {invariantes que, si se rompen, causan el bug de alto blast-radius}

## Dueño humano del merge
{Nombre de quien firma la decisión de merge. El sensor no decide; un humano con juicio sí.}

## Verificación realizada
{Qué se ejecutó: tests + resultado real, smoke, etc. Solo lo HECHO, no aspiraciones.}
```

## Reglas

- **Honesto, no aspiracional**: "Verificación realizada" lista solo lo que se ejecutó de verdad (conteos reales). Lo que falta por verificar va a un apartado aparte o al informe.
- **Las alternativas descartadas son el valor**: un log que solo dice qué se hizo, sin qué se rechazó y por qué, no ayuda al reviewer. El "por qué no la otra opción" es lo que evita re-litigar.
- **Los invariantes son críticos en Tier 2**: explicitar el invariante que el cambio asume permite al reviewer (y a los revisores agénticos) verificar que TODOS los usos lo respetan — incluidos los paths de fondo.
- Output a `/tmp/{feature}-intent-log.md` o como sección del informe advisory. No va a `docs/` (es scratch de revisión, no documentación histórica).
