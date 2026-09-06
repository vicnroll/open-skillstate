---
status: accepted
---

# Bajo orquestación, Σ es jerárquico: uno por worker aislado, más uno consolidado

Cualquier orquestador que despache varios workers en paralelo necesita aislar sus escritorios de trabajo — mediante worktrees de Git, copias del repositorio, contenedores u otro mecanismo equivalente. Eso obliga a decidir dónde vive Σ antes de decidir nada sobre concurrencia, porque **la topología determina si hace falta control de concurrencia siquiera**.

Se decide una jerarquía: **un Σ por worker dentro de su espacio aislado**, escrito sólo por ese worker, y **un Σ consolidado**, escrito por quien integra. Los workers nunca comparten estado entre sí; la consolidación ocurre al integrar.

OpenSkillState **no depende de ningún orquestador concreto**. Un sistema externo puede adoptar esta topología si le resulta útil, pero la corrección del CLI no presupone Syntony, Orca ni ningún otro runtime.

Compartir un Σ único entre workers deshace por la puerta de atrás el aislamiento que el entorno de ejecución establece a propósito, y pone la contención justo en los campos que más se escriben.

## Relación con gestores de tareas externos

Un gestor de tareas puede mantener el estado de una tarea (`todo`, `running`, `done`, asignación, prioridad) mientras `skstate` mantiene el **estado de ejecución** que permite a un agente continuar ese trabajo. Son ámbitos distintos. OpenSkillState no integra ni depende de ningún gestor de tareas concreto.

Σ es estado actual acotado por disciplina y presupuesto, nunca un histórico. El histórico local de OpenSkillState existe fuera del repositorio y fuera del flujo normal del agente que trabaja ([ADR 0017](./0017-historico-local-inalcanzable-desde-el-agente.md)); almacenar cronología no reintroduce O(T²) mientras no vuelva a entrar automáticamente en el prompt.

## Consequences

- **Hace falta un `skstate merge`** que no estaba en el conjunto `get | patch | check | init`, para consolidar los Σ de los workers al integrar sus espacios de trabajo.
- **La concurrencia deja de estar en el camino caliente.** Sin escritores concurrentes sobre el mismo fichero, el control de concurrencia pasa de mecanismo principal a red de seguridad, y combinado con las claves semánticas de [ADR 0002](./0002-listas-como-objetos-con-clave-semantica.md) los choques quedan como el caso raro. Esta consecuencia es la que más tarde dejó sin premisa al *compare-and-swap* que se daba por supuesto: al no quedar más caso concurrente que varias integraciones simultáneas sobre el Σ consolidado, [ADR 0011](./0011-cerrojo-de-fichero-en-lugar-de-compare-and-swap.md) lo sustituyó por un cerrojo de fichero.
- **RFC 7386 no es conmutativo**, así que el orden de promoción puede cambiar el resultado cuando dos fuentes afectan la misma clave. La política está definida en [ADR 0006](./0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md): las colisiones semánticas no se resuelven automáticamente.
- **El CLI resuelve `.openskillstate/` relativo al directorio de trabajo.** Cada espacio aislado obtiene su propio workspace por el mero hecho de ser un directorio distinto.
- **El aislamiento no obliga a usar Git worktrees.** Los worktrees son un mecanismo común y especialmente cómodo, pero contenedores o copias independientes funcionan bajo la misma semántica si se inicializan como worker.

## Considered Options

- **Un Σ compartido fuera de los espacios aislados** — descartado: da vista agregada en vivo a cambio de contención real y de anular el aislamiento.
- **Un Σ por worker sin Σ consolidado** — descartado: deja al integrador sin estado durable y no dice dónde termina el conocimiento que sí sobrevive a una tarea.
- **Acoplar la topología a un orquestador concreto** — descartado: OpenSkillState es una herramienta de estado, no un plugin de un runtime. La interoperabilidad se expresa mediante el CLI y el filesystem, no mediante conocimiento del consumidor.
