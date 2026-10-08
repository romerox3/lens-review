# Metodología — heterogeneidad

> Por qué el fan-out usa revisores con **modelos y priors distintos**, no N copias del mismo. Es la propiedad de la que depende todo el método.

## El hallazgo

Un experimento citado por Osmani corrió **4 revisores en paralelo** (CodeRabbit, Sentry Seer, Greptile, Cursor BugBot) sobre 146 PRs reales, 679 hallazgos, 3,5 semanas. De **617 ubicaciones distintas marcadas**:

- **93,4%** las cazó **exactamente uno** de los cuatro.
- 6% las cazaron dos.
- Casi ninguna, tres.
- **Ninguna las cazaron los cuatro.** Nunca marcaron la misma línea.

Conclusión: no existe "el mejor revisor". Cada revisor caza un conjunto **disjunto** de bugs. Lo que multiplica la cobertura es la **heterogeneidad**, no elegir un campeón.

## Implicación para el fan-out

Maximiza la diversidad de priors, en orden creciente de independencia:

1. **Modelos distintos** (Opus vs Sonnet): barato, in-family, blind spots **parcialmente correlacionados** (misma familia → ángulos ciegos compartidos).
2. **System-prompts/lentes distintos**: un revisor de seguridad-adversario, uno de integridad-de-tests, uno de blast-radius. Instrucciones distintas → hallazgos distintos.
3. **Familia distinta (cross-family)**: la diversidad real de linaje. Un modelo de otra casa tiene ángulos ciegos genuinamente distintos. Es la fuente más potente — y la más cara/dependiente. Este paquete no la incluye.

## Cómo se aplica aquí

- Núcleo in-family: **2×Opus** (correctness, seguridad) + **2×Sonnet** (integridad de tests, blast-radius/arquitectura), cada uno con lente y system-prompt propios → blind spots distintos pese a la familia compartida.
- **Medición**: tras una revisión, mira el **solape** entre revisores. Si coinciden mucho en las mismas líneas, no compraste heterogeneidad — cambia priors. Coincidencia ocasional = alta confianza en ese hallazgo; coincidencia constante = redundancia desperdiciada.

## Anti-pattern

Lanzar 4 revisores con el **mismo modelo y el mismo prompt**. Eso no es un panel heterogéneo; es el mismo revisor ejecutado 4 veces, con el mismo ángulo ciego 4 veces. Coste ×4, cobertura ×1.

## Caveat honesto

Los números de los benchmarks de vendors (CodeRabbit, Greptile…) son parcialmente interesados y dependientes del codebase. La regla de Osmani: **mídelo en tu propio código**. El 93,4% disjunto es el patrón que importa, no las cifras exactas de catch-rate de cada tool.
