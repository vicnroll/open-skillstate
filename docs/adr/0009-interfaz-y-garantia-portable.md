---
status: accepted
---

# La interfaz base es CLI + skill; la garantía portable es la detección de escritura directa

Fuera de Claude Code no hay regla `deny` ni hooks: el CLI es portable porque cualquier agente ejecuta bash, pero el enforcement no lo es. Eso deja sin garantía a dos de los tres backends que un orquestador como Syntony despacha (Codex y OpenCode). Hacen falta dos decisiones distintas que conviene no mezclar — **cómo habla el modelo con skillstate**, y **qué impide o detecta que se lo salte**.

## Interfaz: CLI + skill, con MCP opcional

La skill enseña *cuándo y por qué*: qué comandos existen, cuándo conviene parchear, cómo se deriva un slug, qué es un buen `next_action`. Eso hace falta siempre y ningún servidor MCP lo sustituye. Además una skill puede empaquetar `references/` y `scripts/`, que es donde vive la revelación progresiva de los campos extendidos que [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md) da por supuesta.

Un servidor MCP añadiría validación del *tool input* contra JSON Schema por parte del cliente, y evitaría que el modelo construya una cadena de shell. Se considera **opcional, no requerido**: su argumento práctico más fuerte — el escapado de un JSON anidado como argumento de bash — se neutraliza haciendo que el CLI acepte el parche por **stdin**, donde un heredoc con delimitador entrecomillado no exige escapar nada.

## Garantía: detección, no prevención

El CLI guarda fuera del fichero un hash de lo último que él escribió y lo compara en cada operación. Si `state.json` cambió sin pasar por el CLI, la siguiente llamada lo detecta. No impide el bypass, pero lo vuelve visible **en todos los clientes por igual**, sin depender de nada que el cliente ofrezca.

### Qué detecta de verdad: corrupción, no atajos

La formulación inicial de esta garantía era más ambiciosa y **era incorrecta**. Decía que un desajuste de hash delata a un modelo que ha editado el fichero a mano. No lo delata, porque el Σ del orquestador **está versionado en Git** ([ADR 0008](./0008-politica-de-git-para-el-estado.md)) y eso significa que `git pull`, `git checkout`, un rebase, un merge o un `stash pop` **reescriben `state.json` legítimamente sin pasar por el CLI**. En un repositorio con más de una rama, el desajuste no es la excepción: es rutina. Y una alarma que salta a diario por motivos legítimos deja de ser una señal — se aprende a ignorarla, que es la forma más segura de inutilizar la única garantía portable del diseño.

Se decide **graduar la respuesta por validez** en lugar de por procedencia. Ante un desajuste, el CLI valida el estado contra el esquema:

- **Valida** → el cambio externo estaba bien formado. Se avisa y se restablece la línea base.
- **No valida** → se rechaza, y `check` dice qué está mal.

La consecuencia hay que decirla sin adornos: **un modelo que edite el fichero a mano y lo deje bien formado pasa desapercibido.** Es menos de lo que esta garantía prometía. Pero la promesa era la que estaba mal, no el mecanismo: en un fichero versionado no hay forma de distinguir un atajo de un `git pull` sin acoplar el CLI a Git, y guardar el `HEAD` junto al hash tampoco lo consigue — un atajo con un commit por medio se atribuiría a Git y quedaría igual de invisible. Vale más recortar lo que se promete que mantener escrita una garantía que no se cumple.

Lo que sí queda intacto es el modelo de amenaza: **no es un adversario, es un modelo tomando un atajo**, y el atajo típico deja el fichero mal formado precisamente porque nadie lo validó al escribirlo. La prevención real sigue estando donde siempre estuvo: en la regla `deny`, sólo en Claude Code.

## Consequences

- **Cubre un agujero que abre la propia skill.** Un script empaquetado en `scripts/` es un subproceso arbitrario, la categoría exacta que `deny` no alcanza. Si tocara `state.json` directamente, `deny` no lo pararía; el hash de integridad sí lo detecta. Regla a documentar: los scripts de la skill pasan siempre por el CLI.
- **La garantía deja de ser binaria y pasa a ser por niveles**, y hay que documentarlo con honestidad: prevención en Claude Code vía `deny`, detección en todas partes vía integridad, validación previa donde además haya MCP.

## Considered Options

- **Aceptar la asimetría sin más** — descartado: dejaría la propiedad central del paper (*«malformed outputs cannot corrupt persistent state»*) cumpliéndose sólo en un tercio de los backends.
- **MCP como garantía portable** — descartado *como garantía*: la validación del *tool input* consigue que el parche llegue bien formado, no que sea el único camino de escritura. Portabilidad de la interfaz no es portabilidad de la garantía.
