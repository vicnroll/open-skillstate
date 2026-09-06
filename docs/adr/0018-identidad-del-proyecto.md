---
status: accepted
---

# La identidad separa patrón, producto, CLI y workspace

Al ir a fijar el módulo de Go se comprobó que **`skillstate` ya está publicado en npm por un tercero**, y no como homónimo: es otra implementación del mismo paper — `vitkuz573/skillstate`, TypeScript, MIT, creada el 2026-09-02 — que publica un binario llamado `skillstate` con un subcomando `init`, adaptadores para Claude Code, Codex y OpenCode, y servidor MCP. Además **escribe en `.skillstate/`**, que era el directorio que este proyecto venía usando.

Eran tres colisiones, no una: el nombre del producto, el del binario en el `PATH`, y el directorio de trabajo dentro del repositorio del usuario. Se decide una identidad propia con cuatro nombres que no se solapan:

| | | |
|---|---|---|
| **SKILL.state** | El patrón | Definido por el paper. No es de nadie |
| **OpenSkillState** | El producto | Esta implementación, con decisiones propias |
| **skstate** | La CLI | Una interfaz del producto, no el producto |
| **`.openskillstate/`** | El workspace | Namespace del producto en el repositorio |

## Por qué el producto no se llama igual que su CLI

Porque no son la misma cosa y confundirlos limita el producto por adelantado. `skstate` es hoy la única interfaz, pero el patrón puede crecer hacia servidor MCP, SDK en varios lenguajes, integraciones de agente o un daemon, y ninguna de esas es «la CLI». Es el mismo reparto que `gh` frente a GitHub CLI, o `opencode` frente a OpenCode: el binario es corto porque se teclea, el producto tiene nombre porque se nombra.

Se descartó **SkillState Kit** por lo mismo: «kit» describe bien una colección de herramientas, pero condiciona semánticamente lo que el producto puede llegar a ser, y quedaría pequeño el día que sea más que una colección. **OpenSkillState** identifica el proyecto y no su forma concreta.

El prefijo `Open` **no reclama ser la implementación oficial ni única** — hay precedentes abundantes de productos abiertos que lo usan sin disputar el concepto que implementan. Identifica una implementación abierta y opinionada, nada más.

Y se descartó `os` como binario, que sería el acrónimo natural: demasiado genérico, se lee como *operating system* y no conserva ninguna relación con el patrón. `skstate` la mantiene, evita la colisión y sigue siendo corto.

## Por qué el workspace cambia de nombre

Mantener `.skillstate/` habría reintroducido, en el único sitio donde ambos productos escriben de verdad —el disco del usuario—, exactamente la colisión que se evitó en el binario. Dos herramientas podrían interpretar, validar o modificar el mismo directorio **bajo la premisa falsa de que sus modelos de estado son compatibles**, y aquí esa compatibilidad no se puede dar por supuesta: OpenSkillState no es una reproducción literal del paper.

Conviene además deshacer una suposición que el proyecto arrastraba: **`.skillstate/` no era una convención del patrón**. El paper define Σ y el operador de fusión, no una disposición de ficheros. Ese nombre salió del prototipo v1, que era invención propia; la otra implementación llegó al mismo por la misma derivación obvia. No estábamos respetando un estándar, estábamos coincidiendo por casualidad — y `.skillstate/` sugería falsamente pertenecer al patrón cuando su contenido tiene semántica específica de esta implementación.

`.openskillstate/` pertenece inequívocamente al producto. Se elige frente a `.skstate/` porque la brevedad no compra nada en una ruta que nadie teclea a diario, mientras que la CLI sí necesita ergonomía; y porque el directorio se reserva desde el principio como **namespace estable del workspace**, compartible en el futuro por las demás interfaces del producto y no sólo por la CLI.

Si algún día se quiere interoperar con otra implementación, será **explícito** — `skstate import`, `skstate export` o un formato común bien definido — y no implícito por que varias herramientas escriban sobre el mismo directorio.

## Consequences

- **La distinción se mantiene en toda la documentación, los ADR y el código.** `SKILL.state` siempre es el patrón; `OpenSkillState` siempre el producto; `skstate` siempre su CLI; `.openskillstate/` siempre el workspace. Está fijada en [`CONTEXT.md`](../../CONTEXT.md) y es lo que permite a un lector saber si algo lo impone el paper o lo decidimos nosotros.
- **La existencia de otra implementación deja de ser un problema y pasa a ser un hecho.** Ambos parten del mismo paper y pueden tomar decisiones arquitectónicas distintas; lo único que hacía falta era no compartir nombres ni ficheros.
- **Renombrar costó un `sed` porque se hizo con tres commits y cero usuarios.** Con usuarios habría costado una migración del workspace, que es justo la clase de deuda que este ADR evita.
- **Queda pendiente renombrar el repositorio en GitHub** de `skill-state-kit` a `open-skillstate`. GitHub redirige la ruta antigua, así que no rompe clones existentes, pero el módulo de Go declara ya la ruta nueva.
