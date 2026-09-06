---
status: accepted
---

# Las colecciones de Σ se modelan como objetos con clave semántica, no como arrays

RFC 7386 (JSON Merge Patch) fusiona objetos de forma recursiva pero **reemplaza los arrays enteros**, así que con listas planas añadir un solo hecho obligaría al modelo a reenviar la colección completa en cada parche — el coste en tokens crecería con el tamaño de Σ justo en los campos que más crecen. Se decide modelar `facts`, `decisions`, `blockers` y `files.*` como objetos cuyas claves son slugs derivados del contenido (`auth-en-middleware`), de modo que el borrado por `null` del RFC opera a nivel de ítem **sin ninguna extensión propia** y el tamaño del parche pasa a ser proporcional a los cambios, no al estado.

Las claves las genera el modelo, no el CLI: que el CLI las asignara exigiría sintaxis fuera del estándar para expresar «esto es nuevo», que es precisamente la extensión `$append`/`$remove` descartada. La convención es mecánica y la valida el CLI con expresión regular — kebab-case, 2 a 4 palabras, sin acentos, `^[a-z0-9]+(-[a-z0-9]+){0,3}$`.

## Consequences

- **La deduplicación es idempotente**: reescribir un hecho ya conocido sobrescribe su clave en vez de añadir una entrada casi idéntica. Ataca de frente el envenenamiento de contexto por acumulación de casi-duplicados.
- **La clave semántica funciona además como detector de cronología.** No estaba previsto al decidir esto; se descubrió migrando un estado real al esquema nuevo. Un array acepta que se le anexe narración indefinidamente, porque cada entrada sólo necesita ir después de la anterior. Una clave derivada del contenido obliga a responder «¿de qué trata esta entrada?», y cuando la respuesta honesta es «del séptimo turno de la conversación», la entrada se queda sin nombre posible y su naturaleza queda a la vista. En la migración de referencia, `decisions` tenía 27 entradas de las que casi todas empezaban por «Q1 cerrada», «Q2 cerrada»…: cronología disfrazada de estado, escrita sin que nadie lo notara y duplicando decisiones que ya vivían en sus propios ADR. Al forzar los slugs quedaron 3 entradas, y el estado pasó de unos 11 KB a 3,2 KB. La regla «el estado describe el presente, no la cronología» deja así de depender de la disciplina de quien escribe y pasa a tener un mecanismo que la delata.
- **La concurrencia deja de ser el caso habitual**: dos escritores que registran hechos *distintos* eligen claves distintas y no colisionan. Con ids secuenciales ambos calcularían el mismo «siguiente libre» y chocarían de forma sistemática en cuanto un orquestador despache workers en paralelo.
- **Se pierde el orden de inserción.** Si en algún momento importa presentar los ítems en el orden en que aparecieron, hay que guardarlo explícitamente; el objeto no lo conserva por sí mismo.
- **Riesgo residual bajo**: nada garantiza que el modelo derive el mismo slug para el mismo hecho en dos sesiones distintas. El fallo produce un duplicado, no pérdida de datos, y `skillstate check` puede señalarlo.

## Considered Options

- **RFC 7386 puro sobre arrays** — descartado: obliga a reenviar la colección completa en cada parche, justo en el campo que más crece.
- **RFC 7386 + azúcar `$append` / `$remove`** — descartado: resuelve el coste pero deja de ser el estándar, y esa semántica extra hay que documentarla y mantenerla indefinidamente.
- **Claves secuenciales** (`f1`, `f2`…) — descartado: exigen que el modelo recorra Σ para calcular el siguiente libre, y hacen que dos escritores concurrentes colisionen de forma sistemática y silenciosa.
- **Claves por hash del contenido** (`f-a3f9c1`) — descartado: deduplicación perfecta, pero vuelve Σ ilegible para un humano que quiera auditarlo a mano.
