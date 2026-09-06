---
status: accepted
---

# El rol de worker se declara al crearlo, no en cada invocación

La topología del [ADR 0005](./0005-estado-jerarquico-bajo-orquestacion.md) pone dos Σ en juego, y el [ADR 0008](./0008-politica-de-git-para-el-estado.md) los obliga a tener nombres distintos — `state.json` versionado, `worker-state.json` efímero — porque los worktrees comparten el `.gitignore` del repositorio. Faltaba lo operativo: **cómo sabe el binario sobre cuál de los dos opera**. Equivocarse tiene el peor modo de fallo del diseño: un worker escribiendo sobre el estado consolidado reintroduce en silencio el conflicto entre Git y la promoción semántica.

Se decide que el rol se declare **una vez, al crear el worker**, con `skstate init --worker` en su espacio aislado. A partir de ahí no hay nada que recordar: **si existe `worker-state.json`, se opera sobre él**; si sólo existe `state.json`, se opera sobre ese. La instrucción vive en el cuerpo del `SKILL.md` apuntando a `references/orchestration.md`, siguiendo la revelación progresiva del [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md).

El worker **hereda el `project_id`** del estado consolidado y nunca crea una identidad nueva ([ADR 0019](./0019-identidad-estable-del-proyecto.md)).

## Por qué no una variable de entorno

Una variable como `SKSTATE_ROLE=worker` parece limpia: quien lanza el proceso la inyecta y el modelo ni siquiera necesita saber que existe. Pero convierte la corrección de OpenSkillState en **algo que cada consumidor externo tiene que implementar**. Eso contradice el objetivo de ser agnóstico del orquestador: un runtime que no conozca OpenSkillState se comportaría mal sin aviso.

La declaración persistente evita ese acoplamiento. Cualquier sistema capaz de crear un directorio de trabajo puede ejecutar una vez `skstate init --worker`; a partir de ahí el workspace se describe a sí mismo.

## Por qué tampoco un flag en cada llamada

La alternativa era enseñar que todo `skstate patch` dentro de un worker lleve `--worker`. Se descarta por la forma de la obligación:

| | Cuándo dispara | Qué compite con ella |
|---|---|---|
| «Creas un worker → decláralo» | Al crear el espacio aislado | Nada: la regla se activa durante esa acción concreta |
| `--worker` en cada `patch` | En cada llamada posterior | Todo lo demás del contexto, incluso tras compactación o sesión nueva |

La primera persiste una decisión una sola vez; la segunda convierte una propiedad estructural del workspace en memoria procedimental repetida.

## Por qué la detección de Git avisa pero no decide

En Git puede detectarse un worktree comparando `gitdir` y `commondir`, pero no sirve como mecanismo universal. Un submódulo también puede tener `.git` como fichero y un orquestador puede aislar con contenedores o copias sin usar worktrees.

La detección se queda como **red de seguridad**: si el directorio parece un worktree y nadie declaró un worker, `check` avisa fuerte. No decide el rol y no convierte Git en requisito del producto.

## Consequences

- **La ambigüedad de los dos ficheros se resuelve por presencia del efímero.** Un worktree puede contener el `state.json` versionado heredado y su `worker-state.json`; gana el segundo.
- **No hace falta invocar al binario `git` para decidir el rol.** La detección opcional de worktree puede implementarse leyendo metadatos de Git, pero el funcionamiento base sólo depende del workspace.
- **La declaración persiste como estado, no como instrucción.** Es la tesis del producto aplicada a sí mismo: lo que puede persistirse no se vuelve a derivar en cada paso.
- **`deinit` en un worker es local al worker.** Retira el estado efímero sin tocar la instalación compartida del proyecto principal ([ADR 0012](./0012-init-completa-la-instalacion.md)).

## Considered Options

- **Variable de entorno por orquestador** — descartada: acopla la corrección a cada consumidor.
- **`--worker` en todas las operaciones** — descartado: transforma una propiedad persistente en una obligación repetitiva del modelo.
- **Autodetección exclusiva por Git** — descartada: no cubre todos los mecanismos de aislamiento y puede confundir submódulos.
