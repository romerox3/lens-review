# Lente — blast-radius-architecture

> Alcance del cambio, acoplamiento, diseño. La lente de arquitectura del revisor `lens-review:lens-architecture` (Sonnet). Pregunta central: **¿qué más se rompe por este cambio, y está el diseño a la altura sin over-engineering?**

## Ejes a recorrer

### 1. Blast-radius (cuantitativo)
- Para cada símbolo exportado que cambia de firma/significado: ¿cuántos callers tiene? (`grep -rn "<symbol>"`).
- ¿El cambio toca código a distancia que dependía del significado viejo? (paths de fondo: workers, crons, executors, migraciones — no solo el user-facing).
- ¿Hay un guard estructural (tipo/invariante/lint-as-test) que falle el build ante la violación, o se confía en "acordarse"?

### 2. Acoplamiento y cohesión
- ¿El cambio introduce dependencia entre módulos que deberían estar separados?
- Feature envy: ¿una función usa más datos de otro módulo que del suyo?
- ¿Hay estado oculto/global nuevo que dificulta razonar localmente?

### 3. Separación de capas
- ¿La capa correcta hace cada cosa? (en Teros: backend devuelve **datos** —ids + nombres resueltos—, frontend compone UI; el backend no debe devolver strings de UI).
- ¿La validación está en el boundary, con código limpio dentro? ¿O validación repetida en N capas?

### 4. Duplicación significativa
- ¿El cambio duplica lógica que ya existe? ¿Debería reusar una utilidad existente?
- ¿Copy-paste de un patrón que, si cambia, habrá que tocar en N sitios (shotgun surgery)?
- Reuso vs over-engineering: extraer abstracción se justifica si elimina duplicación que ya causó/causará bugs y hay >1 consumidor. No por "patrones por patrones".

### 5. Simplicidad / over-engineering
- ¿Es la solución más simple que resuelve el problema, sin indirección gratuita?
- ¿Archivo/abstracción nuevos justificados (elimina duplicación, >1 consumidor, mejora navegabilidad), o añaden capas sin beneficio?
- Complejidad cognitiva: funciones largas (>50 líneas), anidamiento profundo, demasiados branches.

### 6. Evolución y compatibilidad
- Si toca un primitivo/contrato compartido: ¿el cambio es **aditivo y retrocompatible** (props opcionales, tokens nuevos, types ampliados), o rompe call-sites existentes sin plan de migración?
- ¿El cambio de un schema/evento/API necesita deploy atómico, o tolera shape viejo+nuevo?

## Hallazgo típico

```
MAYOR · architecture · packages/backend/src/mca/connection-manager.queries-board.ts:120 — este path paralelo al WsRouter handler no reusa _helpers.ts; el shape diverge del handler equivalente, y un cambio futuro tocará dos sitios sin guard que lo detecte · extraer el shape a _helpers.ts compartido + test de simetría set↔switch · confianza: alta
```

## Anti-patterns

- Cambiar el significado de algo y scopear solo la superficie obvia (re-auditar TODOS los usos).
- Backend devolviendo strings de UI (`message: "Workspace archivado"`) en vez de datos.
- Extraer una abstracción con un solo consumidor "por si acaso".
- Romper la firma de un prop compartido (renombrar `error`→`message`) sin plan de migración.
- Función de 200 líneas con 8 niveles de anidamiento "porque funciona".

## Disciplina

- Mide el blast-radius con grep real, no a ojo.
- Distingue "smell sistémico ortogonal" (va a un ticket dedicado) de "bug del PR" (se arregla en el PR). No mezclar limpieza masiva con el cambio puntual.
- Prefiere un guard estructural (lint-as-test, tipo, invariante) sobre la disciplina humana.
