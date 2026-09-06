---
status: accepted
---

# `skstate patch` acepta sólo el parche, no el contrato de dos claves del paper

El apéndice A.4 del paper define un contrato de salida de dos claves exactas — `{"state_patch": …, "action": …}` — y el documento de traslado propone «respetarlo literalmente». Se decide **no hacerlo**: el CLI acepta únicamente el parche de estado.

El motivo es que ese contrato es un **contrato de salida del modelo**, no un **contrato de entrada del CLI**, y confundir las dos capas arrastra una responsabilidad que OpenSkillState no necesita. En el paper la salida del modelo *es* ese JSON y nada más: un harness lo parsea, aplica el parche y **ejecuta la acción**. Las dos claves son los dos únicos canales por los que el modelo puede afectar al mundo. En cualquier arquitectura donde el modelo tenga sus propias herramientas, `action` describe algo que ya viaja por otro canal, duplica `next_action` — que ya vive en el estado y ya se serializa — y puede desincronizarse de lo que el modelo acabe ejecutando sin que nadie lo detecte.

## Por qué un orquestador tampoco lo rescata por defecto

Un orquestador de agentes suele despachar **tareas o sesiones**, no cada paso interno del bucle del modelo. El worker recibe un trabajo y ejecuta sus propias herramientas por dentro; el orquestador no necesita parsear una clave `action` de cada respuesta para convertirla en una llamada concreta.

OpenSkillState no presupone ningún orquestador específico. Si un harness concreto sí define un contrato de salida de dos claves, ese harness puede parsearlo y enrutar únicamente `state_patch` hacia `skstate patch`. **El CLI no necesita saber que `action` existe en ninguna arquitectura.**

## Considered Options

- **Respetar el contrato literal** — descartado: arrastra una clave redundante que puede mentir sin que nada lo detecte.
- **Aceptarlo ahora e ignorar `action`** — descartado: aceptar claves desconocidas en silencio contradice la validación determinista que justifica que el CLI posea el esquema.
