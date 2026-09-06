---
status: accepted
---

# `skstate merge` promueve un subconjunto declarado y falla de forma ruidosa ante conflicto

La jerarquía de [ADR 0005](./0005-estado-jerarquico-bajo-orquestacion.md) obliga a consolidar los Σ de los workers al integrar sus espacios de trabajo. Se decide que esa operación **no es una unión de dos Σ, sino la promoción de un subconjunto declarado en el esquema**, y que cualquier colisión residual detiene la operación en vez de resolverse sola.

## Promoción, no unión

El Σ de un worker recién terminado mezcla dos cosas: estado que muere con la tarea (`objective`, `next_action`, `status`, `mode`, `blockers`, `hypotheses`, `files.relevant`) y estado que puede sobrevivirla (`facts`, `decisions`, `constraints`, `files.modified`, `verification.checks`). Unirlo todo llenaría el Σ consolidado con objetivos, acciones y detalles locales de una docena de trabajos, y lo haría crecer sin límite justo donde tiene que permanecer pequeño.

Sólo se promueven los nodos declarados como promovibles en el esquema mediante `x-promote`. La promoción es **recursiva**: no hace falta declarar promovible un objeto padre si sólo algunos de sus hijos lo son. Por eso `files.modified` se promueve y `files.relevant` no; del mismo modo, `verification.checks` se promueve y `verification.overall` no.

`constraints` **sí se promueve**. Una restricción impuesta desde fuera de la tarea —compatibilidad, un no-go del usuario, una condición de despliegue— sigue limitando decisiones posteriores aunque haya sido descubierta por un worker.

**Debilidad asumida**: la granularidad de `facts`, `decisions` y `constraints` sigue siendo por colección, así que dentro de ellas pueden convivir elementos durables y locales. Se acepta a cambio de que la operación sea determinista y de coste cero en tiempo de ejecución. Si resulta ruidoso en la práctica, la salida es marcar la durabilidad por ítem — pero pagar ese coste antes de tener evidencia sería optimizar contra una suposición.

## `verification.overall` es derivado, no promocionado

`verification.overall` resume los checks que existen **en el estado destino**, no el resultado del último worker integrado. Promocionarlo produciría resultados absurdos: dos workers pueden aportar checks distintos y el `overall` del último no representa al conjunto.

Se decide que `overall` sea un campo **derivado por el runtime** después de `patch`, `merge` y `migrate`:

- sin checks, o si todos están `not_run` → `not_run`
- si existe algún `failed` → `failed`
- si todos están `passed` → `passed`
- cualquier mezcla restante de `passed` y `not_run` → `partial`

El modelo no lo escribe. Un parche que incluya `verification.overall` se rechaza como escritura sobre un campo derivado; el CLI lo calcula. Esto evita dos fuentes de verdad dentro del mismo objeto.

## Fallar ruidoso

Las claves semánticas de [ADR 0002](./0002-listas-como-objetos-con-clave-semantica.md) ya resuelven la mayoría de los casos sin que nadie decida: dos workers que registran hechos distintos escriben claves distintas y RFC 7386 las une. Queda sólo la colisión de **clave idéntica con valor distinto**, que por construcción significa que ambos escribieron sobre lo mismo y discrepan. `merge` sale con error listando esas claves. Los escalares no promovibles pertenecen al Σ que los posee y no entran en la operación.

## Consequences

- **El juicio vive fuera del CLI.** Cuando `merge` falla, quien integra resuelve emitiendo un **parche explícito**. El modelo o la persona decide; la escritura sigue siendo determinista y auditable, y el CLI nunca adivina.
- **La integración puede detenerse.** Es el precio de no perder datos en silencio, y es coherente con proteger las escrituras concurrentes mediante un cerrojo ([ADR 0011](./0011-cerrojo-de-fichero-en-lugar-de-compare-and-swap.md)).
- **La promoción no depende de un orquestador.** `merge` opera sobre estados OpenSkillState y su esquema; el consumidor que creó esos estados es irrelevante.
- **El agregado de verificación siempre describe el destino.** Integrar un check nuevo puede cambiar `overall` aunque el worker que lo produjo tuviera otro valor local.

## Considered Options

- **Último gana** — descartado: es *lost update* con otro nombre.
- **Conservar ambos renombrando** (`…-2`, o namespacing por id de worker) — descartado: no elimina la colisión, la vuelve indetectable. Llena Σ de casi-duplicados, rompe la idempotencia y fija procedencia transitoria dentro de claves permanentes.
- **Que el Integrator decida qué promover** — descartado como mecanismo base: resuelve la debilidad de granularidad con criterio real, pero deja de ser determinista y el mismo worker podría promover cosas distintas en dos ejecuciones idénticas. Sigue disponible como capa opcional encima.
- **Promover `verification.overall`** — descartado: resume el estado origen, no el conjunto de checks del destino.
- **Promover todo y podar después** — descartado: traslada el problema al integrador y permite crecimiento innecesario antes de aplicar criterio.
