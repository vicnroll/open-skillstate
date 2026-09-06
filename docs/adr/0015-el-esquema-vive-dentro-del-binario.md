---
status: accepted
---

# El esquema vive dentro del binario, no en el repositorio del usuario

El [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md) parte de que el CLI es el dueño del esquema, y esa propiedad es la que hace que la validación signifique algo: el paper la enuncia como *«schema ownership and validation reside in the deterministic runtime rather than the model»*. Una copia editable de `schema.json` por proyecto volvería esa propiedad nominal: bastaría relajarla para que ese repositorio aceptase algo que el binario debería rechazar.

Se decide que el binario **embeba todos los esquemas que conoce** y aplique a cada proyecto el que corresponda a su `schema_version`. En `.openskillstate/` no se instala ningún esquema.

```text
binario
├── esquema 1  ── se aplica en el proyecto A  (state.json: schema_version 1)
└── esquema 2  ── se aplica en el proyecto B  (state.json: schema_version 2)
```

Una tabla y no un único esquema, porque `migrate` necesita conocer origen y destino para transformar entre ellos ([ADR 0014](./0014-migracion-explicita-de-schema-version.md)). Una vez están todos dentro, validar un proyecto contra su propia versión sale gratis.

## Qué reemplaza a la copia en disco

`skstate schema` imprime el esquema vigente, y `--version N` cualquier otro que el binario conozca. Cubre los motivos legítimos para querer verlo — auditoría humana o alimentar una interfaz tipada como MCP — sin dejar un fichero capaz de divergir.

El estado inicial tampoco necesita una copia pre-generada en la carga útil. Desde el [ADR 0019](./0019-identidad-estable-del-proyecto.md), `init` debe generar un `project_id` único; por eso **construye el documento inicial a partir del esquema embebido y de la forma canónica definida por ADR 0003**. Un template estático de `state.json` sería una segunda fuente de defaults y no podría contener una identidad válida única.

## Consequences

- **`.openskillstate/` contiene datos y metadatos del proyecto, no la definición del contrato.** `state.json`, `installed.json`, el estado efímero y registros locales pueden vivir ahí; `schema.json` no.
- **El esquema deja de ser carga útil y pasa a ser código fuente.** Vive en `schema/`, se compila dentro del binario y nunca se copia a cada proyecto.
- **Un proyecto no puede personalizar el esquema.** Es deliberado. Un esquema por proyecto volvería Σ no portable entre repositorios y rompería `merge` bajo ejecución aislada, donde origen y destino tienen que coincidir en la semántica de `x-promote` ([ADR 0006](./0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md)).
- **Añadir un campo exige una versión del binario.** Es el precio de la propiedad real del esquema, y es coherente con que `x-tier`, `x-promote`, `x-derived` y `x-immutable` dirijan comportamiento del runtime.
- **Defaults y canonicalización tienen una sola autoridad.** `init` y `patch` usan la misma versión embebida; no hay un template que pueda quedar desfasado.

## Considered Options

- **La copia en `.openskillstate/` como fuente de verdad** — descartado: convierte la propiedad del esquema en una afirmación que el primer usuario que edite el fichero desmiente.
- **La copia en `.openskillstate/` como artefacto derivado**, escrita por `init` y verificada por `check` — descartado: añade un fichero y un modo de fallo para comprar una auditabilidad que `skstate schema` ya da.
- **Template estático de estado inicial** — descartado: duplica defaults del esquema/canonicalizador y no puede generar una identidad única por instalación.
