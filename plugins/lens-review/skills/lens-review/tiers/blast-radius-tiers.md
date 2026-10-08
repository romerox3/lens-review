# Tiering por blast-radius

> Dimensiona el fan-out por **lo que el cambio rompe si está mal**, no por su tamaño en líneas. Un one-liner en el path de pagos es Tier 2; un refactor de 800 líneas de tests puede ser Tier 1.

## La matriz

| Tier | El cambio toca… | Revisores | Validación | Extra |
|---|---|---|---|---|
| **0 · Ligero** | config, docs, cosmético, rename/move mecánico, formateo | `lens-review:lens-architecture` (solo convenciones) | — | declarar "cosmético, lentes técnicas no aplican" |
| **1 · Estándar** | feature/fix normal, lógica de dominio, UI, endpoints sin alto riesgo | `lens-review:lens-correctness` + `lens-review:lens-tests` + `lens-review:lens-architecture` | CRÍTICO + MAYOR | — |
| **2 · Full** | ver lista de **dominios de alto riesgo** abajo | + `lens-review:lens-security` | **todo** + perspectiva diversa en CRÍTICO | **intent log obligatorio** + dueño humano nombrado |

## Dominios de alto riesgo → Tier 2 automático

Si **cualquier** path tocado cae en uno de estos, es Tier 2:

- **Pagos / billing**: cargos, suscripciones, cursores de facturación, reconciliación.
- **Auth / sesión**: login, refresh, logout, OAuth, tokens, password.
- **Authz / permisos**: control de acceso, roles, ownership, membership, sharing.
- **Crypto**: cifrado, firma, hashing de secretos, manejo de keys.
- **Migración de DB / cambio de schema**: alteraciones de datos, índices, backfills.
- **Concurrencia / estado distribuido**: locks, transacciones, jobs, schedulers, executors, workers.
- **Borrado / mutación destructiva de datos**: deletes, purges, overwrites masivos.
- **Cambio de significado de un invariante**: un id pasa de global a per-usuario, una unidad/enum/formato cambia (re-auditoría total obligatoria).
- **Superficie de entrada a LLM con texto no confiable** (riesgo prompt-injection).

## Cómo decidir el tier

1. **Mapea los paths tocados a dominios** (`git diff --stat` → clasifica cada archivo).
2. **¿Algún dominio de alto riesgo?** → Tier 2. Fin.
3. **¿Hay lógica de dominio / comportamiento?** → Tier 1.
4. **¿Solo config/docs/cosmético?** → Tier 0.
5. **Blast-radius como modificador**: si un cambio "Tier 1" toca un símbolo con **muchos callers** y cambia su **significado**, súbelo a Tier 2 (el alcance lo hace de alto riesgo aunque el dominio no esté en la lista).
6. **Ante la duda entre dos tiers, sube** (conservador). El coste de un revisor extra es menor que el de un bug en producción.

## Por qué Tier 2 exige intent log y dueño humano

En cambios de alto blast-radius, el reviewer agéntico puede ser "el primer humano-equivalente que ve este código". El **intent/decision log** ([`../templates/intent-decision-log.md`](../templates/intent-decision-log.md)) fuerza a explicitar qué se intentó y qué alternativas se descartaron — sin él, el reviewer hereda código sin contexto y revisa a ciegas. El **dueño humano nombrado** asegura que alguien con juicio firma la decisión de merge (el sensor no decide; ver [`../methodology/sensor-not-verdict.md`](../methodology/sensor-not-verdict.md)).

## Ejemplos

| Cambio | Tier | Por qué |
|---|---|---|
| Bump de versión en `package.json` | 0 | config |
| Añadir un campo a un formulario de UI | 1 | feature normal |
| Cambiar el cálculo del cursor de facturación | 2 | billing |
| One-liner que cambia el filtro de un executor de jobs | 2 | concurrencia + cambio de significado |
| Refactor mecánico que renombra una función con 40 callers | 1 (o 2 si cambia significado) | mucho blast-radius pero sin cambio semántico → 1; si la semántica cambia → 2 |
| Añadir un MCA que ingiere comentarios de PR y los pasa a un LLM con tools | 2 | superficie de prompt-injection |
