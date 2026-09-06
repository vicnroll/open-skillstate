---
status: accepted
---

# `init` completa la instalación; `uninstall` y `deinit` separan integración de datos

Un `init` que termina diciendo «instalación completada» y deja pasos manuales pendientes está roto. Se decide que `skstate init` **complete la inicialización del proyecto**, detectando el entorno y generando lo que corresponda: workspace, skills, instrucciones, reglas de cliente, hooks y política Git.

Esa decisión obliga a definir desde v1 dos caminos inversos distintos. Quitar la integración del agente y dejar de considerar el repositorio un proyecto OpenSkillState **no son la misma operación**:

- `skstate uninstall` retira las **integraciones de cliente** pero conserva el workspace, el estado y la identidad del proyecto.
- `skstate deinit` desinicializa el **proyecto OpenSkillState completo**: ejecuta el equivalente de `uninstall`, retira la política Git y elimina el workspace. Es destructivo para Σ y requiere consentimiento explícito.

El histórico local no se purga en ninguna de las dos operaciones salvo petición explícita; su ciclo de vida es distinto ([ADR 0017](./0017-historico-local-inalcanzable-desde-el-agente.md)).

## Qué hace `init`

`init`:

1. construye `.openskillstate/state.json` con la forma canónica de la versión vigente;
2. genera el `project_id` estable y el nombre `project` ([ADR 0019](./0019-identidad-estable-del-proyecto.md));
3. instala la skill en las rutas nativas necesarias;
4. añade las entradas de `.gitignore` para `worker-state.json` e `.integrity`;
5. añade `.openskillstate/state.json -merge` a `.gitattributes` para que Git nunca fusione el estado consolidado en silencio ([ADR 0008](./0008-politica-de-git-para-el-estado.md));
6. en Claude Code, fusiona la regla `deny` y los hooks en `.claude/settings.json`;
7. opcionalmente inserta el bloque delimitado en `CLAUDE.md` y `AGENTS.md`.

El estado inicial **no se copia desde un template estático**: `project_id` debe ser único por instalación y el binario ya conoce la forma canónica del esquema.

## Dos rutas de skill, no una por cliente

Las rutas base son `.claude/skills/` para Claude Code y `.agents/skills/` para clientes que adopten la convención compartida. Multiplicar copias del mismo fichero por cliente no añade compatibilidad, añade lugares donde el contenido puede divergir.

El criterio general es instalar en la convención compartida siempre y añadir una ruta propietaria sólo cuando un cliente no la lea. OpenSkillState no contiene lógica específica de un orquestador.

## Los hooks apuntan al binario, no a scripts

La configuración que `init` escribe apunta al propio binario — `skstate hook stop`, `skstate hook session-start` — y no a scripts empaquetados en el repositorio.

Un script de hook tendría que leer y entender el estado, reintroduciendo una dependencia de runtime y un segundo sitio donde viva conocimiento operativo del esquema. El [ADR 0001](./0001-cli-como-binario-compilado.md) y el [ADR 0015](./0015-el-esquema-vive-dentro-del-binario.md) existen precisamente para evitar esa deriva.

`hook` no forma parte de la superficie normal de usuario: es el punto de entrada que invoca el cliente.

## Configuración fusionada y registro de propiedad

`init` puede modificar ficheros que el usuario ya tiene: `.claude/settings.json`, `.gitignore`, `.gitattributes`, `CLAUDE.md` y `AGENTS.md`. Nunca los sustituye completos.

Para Markdown se usan bloques delimitados:

```text
<!-- BEGIN skstate -->
...
<!-- END skstate -->
```

Para JSON y líneas compartidas no existe un marcador interno fiable. Por eso `.openskillstate/installed.json` registra **exactamente qué añadió esta instalación**. Reejecutar `init` sustituye lo propio en vez de duplicarlo; retirar la integración elimina sólo lo que el registro demuestra que pertenece a OpenSkillState.

Detectar por contenido actual se descarta: el usuario puede haber modificado una entrada o una versión nueva puede instalar una forma distinta. Adivinar propiedad durante el borrado es peor que conservar un registro explícito.

## Claude Code: regla de escritura

La regla instalada es:

```json
{
  "permissions": {
    "deny": [
      "Edit(/.openskillstate/**)"
    ]
  }
}
```

`Edit(path)` es la regla que Claude Code consulta para las herramientas integradas que modifican ficheros. La `/` inicial se ancla al *primary working directory* de los project settings, por lo que en un worktree protege el workspace de ese worktree. No se añade `Write(path)`: la documentación actual acepta esa forma pero no la consulta para permisos de path.

`init` **no desactiva Auto Memory** ni otras memorias del cliente. Son contexto externo a OpenSkillState y pueden coexistir con Σ; un consumidor que necesite aislamiento de contexto más estricto debe declararlo en su propia configuración ([ADR 0009](./0009-interfaz-y-garantia-portable.md)).

## Consentimiento sin romper ejecución headless

Editar ficheros que el usuario ha escrito exige una decisión, pero un prompt interactivo no sirve en ejecución headless. Se resuelve por detección de TTY:

- **Con TTY** — pregunta si debe editar `CLAUDE.md` / `AGENTS.md`.
- **Sin TTY** — exige `--write-instructions` o `--no-write-instructions`; si no recibe ninguna, falla diciendo qué falta.

La regla es agnóstica del consumidor: cualquier CI, script u orquestador puede usar la vía headless sin que OpenSkillState tenga que conocerlo.

## `uninstall`: quitar integración sin perder continuidad

`skstate uninstall`:

- elimina las skills que `installed.json` identifica como instaladas;
- retira de `.claude/settings.json` únicamente la regla y hooks añadidos por OpenSkillState;
- retira los bloques delimitados de `CLAUDE.md` / `AGENTS.md`;
- actualiza `installed.json` para reflejar que ya no hay integraciones instaladas.

**No elimina** `state.json`, `project_id`, `.gitattributes`, las entradas de `.gitignore` necesarias para el workspace ni el histórico. El proyecto sigue inicializado y puede seguir usando el CLI directamente o reinstalar integraciones más tarde sin perder identidad.

## `deinit`: dejar de ser un proyecto OpenSkillState

`skstate deinit` es el inverso completo de `init` a nivel de proyecto:

1. retira las integraciones como `uninstall`;
2. elimina las entradas propias de `.gitattributes` y `.gitignore`;
3. elimina `.openskillstate/`, incluido `state.json`, `installed.json` e integridad local.

Con TTY pide confirmación explícita antes de borrar Σ. Sin TTY exige `--yes`; no existe un borrado destructivo implícito.

`deinit` **no borra el histórico local**. Si se desea, `--purge-history` puede solicitar además el borrado de los eventos de ese `project_id`, pero debe ser explícito porque el histórico vive fuera del repositorio. Si después se vuelve a ejecutar `init`, se genera un `project_id` nuevo y se considera una nueva instancia del proyecto.

En un workspace declarado como worker, `deinit` elimina sólo el `worker-state.json` efímero y sus metadatos locales; no modifica los ficheros compartidos/versionados del proyecto principal.

## Consequences

- `init` deja un proyecto utilizable sin pasos manuales ocultos.
- `uninstall` permite cambiar de cliente o retirar automatización sin destruir continuidad.
- `deinit` tiene una semántica destructiva clara y distinta de desinstalar.
- La propiedad de las modificaciones se registra en lugar de inferirse.
- Añadir un cliente nuevo suele costar cero si lee `.agents/`; sólo las capacidades propietarias requieren adaptación explícita.

## Considered Options

- **`init` como scaffolder que sólo imprime instrucciones** — descartado: produce instalaciones incompletas que parecen correctas.
- **Un único comando inverso que siempre borra todo** — descartado: mezcla integración con datos y hace peligroso quitar una skill o hook.
- **Detectar por contenido qué retirar** — descartado: falla cuando el usuario o una versión posterior modifica lo instalado.
- **Desactivar configuraciones externas del cliente por defecto** — descartado: OpenSkillState no debe apropiarse de opciones que no son necesarias para la corrección del estado.
