# Lente — conventions-claudemd

> Cumplimiento de las convenciones del repo (CLAUDE.md / REVIEW.md). Segunda lente del revisor `lens-review:lens-architecture` (Sonnet). Pregunta central: **¿respeta el cambio las reglas que este repo ya declaró por escrito?**

## Cómo cargar las reglas

1. **Lee el `CLAUDE.md`** del repo (raíz + el del subdirectorio tocado, si existe). Es la fuente de verdad de convenciones.
2. **Lee `REVIEW.md`** en la raíz si existe (reglas solo-de-revisión; el producto hosted Code Review de Anthropic las trata así).
3. Trata las violaciones de convención como **MENOR/NIT** por defecto (nivel "nit"), **salvo** que CLAUDE.md marque la regla como invariante duro (entonces sube a MAYOR).

## Ejes a recorrer

### 1. Naming y estructura
- IDs con el prefijo correcto (en Teros: `<prefix>_<16hex>`, prefixes de `core/src/ids.ts`).
- WS actions `domain.verb`; MCAs `mca.<provider>`; files kebab-case; components PascalCase; hooks `use*`; stores `*Store`.
- Conventional commits (`feat:`, `fix:`, `refactor:`…), título/body en el idioma que pida el repo.

### 2. Reglas de arquitectura declaradas
- Principios del repo (en Teros: No Fallbacks, Fail Fast, Strict Interfaces, DB Writes Are Contracts, Workspace Is Sovereign).
- Reglas específicas de dominio (p.ej. nuevas acciones `query_conversations` de un MCA interno deben ir al enum del schema o "Invalid message format").

### 3. Docs que se mueven con el código
- Si el cambio toca API (WS action, schema, env var, MCA tool surface): ¿se actualizó la doc correspondiente en el **mismo** cambio? (regla "code and docs move together").
- `tools.json` ↔ params del handler en sync (drift conocido: añadir param y olvidar regenerar deja el campo invisible al agente).

### 4. Reglas de seguridad/proceso del repo
- ¿El cambio respeta los gates del repo (linter Biome, límites de complejidad, tamaño de PR)?
- ¿Toca algo marcado como "no tocar sin leer" en CLAUDE.md (secrets scope, manifest icon path, patches dev-only)?

### 5. Anti-patterns documentados del repo
- CLAUDE.md suele listar anti-patterns verificados (en Teros: doble-wrap de URLs static, pickFields que olvida campos visuales, driver de animación react-native en web…). ¿El diff reincide en alguno?

## Hallazgo típico

```
MENOR · conventions · packages/backend/src/handlers/board.ts:30 — nueva WS action `board.archive-task` sin actualizar docs/context/API_SURFACE.md; CLAUDE.md exige "code and docs move together" al cambiar el API surface · actualizar API_SURFACE.md en este mismo cambio · confianza: alta
```

## Anti-patterns

- Revisar convenciones de memoria en vez de leer el CLAUDE.md real del repo (las reglas cambian por repo).
- Subir toda violación de estilo a bloqueante. Estilo es nit salvo invariante duro declarado.
- Ignorar el CLAUDE.md del subdirectorio (a veces hay reglas locales más específicas que la raíz).

## Disciplina

- **Cita la regla**: cada hallazgo de convención apunta a la línea/sección de CLAUDE.md que se viola. Sin cita, es opinión, no convención.
- Si el repo no tiene CLAUDE.md/REVIEW.md, declara "sin convenciones escritas; reviso solo arquitectura" y no inventes reglas.
