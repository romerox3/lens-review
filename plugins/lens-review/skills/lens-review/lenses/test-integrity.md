# Lente — test-integrity

> Escrutinio del **test-diff** con más cuidado que el código. La lente del revisor `lens-review:lens-tests` (Sonnet). Principio de Osmani: los agentes "arreglan" tests reescribiendo los asserts para que cuadren con el comportamiento roto. Un diff que reescribe muchos asserts es un **red flag**, no una mejora.

## Por qué los tests merecen MÁS escrutinio que el código

El código nuevo se revisa porque puede estar mal. El test nuevo se revisa porque **da una falsa sensación de seguridad si está mal** — un test verde que no muerde es peor que no tener test: el CI pasa y nadie mira. Coverage ≠ calidad; la métrica real es **mutation score** (¿el test falla si rompes el código?).

## Ejes a recorrer

### 1. Asserts reescritos (el red flag central)
- ¿El diff **modifica asserts existentes** en vez de solo añadir tests? ¿Por qué cambió el valor esperado?
- ¿El nuevo valor esperado refleja el comportamiento **correcto**, o el comportamiento **actual (posiblemente roto)** del código que se acaba de cambiar?
- Patrón sospechoso: el mismo PR cambia lógica Y reescribe los asserts que la cubrían, dejándolos "verdes" por construcción. ¿Quién garantiza que el nuevo esperado es el correcto?

### 2. ¿El test puede fallar? (mutation thinking)
- Para cada test nuevo: si rompieras el código a propósito (negar una condición, cambiar un `+` por `-`), ¿el test se pondría rojo? Si no, no muerde.
- Asserts triviales (`expect(result).toBeDefined()`, `expect(fn).toHaveBeenCalled()`) que pasan casi siempre → falsa cobertura.
- Afirmar payload **exacto** (`toEqual`) vs laxo (`toMatchObject`, "se llamó"). Lo laxo deja pasar regresiones.

### 3. Cobertura del path que de verdad falla
- ¿El test toca el path real que puede romperse, o una capa más arriba (falsa confianza)?
- Incidente canónico (Teros scheduler): los tests cubrían el CRUD, no el executor — el bug vivía 10 días en el path no testeado.
- Bug fix sin **regression test** que reproduzca el fallo con el boundary/escenario real → hallazgo.

### 4. Mocks fieles al boundary
- ¿El mock replica el comportamiento REAL del boundary? (un mock de `fetch` con `Object.entries` en vez de `new Headers()` no reproduce el combine de cabeceras → bug pasa verde).
- ¿Se afirma el payload que SALE al boundary, no solo "se llamó"?

### 5. Fronteras enumeradas (CORRECT) y errores (RIGHT-BICEP)
- ¿Se cubren null/0/""/vacío/1/N/límites/expiry/concurrencia, no solo el happy path?
- ¿Hay test del camino de error y del boundary, no solo del éxito?

### 6. Determinismo
- ¿Tests flaky por timing (`setTimeout` arbitrario en vez de espera por evento)?
- ¿Dependencia de orden entre tests (estado global, `mock.module` que no se resetea)?

## Hallazgo típico

```
MAYOR · test-integrity · tests/unit/billing/cursor.test.ts:40 — este PR cambia la lógica del cursor Y reescribe 6 asserts para que esperen el nuevo valor; no hay evidencia de que el nuevo esperado sea el correcto y no el del bug. Romper la lógica a propósito no pone rojo ningún test (no muerden) · añadir un test que falle ANTES del fix (reproduzca el bug) y afirme el payload exacto · confianza: media
```

## Anti-patterns

- Reescribir asserts para "poner verde" sin justificar el nuevo esperado.
- `expect(x).toBeTruthy()` / `toBeDefined()` como única aserción.
- Mock que no reproduce el boundary real → el bug que debía cazar pasa verde.
- Fix sin regression test.
- "Coverage subió a 90%" como prueba de calidad. La métrica es mutation score.

## Disciplina

- Para cada assert reescrito, exige la justificación del nuevo valor esperado.
- Recomienda mutation testing donde coverage no basta (paquetes con runner soportado).
- Si el diff no corrió los tests (intake), súbelo como hallazgo: tests no ejecutados no prueban nada.
