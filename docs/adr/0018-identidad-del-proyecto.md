---
status: accepted
---

# La identidad separa patrón, producto, CLI y workspace

Al fijar el módulo de Go se comprobó que **`skillstate` ya estaba publicado en npm por un tercero**, y no como homónimo casual: era otra implementación del mismo paper que publicaba un binario llamado `skillstate`, adaptadores para varios agentes y un workspace `.skillstate/`.

Eran tres colisiones, no una: el nombre del producto, el del binario en el `PATH`, y el directorio de trabajo dentro del repositorio del usuario. Se decide una identidad propia con cuatro nombres que no se solapan:

| | | |
|---|---|---|
| **SKILL.state** | El patrón | Definido por el paper. No es de nadie |
| **OpenSkillState** | El producto | Esta implementación, con decisiones propias |
| **skstate** | La CLI | Una interfaz del producto, no el producto |
| **`.openskillstate/`** | El workspace | Namespace del producto en el repositorio |

## Por qué el producto no se llama igual que su CLI

Porque no son la misma cosa y confundirlos limita el producto por adelantado. `skstate` es hoy la interfaz prevista, pero el patrón puede crecer hacia servidor MCP, SDK en varios lenguajes, integraciones de agente o un daemon, y ninguna de esas cosas es «la CLI».

Se descartó **SkillState Kit** por lo mismo: «kit» describe una colección de herramientas, pero condiciona semánticamente lo que el producto puede llegar a ser. **OpenSkillState** identifica el proyecto y no su forma concreta.

El prefijo `Open` no reclama ser la implementación oficial ni única. Identifica una implementación abierta y opinionada.

Y se descartó `os` como binario: demasiado genérico, se lee como *operating system* y no conserva una relación clara con el patrón. `skstate` la mantiene, evita la colisión y sigue siendo corto.

## Por qué el workspace cambia de nombre

Mantener `.skillstate/` habría reintroducido, en el único sitio donde dos implementaciones escriben de verdad —el disco del usuario—, exactamente la colisión que se evitó en el binario. Dos herramientas podrían interpretar o modificar el mismo directorio bajo la premisa falsa de que sus modelos de estado son compatibles.

`.skillstate/` no era una convención del paper. El paper define Σ y el operador de fusión, no una disposición de ficheros. Ese nombre salió del prototipo y la otra implementación llegó a la misma derivación obvia.

`.openskillstate/` pertenece inequívocamente al producto. Se elige frente a `.skstate/` porque la brevedad no compra nada en una ruta que nadie teclea a diario, mientras que la CLI sí necesita ergonomía; y porque el directorio se reserva como **namespace estable del workspace**, compartible por futuras interfaces del producto.

Si algún día se quiere interoperar con otra implementación, será **explícito** — import/export o un formato común definido — y no implícito porque varias herramientas escriban sobre el mismo directorio.

## Consequences

- **La distinción se mantiene en toda la documentación, ADR y código.** `SKILL.state` siempre es el patrón; `OpenSkillState` el producto; `skstate` su CLI; `.openskillstate/` su workspace. Está fijada también en [`CONTEXT.md`](../../CONTEXT.md).
- **La existencia de otra implementación deja de ser un problema y pasa a ser un hecho.** Ambas pueden tomar decisiones arquitectónicas distintas sin compartir nombres ni ficheros.
- **El repositorio ya usa `vicnroll/open-skillstate`.** El cambio de identidad se cerró antes de tener usuarios o releases que migrar.
- **El workspace es namespace del producto, no de la CLI.** Futuras interfaces pueden compartirlo sin obligar a renombrar datos persistidos.

## Considered Options

- **Mantener `skillstate` para producto y binario** — descartado: colisiona con una implementación existente y confunde patrón con producto.
- **SkillState Kit** — descartado: describe una forma de distribución concreta y condiciona evolución futura.
- **`.skillstate/`** — descartado: no es estándar del paper y puede ser escrito por otra implementación con semántica distinta.
- **`.skstate/`** — válido técnicamente, pero descartado: optimiza una ruta que casi nunca se teclea a costa de perder el namespace humano estable del producto.
