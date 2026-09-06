---
status: accepted
---

# `skstate patch` acepta sólo el parche, no el contrato de dos claves del paper

El apéndice A.4 del paper define un contrato de salida de dos claves exactas — `{"state_patch": …, "action": …}` — y el documento de traslado propone «respetarlo literalmente». Se decide **no hacerlo**: el CLI acepta únicamente el parche de estado.

El motivo es que ese contrato es un **contrato de salida del modelo**, no un **contrato de entrada del CLI**, y el documento confunde las dos capas. En el paper la salida del modelo *es* ese JSON y nada más: un harness lo parsea, aplica el parche y **ejecuta la acción**. Las dos claves son los dos únicos canales por los que el modelo puede afectar al mundo. En cualquier arquitectura donde el modelo tenga sus propias herramientas, `action` describe algo que ya viaja por otro canal, duplica `next_action` — que ya vive en el estado y ya se serializa — y puede desincronizarse de lo que el modelo acabe ejecutando sin que nadie lo detecte.

## Por qué un orquestador tampoco lo rescata

La objeción evidente es que un harness real recuperaría la función original de `action`. Se comprobó contra un caso concreto — [Syntony](https://github.com/vicnroll/syntony), orquestador headless que despacha workers vía `codex exec`, `claude -p` y `opencode serve` — y **no la recupera**: su unidad de despacho es la *tarea*, no el *paso*. Syntony lanza un worker con un trabajo y el worker corre su propio bucle de herramientas por dentro; el orquestador nunca parsea salida del modelo para ejecutar un comando concreto.

Y aunque algún sistema sí quisiera dos claves de su modelo, las parsearía él y enrutaría sólo `state_patch` hacia el CLI. **El CLI no necesita saber que `action` existe en ninguna arquitectura.**

## Considered Options

- **Respetar el contrato literal** — descartado: arrastra una clave redundante que puede mentir sin que nada lo detecte.
- **Aceptarlo ahora e ignorar `action`** — descartado: aceptar claves desconocidas en silencio contradice la validación determinista que justifica que el CLI posea el esquema.
