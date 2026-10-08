# Lente — correctness-and-bugs

> Bugs de lógica, edge cases, manejo de errores, contratos rotos. La lente del revisor `lens-review:lens-correctness` (Opus). Pregunta central: **¿hace el código lo que la intención dice, en todos los caminos?**

## Ejes a recorrer

### 1. Lógica vs intención
- ¿El código implementa lo que la intención declara — ni más, ni menos, ni distinto?
- ¿Hay off-by-one, condiciones invertidas, `&&`/`||` confundidos, negaciones mal puestas?
- ¿Los early-return / `continue` / `break` salen en el punto correcto?
- ¿El happy path es correcto? (a veces el bug está ahí, no en el edge case)

### 2. Edge cases del input
- `null` / `undefined` / vacío (`""`, `[]`, `{}`) / `0` / `false` legítimos: ¿se distinguen de "ausente"?
- Truthiness: `if (x)` cuando `x` puede ser `0`/`""`/`false` válidos → usar `x != null`.
- Límites: tamaños no acotados, rangos numéricos (negativo donde se espera positivo, overflow), encoding (UTF-8 multibyte, surrogates).
- Boundary validation al entry point: ¿se valida shape/tipo donde el dato entra al sistema?

### 3. Errores upstream
- Cada call externa (DB/API/cache/FS): ¿try/catch explícito? ¿timeout configurado (no el default hostil)?
- ¿Se distingue "no data" de "error obteniendo data"? (confundirlos produce bugs sutiles)
- ¿Fail-open o fail-closed? ¿Es decisión consciente y correcta para el caso?
- `catch (e) { log(e) }` sin rethrow ni decisión → error tragado.

### 4. Concurrencia e idempotencia
- Patrón "leer-decidir-escribir" sin atomicidad → race window.
- Estado compartido sin lock/transacción/atomic/CAS.
- Mutaciones de estado externo: ¿idempotentes naturalmente o necesitan idempotency key? Retries que duplican side-effects.
- Handlers async re-entrantes; timers que disparan durante la operación principal.

### 5. Estados parciales tras fallo
- Operación multi-paso que falla a la mitad: ¿estado consistente o corrupto?
- Multi-recurso sin transacción: ¿compensación/saga/outbox? ¿"intent" persistido antes de ejecutar?

### 6. Cambio de significado (re-auditoría)
- Si el diff **cambia el significado** de una clave/campo/id/unidad/enum/formato: ¿se re-auditaron **todos** los usos, incluidos paths de fondo (workers, crons, executors, migraciones)?
- Tests verdes ≠ cubierto: ¿el test toca el path que de verdad puede fallar, o una capa más arriba?

## Hallazgo típico

```
CRÍTICO · correctness · packages/backend/src/scheduler/executor.ts:88 — el executor filtra jobs solo por `id`, pero el id pasó a ser per-usuario en este diff; dos usuarios con el mismo id se pisan (cross-update) · scopear el filtro por (userId, id) · confianza: alta
```

## Anti-patterns que delatan bugs

- "En desarrollo nunca pasó null" — producción ve inputs que dev no vio.
- `await x; await y; await z` sin transacción ni rollback → estados intermedios corruptos si `y` falla.
- "Si reintentamos 3 veces es seguro" sin idempotency key → duplica.
- `setTimeout`/`setInterval` sin clear en cleanup → handler + memory leak.
- Cambiar el significado de un id/campo y re-auditar solo la superficie obvia (el CRUD), no el executor/worker.

## Disciplina

- Lee el código REAL alrededor del hunk (función entera + callers), no solo el diff.
- Para cada hallazgo, construye el escenario concreto (inputs + secuencia). Si no se puede construir, baja confianza.
- Para comportamiento de librería/API no trivial: verifica con context7/WebSearch, no asumas.
