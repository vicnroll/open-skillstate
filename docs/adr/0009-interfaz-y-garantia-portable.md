---
status: accepted
---

# La interfaz base es CLI + skill; la garantía portable es la detección de escritura directa

Fuera de Claude Code no hay regla `deny` ni hooks: el CLI es portable porque cualquier agente ejecuta bash, pero el enforcement no lo es. Eso deja sin garantía a dos de los tres backends que un orquestador como Syntony despacha (Codex y OpenCode). Hacen falta dos decisiones distintas que conviene no mezclar — **cómo habla el modelo con skillstate**, y **qué impide o detecta que se lo salte**.

## Interfaz: CLI + skill, con MCP opcional

La skill enseña *cuándo y por qué*: qué comandos existen, cuándo conviene parchear, cómo se deriva un slug, qué es un buen `next_action`. Eso hace falta siempre y ningún servidor MCP lo sustituye. Además una skill puede empaquetar `references/` y `scripts/`, que es donde vive la revelación progresiva de los campos extendidos que [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md) da por supuesta.

Un servidor MCP añadiría validación del *tool input* contra JSON Schema por parte del cliente, y evitaría que el modelo construya una cadena de shell. Se considera **opcional, no requerido**: su argumento práctico más fuerte — el escapado de un JSON anidado como argumento de bash — se neutraliza haciendo que el CLI acepte el parche por **stdin**, donde un heredoc con delimitador entrecomillado no exige escapar nada.

## Garantía: detección, no prevención

El CLI guarda fuera del fichero un hash de lo último que él escribió y lo compara en cada operación. Si `state.json` cambió sin pasar por el CLI, la siguiente llamada lo detecta. No impide el bypass, pero lo vuelve **ruidoso en todos los clientes por igual**, sin depender de nada que el cliente ofrezca.

Es suficiente porque el modelo de amenaza no es un adversario, es **un modelo tomando un atajo**. Contra un adversario la detección no serviría — también podría actualizar el registro de integridad —; contra un atajo sirve, porque el atajo se toma justamente cuando se supone que nadie mira.

## Consequences

- **Cubre un agujero que abre la propia skill.** Un script empaquetado en `scripts/` es un subproceso arbitrario, la categoría exacta que `deny` no alcanza. Si tocara `state.json` directamente, `deny` no lo pararía; el hash de integridad sí lo detecta. Regla a documentar: los scripts de la skill pasan siempre por el CLI.
- **La garantía deja de ser binaria y pasa a ser por niveles**, y hay que documentarlo con honestidad: prevención en Claude Code vía `deny`, detección en todas partes vía integridad, validación previa donde además haya MCP.

## Considered Options

- **Aceptar la asimetría sin más** — descartado: dejaría la propiedad central del paper (*«malformed outputs cannot corrupt persistent state»*) cumpliéndose sólo en un tercio de los backends.
- **MCP como garantía portable** — descartado *como garantía*: la validación del *tool input* consigue que el parche llegue bien formado, no que sea el único camino de escritura. Portabilidad de la interfaz no es portabilidad de la garantía.
