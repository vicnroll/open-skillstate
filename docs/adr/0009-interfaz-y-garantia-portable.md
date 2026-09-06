---
status: accepted
---

# La interfaz base es CLI + skill; la garantía portable es detectar corrupción, no controlar el cliente

Fuera de Claude Code no hay una regla `deny` ni los mismos hooks: el CLI es portable porque cualquier agente puede ejecutar un proceso, pero el enforcement del cliente no lo es. Hacen falta dos decisiones distintas que conviene no mezclar — **cómo habla el modelo con OpenSkillState**, y **qué impide o detecta que se lo salte**.

## Interfaz: CLI + skill, con MCP opcional

La skill enseña *cuándo y por qué*: qué comandos existen, cuándo conviene parchear, cómo se deriva un slug, qué es un buen `next_action`. Eso hace falta siempre y ningún servidor MCP lo sustituye. Además una skill puede empaquetar `references/`, que es donde vive la revelación progresiva de los campos extendidos que [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md) da por supuesta.

Un servidor MCP añadiría validación del *tool input* contra JSON Schema por parte del cliente, y evitaría que el modelo construya una cadena de shell. Se considera **opcional, no requerido**: su argumento práctico más fuerte — el escapado de un JSON anidado como argumento de bash — se neutraliza haciendo que el CLI acepte el parche por **stdin**, donde un heredoc con delimitador entrecomillado no exige escapar nada.

OpenSkillState no depende de ningún orquestador concreto. Un runtime externo puede invocar el CLI, envolverlo en MCP o integrarlo en su propio protocolo sin que el binario conozca al consumidor.

## Garantía: detección, no prevención universal

El CLI guarda fuera del fichero un hash de lo último que él escribió y lo compara en cada operación. Si `state.json` cambió sin pasar por el CLI, la siguiente llamada lo detecta. No impide el bypass, pero vuelve visible una divergencia **en todos los clientes por igual**, sin depender de una API propietaria.

### Qué detecta de verdad: corrupción, no atajos

Un desajuste de hash no demuestra que un modelo editó el fichero a mano. El Σ consolidado está versionado en Git ([ADR 0008](./0008-politica-de-git-para-el-estado.md)) y `git pull`, `git checkout`, un rebase o un merge pueden reescribir `state.json` legítimamente sin pasar por el CLI. Una alarma que salta por operaciones correctas deja de ser señal.

Ante un desajuste, el CLI valida el estado contra el esquema:

- **Valida** → el cambio externo está bien formado. Se avisa y se restablece la línea base de integridad.
- **No valida** → se rechaza y `check` explica qué está mal.

La consecuencia hay que decirla sin adornos: **un modelo que edite el fichero a mano y lo deje bien formado pasa desapercibido como bypass**. La prevención real vive sólo donde el cliente puede imponerla.

El modelo de amenaza no es un adversario intentando romper el sistema, sino un agente tomando un atajo. La propiedad portable que sí se puede sostener es que una siguiente operación del runtime no continúa silenciosamente sobre un documento que viola el esquema.

## Claude Code: prevención adicional

En Claude Code `init` instala una regla de proyecto:

```json
{
  "permissions": {
    "deny": ["Edit(/.openskillstate/**)"]
  }
}
```

La forma `Edit(/...)` está anclada al *primary working directory* de los settings del proyecto y cubre las herramientas integradas de escritura; la documentación actual indica además que las reglas de path se consultan contra `Edit(path)` y `Read(path)`, no contra `Write(path)`. Los comandos bash reconocidos y sus redirecciones también se someten a la política, pero un subproceso arbitrario puede quedar fuera de ese análisis. Por eso la regla es prevención adicional, no la garantía portable.

## El cliente puede aportar contexto externo

OpenSkillState controla Σ, **no todo el contexto que un cliente decida cargar**. Un cliente puede añadir instrucciones, memorias propias, variables de entorno o cualquier otra fuente. En Claude Code, por ejemplo, Auto Memory se carga al inicio de cada conversación y se comparte entre worktrees del mismo repositorio.

Eso no obliga a desactivarla: puede contener conocimiento complementario o redundante con Σ. Pero impide afirmar que un worker recibe «literalmente sólo Σ». La formulación correcta es que **OpenSkillState puede eliminar la dependencia de la transcripción previa cuando el runtime arranca una sesión fresca, pero no monopoliza el resto del contexto del cliente**.

`init` no modifica Auto Memory ni otras memorias externas. Si un consumidor necesita un entorno de contexto estrictamente controlado, esa política pertenece al consumidor o al cliente, no al núcleo portable de OpenSkillState.

## Consequences

- **La garantía queda explícitamente por niveles**: prevención donde el cliente la ofrece, detección/validación portable en el CLI, validación previa adicional donde exista MCP.
- **Los scripts o procesos auxiliares deben pasar por el CLI.** Si escriben `state.json` directamente pueden evitar la prevención específica de un cliente.
- **OpenSkillState no promete exclusividad sobre el prompt.** El plano de contexto depende del runtime y de otras fuentes que éste cargue.
- **Portabilidad de interfaz no equivale a portabilidad de enforcement.** La primera sí es objetivo del producto; la segunda se documenta por cliente.

## Considered Options

- **Aceptar la asimetría sin detección portable** — descartado: dejaría la propiedad central del estado dependiendo completamente del cliente.
- **MCP como garantía portable** — descartado *como garantía*: la validación del *tool input* consigue que el parche llegue bien formado, no que sea el único camino de escritura.
- **Desactivar memorias externas del cliente desde `init`** — descartado como política general: OpenSkillState no debe apropiarse de configuración que puede tener usos legítimos ajenos a Σ. Un modo estricto de un consumidor puede hacerlo de forma explícita.
