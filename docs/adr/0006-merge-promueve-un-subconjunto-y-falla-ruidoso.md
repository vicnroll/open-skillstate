---
status: accepted
---

# `skstate merge` promueve un subconjunto declarado y falla de forma ruidosa ante conflicto

La jerarquía de [ADR 0005](./0005-estado-jerarquico-bajo-orquestacion.md) obliga a consolidar los Σ de los workers al integrar sus worktrees. Se decide que esa operación **no es una unión de dos Σ, sino la promoción de un subconjunto declarado en el esquema**, y que cualquier colisión residual detiene la operación en vez de resolverse sola.

## Promoción, no unión

El Σ de un worker recién terminado mezcla dos cosas: estado que muere con la tarea (`objective`, `next_action`, `status`) y estado que sobrevive a ella (`facts`, `decisions`, `files.modified`, `verification`). Unirlo todo llenaría el Σ del orquestador con el `objective` y el `next_action` de una docena de workers, y lo haría crecer sin límite justo donde tiene que estar acotado. Sólo se promueven los campos declarados como promovibles en el esquema.

**Debilidad asumida**: la granularidad es por campo, así que dentro de `facts` conviven hechos durables sobre el repositorio y hechos locales de la tarea, y la promoción no los distingue. Se acepta a cambio de que la operación sea determinista y de coste cero en tiempo de ejecución. Si resulta ruidoso en la práctica, la salida es marcar la durabilidad por ítem — pero pagar ese coste antes de tener workers reales ejecutándose sería optimizar contra una suposición.

## Fallar ruidoso

Las claves semánticas de [ADR 0002](./0002-listas-como-objetos-con-clave-semantica.md) ya resuelven la mayoría de los casos sin que nadie decida: dos workers que registran hechos distintos escriben claves distintas y RFC 7386 las une. Queda sólo la colisión de **clave idéntica con valor distinto**, que por construcción significa que ambos escribieron sobre lo mismo y discrepan. `merge` sale con error listando esas claves. Los escalares no se fusionan nunca: pertenecen al Σ que los posee.

## Consequences

- **El juicio vive fuera del CLI.** Cuando `merge` falla, el Reviewer/Integrator del orquestador — que es un agente — resuelve emitiendo un **parche explícito**. El modelo decide, la escritura sigue siendo determinista y auditable, y el CLI nunca adivina. Es el mismo reparto que en [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md): lo mecánico dentro, el criterio encima.
- **La integración puede detenerse.** Es el precio de no perder datos en silencio, y es coherente con proteger las escrituras concurrentes mediante un cerrojo ([ADR 0011](./0011-cerrojo-de-fichero-en-lugar-de-compare-and-swap.md)): no tendría sentido evitar el *lost update* en el camino de escritura y aceptarlo en el de consolidación.

## Considered Options

- **Último gana** — descartado: es *lost update* con otro nombre.
- **Conservar ambos renombrando** (`…-2`, o namespacing por id de worker, `slug@worker-3`) — descartado: no elimina la colisión, la vuelve indetectable. Llena Σ de casi-duplicados que nada marca como relacionados, rompe la idempotencia entre workers que motivó las claves semánticas, y fija procedencia de una ejecución transitoria dentro de una clave permanente. Además el id de worker no existe en el camino interactivo.
- **Que el Integrator decida qué promover** — descartado como mecanismo base: resuelve la debilidad de granularidad con criterio real, pero deja de ser determinista y el mismo worker podría promover cosas distintas en dos ejecuciones idénticas. Sigue disponible como capa opcional encima.
- **Promover todo y podar después** — descartado: traslada el problema al orquestador, que necesitaría un criterio de poda que es esta misma pregunta desplazada en el tiempo.
