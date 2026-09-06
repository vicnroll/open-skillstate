---
status: accepted
---

# El esquema vive dentro del binario, no en el repositorio del usuario

El [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md) parte de que el CLI es el dueño del esquema, y esa propiedad es la que hace que la validación signifique algo: el paper la enuncia como *«schema ownership and validation reside in the deterministic runtime rather than the model»*. Pero la primera carga útil enviaba `schema.json` dentro de `.openskillstate/`, es decir, **una copia por proyecto y editable**. Así la propiedad es nominal: basta relajar `additionalProperties` o ensanchar el patrón de slug en esa copia para que el binario acepte en ese repositorio lo que rechaza en todos los demás. Y son N copias de un fichero que no tiene nada de local, capaces de divergir del binario que las interpreta.

Se decide que el binario **embeba todos los esquemas que conoce** y aplique a cada proyecto el que corresponda a su `schema_version`. En `.openskillstate/` no se copia ninguno.

```text
binario
├── esquema 1  ── se aplica en el proyecto A  (state.json: schema_version 1)
└── esquema 2  ── se aplica en el proyecto B  (state.json: schema_version 2)
```

Una tabla y no un único esquema, porque `migrate` necesita de todas formas conocer origen y destino para transformar entre ellos ([ADR 0014](./0014-migracion-explicita-de-schema-version.md)). Una vez están todos dentro, validar un proyecto contra su propia versión sale gratis.

## Qué reemplaza a la copia en disco

`skstate schema` imprime el esquema vigente, y `--version N` cualquier otro que el binario conozca. Cubre los dos motivos por los que tenía sentido tenerlo en el repositorio — que una persona lo audite, y que un servidor MCP obtenga el JSON Schema con el que validar el *tool input* ([ADR 0009](./0009-interfaz-y-garantia-portable.md)) — sin dejar un fichero que pueda divergir.

## Consequences

- **`.openskillstate/` queda como directorio de datos puro.** Sólo el estado, el estado efímero del worker y el registro de integridad. Un fichero menos que puede desincronizarse, y una pregunta menos que responder sobre si el registro de integridad debía cubrirlo también.
- **El esquema deja de ser carga útil y pasa a ser código fuente.** No se instala; se compila dentro. Sale de `skills/`, que es lo que `init` copia, y pasa a `schema/`, que es lo que el binario embebe.
- **Un proyecto no puede personalizar el esquema.** Es deliberado. Un esquema por proyecto volvería Σ no portable entre repositorios y rompería `merge` en silencio bajo orquestación, donde worker y orquestador tienen que coincidir en la forma del estado para que la promoción por campo signifique algo ([ADR 0006](./0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md)).
- **Añadir un campo exige una versión del binario.** Es el precio de la propiedad real del esquema, y es coherente con que `x-tier` y `x-promote` sean datos que dirigen el comportamiento: si el usuario pudiera editarlos, estaría reprogramando `get` y `merge` desde un fichero de configuración.

## Considered Options

- **La copia en `.openskillstate/` como fuente de verdad** — descartado: es lo que había, y convierte la propiedad del esquema en una afirmación que el primer usuario que edite el fichero desmiente.
- **La copia en `.openskillstate/` como artefacto derivado**, escrita por `init` y verificada por `check` contra la embebida — descartado: funciona, pero añade un fichero, una comprobación y un modo de fallo nuevo para comprar una auditabilidad que un comando ya da.
