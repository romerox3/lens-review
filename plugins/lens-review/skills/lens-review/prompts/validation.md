# Prompt — Validación per-finding (Paso 4)

Cada hallazgo se valida con un subagente **escéptico e independiente** que intenta **refutarlo** leyendo el código real. Filosofía: ver [`../methodology/dont-trust-the-report.md`](../methodology/dont-trust-the-report.md). Un falso positivo erosiona la confianza más rápido que un falso negativo — por eso solo pasa lo de **alta señal**.

## Por qué validar

Los revisores generan hallazgos plausibles que a veces no son reales (el símbolo ya valida aguas arriba, el "race" no es alcanzable, el caller garantiza el invariante). Sin un paso de refutación, esos falsos positivos llegan al informe y el usuario aprende a ignorar la skill.

## Qué validar (por tier)

- **Tier 0/1**: valida CRÍTICO y MAYOR (los que recomiendan bloquear). MENOR/NIT pasan sin validar pero marcados como "no validados".
- **Tier 2**: valida **todo**, incluido MENOR si toca el dominio sensible.

## Cómo lanzar el validador

Un validador por hallazgo (o por lote pequeño si son muchos), en paralelo. Modelo: el mismo tier que el revisor que lo generó (fable para correctness/security — fallback opus; opus para tests; sonnet para architecture). Puedes usar `agentType="general-purpose"` con prompt escéptico, o el mismo `agentType` del revisor con instrucción de refutar.

Prompt del validador (rellenar):

```
Vas a INTENTAR REFUTAR un hallazgo de code review. Tu sesgo por defecto es: REFUTADO si hay cualquier duda.

## Hallazgo a refutar
<SEVERIDAD · lente · file:line — descripción · fix · confianza>

## Intención del cambio
<statement del intake>

## Refs
git diff <base>...<head> · archivos: <lista>

## CRÍTICO: no confíes en el hallazgo
El revisor pudo equivocarse. Verifica TÚ leyendo el código real:
- Lee file:line y su contexto (la función entera, los callers, las validaciones aguas arriba).
- ¿El problema es REAL y ALCANZABLE con inputs/flujos plausibles? ¿O ya está mitigado en otro sitio?
- ¿El fix sugerido es correcto y necesario?
- Construye, si puedes, el escenario concreto que dispara el bug (inputs + secuencia). Si no puedes construirlo, es señal de REFUTADO.

## Veredicto (formato exacto)
veredicto: CONFIRMADO | REFUTADO | INCIERTO
razon: <1-2 frases con file:line de la evidencia>
severidad_ajustada: <si CONFIRMADO, confirma o corrige la severidad>
escenario: <si CONFIRMADO, el trigger concreto; si no, "no reproducible">
```

## Filtrado

| Veredicto | Acción |
|---|---|
| CONFIRMADO | Pasa al informe con la severidad ajustada. |
| REFUTADO | **Se descarta.** No aparece en el informe (o, opcional, en un apéndice "refutados" con la razón). |
| INCIERTO | Pasa como **MENOR** con etiqueta "no confirmado — requiere ojo humano". Nunca como CRÍTICO/MAYOR. |

## Perspectiva diversa (opcional, Tier 2)

Para hallazgos CRÍTICO en Tier 2, lanza 2-3 validadores con **lentes distintas** (corrección / ¿se reproduce? / ¿impacto real en seguridad?) en vez de 3 copias del mismo. Confirma solo si la mayoría confirma. Diversidad caza modos de fallo que la redundancia no.

Continúa a [`synthesis.md`](synthesis.md).
