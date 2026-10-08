# Metodología — no confíes en el reporte

> Cada hallazgo se verifica leyendo el código real, no creyendo al revisor que lo emitió. Adaptado del patrón de `subagent-driven-development/spec-reviewer-prompt.md`.

## El problema

Un revisor LLM produce hallazgos **plausibles**. Plausible ≠ real. El hallazgo puede ser:
- **Ya mitigado**: el símbolo valida aguas arriba, el caller garantiza el invariante.
- **No alcanzable**: el "race" necesita un flujo que no existe; el input "malicioso" nunca llega a la función peligrosa.
- **Mal entendido**: el revisor interpretó mal el contrato o el comportamiento de una librería.

Sin un paso de refutación, esos falsos positivos llegan al informe. Y **un falso positivo erosiona la confianza más rápido que un falso negativo**: tras dos o tres hallazgos inventados, el usuario deja de leer la skill.

## La disciplina

Para cada hallazgo (priorizado por severidad — ver [`../prompts/validation.md`](../prompts/validation.md)):

**NO:**
- Tomar la palabra del revisor sobre lo que encontró.
- Confiar en su descripción del problema sin mirar el código.
- Aceptar su interpretación del contrato/comportamiento.

**SÍ:**
- Leer el `file:line` real y su contexto (función entera, callers, validaciones aguas arriba).
- Comprobar que el problema es **real Y alcanzable** con inputs/flujos plausibles.
- **Construir el escenario concreto** que lo dispara. Si no se puede construir → señal fuerte de REFUTADO.
- Verificar el comportamiento de la librería/API con context7/WebSearch si la afirmación depende de ello.

## Sesgo por defecto

El validador arranca con sesgo **escéptico**: ante la duda, **REFUTADO**. Es preferible perder un hallazgo dudoso (el humano puede cazarlo) que inundar el informe de ruido que entrena al usuario a ignorar la skill.

| Veredicto del validador | Qué pasa |
|---|---|
| CONFIRMADO (con escenario concreto) | Va al informe. |
| REFUTADO | Se descarta (o a un apéndice "refutados" con la razón). |
| INCIERTO (no se pudo confirmar ni refutar) | Va como MENOR "no confirmado, requiere ojo humano". Nunca CRÍTICO/MAYOR. |

## Esto también aplica a los revisores entre sí

Los revisores son **ciegos entre ellos**: ninguno confía en el hallazgo de otro. Si dos coinciden de forma independiente, eso es señal de **alta confianza** (convergencia sin colusión), no una excusa para saltarse la validación.
