---
status: accepted
---

# Bajo orquestación, Σ es jerárquico: uno por worker en su worktree, más uno del orquestador

Un orquestador como [Syntony](https://github.com/vicnroll/syntony) aísla cada worker en su propio worktree de Git por diseño, y despacha varios en paralelo. Eso obliga a decidir dónde vive Σ antes de decidir nada sobre concurrencia, porque **la topología determina si hace falta control de concurrencia siquiera**. Se decide una jerarquía: **un Σ por worker dentro de su worktree**, escrito sólo por ese worker, y **un Σ del orquestador**, escrito sólo por el orquestador. Los workers nunca comparten estado entre sí; la consolidación ocurre al integrar, que es donde el Reviewer/Integrator ya vive.

Compartir un Σ único entre workers deshace por la puerta de atrás el aislamiento que el worktree establece a propósito, y pone la contención justo en los campos que más se escriben.

## Relación con Brainy

`brainyd` gestiona **tareas y sus estados**; `skillstate` guarda el **estado de ejecución** de la tarea que un worker tiene entre manos. Son ámbitos distintos y no se solapan. **Σ** es estado actual acotado, nunca un histórico: si creciera con la cronología reintroduciría el O(T²) que el patrón viene a eliminar.

Eso sigue siendo cierto de Σ, pero **no de skillstate**: el histórico existe, fuera del repositorio y fuera del alcance del modelo que trabaja ([ADR 0017](./0017-historico-local-inalcanzable-desde-el-agente.md)). Lo que reintroduce el O(T²) es que la cronología entre en el prompt, no que esté guardada.

## Consequences

- **Hace falta un `skillstate merge`** que no estaba en el conjunto `get | patch | check | init`, para consolidar los Σ de los workers al integrar sus worktrees.
- **La concurrencia deja de estar en el camino caliente.** Sin escritores concurrentes sobre el mismo fichero, el control de concurrencia pasa de mecanismo principal a red de seguridad, y combinado con las claves semánticas de [ADR 0002](./0002-listas-como-objetos-con-clave-semantica.md) los choques quedan como el caso raro. Esta consecuencia es la que más tarde dejó sin premisa al *compare-and-swap* que se daba por supuesto: al no quedar más caso concurrente que varias integraciones simultáneas sobre el Σ del orquestador, [ADR 0011](./0011-cerrojo-de-fichero-en-lugar-de-compare-and-swap.md) lo sustituyó por un cerrojo de fichero.
- **RFC 7386 no es conmutativo**, así que el orden de fusión al integrar cambia el resultado. La política de fusión hay que decidirla explícitamente, no heredarla del estándar.
- **El CLI resuelve `.skillstate/` relativo al directorio de trabajo**, de modo que la jerarquía no requiere configuración: cada worktree obtiene el suyo por el mero hecho de ser un directorio distinto.

## Considered Options

- **Un Σ compartido fuera de los worktrees** — descartado: da vista agregada en vivo a cambio de contención real y de anular el aislamiento por worktree.
- **Un Σ por worktree sin Σ de orquestador** — descartado: deja al orquestador sin ningún estado propio y no dice quién consolida.
