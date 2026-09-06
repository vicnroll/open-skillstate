---
status: accepted
---

# El rol de worker se declara al crearlo, no en cada invocación

La topología del [ADR 0005](./0005-estado-jerarquico-bajo-orquestacion.md) pone dos Σ en juego, y el [ADR 0008](./0008-politica-de-git-para-el-estado.md) los obliga a tener nombres distintos — `state.json` versionado, `worker-state.json` efímero — porque los worktrees comparten el `.gitignore` del repositorio. Faltaba lo operativo: **cómo sabe el binario sobre cuál de los dos opera**. Equivocarse tiene el peor modo de fallo del diseño: un worker escribiendo sobre el estado versionado reintroduce en silencio el conflicto entre Git y la fusión semántica que el ADR 0008 existe para evitar, y no se descubre hasta integrar.

Se decide que el rol se declare **una vez, al crear el worker**, con `skillstate init --worker` en su worktree. A partir de ahí no hay nada que recordar: **si existe `worker-state.json`, se opera sobre él**; si sólo existe `state.json`, se opera sobre ese. La instrucción vive en el cuerpo del `SKILL.md` apuntando a `references/orchestration.md`, siguiendo la revelación progresiva del [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md).

## Por qué no una variable de entorno

Era la propuesta más limpia en apariencia: el orquestador lanza `SKILLSTATE_ROLE=worker claude -p …` y todo `skillstate` que se ejecute dentro lo hereda sin que el modelo sepa que existe. Pero convierte la corrección de skillstate en **algo que cada orquestador tiene que implementar**, y eso es el mismo error que el [ADR 0009](./0009-interfaz-y-garantia-portable.md) ya rechazó al descartar la prevención vía `deny` como garantía portable: una propiedad que depende de que el entorno coopere no es una propiedad, es una esperanza.

El coste es concreto y comprobable. Orca orquesta con worktrees y no conoce skillstate. Syntony habría que modificarlo. Cualquier orquestador futuro, igual. Y mientras no lo hagan, el comportamiento es incorrecto sin avisar. Un kit que aspira a ser agnóstico del orquestador no puede pedirle al orquestador que lo conozca.

## Por qué tampoco un flag en cada llamada

La alternativa era enseñar en el skill que todo `skillstate patch` dentro de un worker lleve `--worker`. Se descartó por la **forma de la obligación**, no por desconfianza en el modelo:

| | Cuándo dispara | Qué compite con ella |
|---|---|---|
| «Orquestas → declara el worker» | Al crear el worker, un acto deliberado que el agente está ejecutando | Nada: la regla se activa mientras el agente hace justo eso |
| `--worker` en cada `patch` | En cada llamada, la mayoría sin relación con orquestar | Todo lo demás del contexto, en la llamada cuarenta y tras una compactación |

La primera es una regla condicional sobre una acción concreta y funciona bien en la práctica — hay orquestadores basados en skills que despachan miles de workers con protocolos bastante más delicados que este. La segunda es una obligación ambiente, y además el mismo `SKILL.md` se ejecuta bajo `claude -p`, `codex exec` y `opencode`, con fiabilidades distintas.

## Por qué la detección de Git avisa pero no decide

Se comprobó empíricamente el mecanismo:

| Situación | `--git-dir` vs `--git-common-dir` | `.git` es |
|---|---|---|
| Repositorio normal | iguales | directorio |
| Worktree | **distintos** | fichero |
| Submódulo | iguales | **fichero** |
| Fuera de un repositorio | el comando falla | no existe |

Funciona, pero no puede ser el mecanismo de decisión por dos motivos. El atajo evidente —«si `.git` es un fichero, es un worktree»— es **incorrecto**, porque un submódulo también lo es. Y sobre todo, sólo cubre el aislamiento por worktree: si un orquestador aísla con contenedores o copias del repositorio, no hay nada que detectar y el fallo vuelve a ser silencioso.

Se queda como **red de seguridad gratuita**: si esto parece un worktree y nadie declaró un worker, avisar fuerte. No decide, no acopla, y convierte el único hueco de la declaración explícita en ruidoso en lugar de silencioso.

## Consequences

- **La ambigüedad de los dos ficheros se resuelve con una línea.** Un worktree tiene el `state.json` versionado *y* el `worker-state.json` propio; gana el del worker.
- **No hace falta invocar al binario `git`.** Leer el fichero `.git`, seguir su puntero `gitdir:` y compararlo con `commondir` son unas veinte líneas y cero dependencias externas — coherente con el [ADR 0001](./0001-cli-como-binario-compilado.md), y funciona en una imagen mínima sin `git` instalado.
- **La declaración persiste como estado, no como instrucción.** Es la misma tesis del producto aplicada a sí mismo: lo que se puede persistir no se vuelve a derivar en cada paso.
