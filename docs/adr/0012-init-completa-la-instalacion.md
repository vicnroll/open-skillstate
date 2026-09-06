---
status: accepted
---

# `skillstate init` completa la instalación, incluida la edición de los ficheros de instrucciones

Un `init` que termina diciendo «instalación completada» y deja al usuario un paso manual pendiente está roto: nadie lee el aviso, y el kit queda a medias sin que nada lo indique. No existen apenas herramientas que se instalen y luego pidan que añadas algo a mano. Se decide que `init` **complete la instalación**, detectando el entorno y generando lo que corresponda — `.skillstate/`, la skill en las rutas nativas de cada cliente presente, la entrada de `.gitignore`, y la configuración de permisos y hooks donde apliquen — **incluida la sección en `CLAUDE.md` y `AGENTS.md`**.

## Dos rutas de skill, no una por cliente

«Las rutas nativas de cada cliente presente» son **dos**, no tres: `.claude/skills/` para Claude Code y `.agents/skills/` para el resto. **OpenCode lee `.agents/`**, comprobado, así que la copia en `.opencode/` que llevaba el prototipo desaparece de la carga útil: cubría un cliente que ya estaba cubierto.

El criterio general es instalar en la convención compartida siempre, y añadir una ruta propietaria sólo cuando un cliente no lea la compartida. Multiplicar copias del mismo fichero por cliente no añade compatibilidad, añade sitios donde el contenido puede divergir sin que nadie lo note.

## Los hooks apuntan al binario, no a scripts

La configuración de hooks que `init` escribe apunta al propio binario — `skillstate hook stop`, `skillstate hook session-start` — y no a un script empaquetado en el repositorio.

El motivo es el mismo que sostiene el [ADR 0001](./0001-cli-como-binario-compilado.md). Un script de hook tendría que leer el estado para decidir si bloquea, y para eso necesita `jq`, `python` o parsear JSON en bash: reintroduce por la puerta de atrás la dependencia de runtime que se rechazó para el CLI, y en repositorios de cualquier stack. Además pondría conocimiento del esquema en un segundo sitio, capaz de desincronizarse del binario sin que nada lo detecte — que es exactamente la razón por la que el esquema no se copia al repositorio ([ADR 0015](./0015-el-esquema-vive-dentro-del-binario.md)).

`hook` no forma parte de la superficie de usuario: nadie lo escribe nunca. Es el punto de entrada por el que el cliente invoca al binario, y por eso no aparece en la tabla de comandos.

## Consentimiento sin romper la ejecución headless

Editar ficheros que el usuario ha escrito exige permiso, pero pedirlo por consola rompería la llamada desde un orquestador headless como Syntony. Se resuelve por detección de TTY:

- **Con TTY** — pregunta si debe editar `CLAUDE.md` / `AGENTS.md`.
- **Sin TTY** — exige `--write-instructions` o `--no-write-instructions`, y si no recibe ninguna **falla** en lugar de decidir por su cuenta. Un `init` headless que elige en silencio es peor que uno que se detiene diciendo qué bandera falta.

## Edición delimitada

La sección se escribe entre marcas `<!-- BEGIN skillstate -->` y `<!-- END skillstate -->`. Eso la hace **idempotente** — reejecutar `init` sustituye el bloque en vez de duplicarlo — y **reversible**, porque se puede retirar sin tocar el resto del fichero. El contenido que el usuario haya escrito fuera del bloque no se altera nunca.

## Consequences

- **`init` deja de ser un scaffolder inerte** y pasa a modificar ficheros existentes, lo que exige que sea conservador por defecto: no sobrescribe nada fuera de su bloque delimitado.
- **Hace falta el camino inverso.** Si `init` edita, algo tiene que poder deshacerlo limpiamente.
- **Dar soporte a un cliente nuevo suele costar cero.** Si lee `.agents/`, ya está instalado. Sólo obliga a tocar `init` el cliente que exija ruta propia, y ese caso hay que justificarlo.
