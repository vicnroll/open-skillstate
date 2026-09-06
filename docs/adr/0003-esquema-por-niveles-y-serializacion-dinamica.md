---
status: accepted
---

# El esquema se organiza en niveles y `skillstate get` serializa sólo lo que tiene contenido

El análisis previo criticaba que el esquema tuviera trece campos, porque Σ viaja en todos los prompts, y proponía amputarlo a los cinco que el paper validó. Esa crítica asume que el fichero *es* lo que llega al prompt, lo cual dejó de ser cierto en cuanto se decidió que un CLI fuera el dueño del esquema (ver [ADR 0001](./0001-cli-como-binario-compilado.md)): `skillstate get` no es un `cat`, es una **función de renderizado**. Se decide separar lo que el esquema *permite* de lo que la serialización *emite*, de modo que un campo raro no cueste nada cuando está vacío y no haga falta elegir entre capacidad y compacidad.

## Los tres niveles

- **Núcleo** — `status`, `objective`, `next_action`, `facts`, `decisions`, `files`, `verification`. Documentados en detalle en el skill y **serializados siempre**, aunque estén vacíos: el esqueleto le enseña al modelo la forma del estado que debe mantener, y la ausencia de `next_action` sería ambigua justo donde el hook `Stop` necesita distinguir «no hay siguiente paso» de «se olvidó de ponerlo».
- **Extendido** — `hypotheses`, `blockers`, `constraints`. Validan igual que el núcleo, pero **sólo se serializan cuando tienen contenido**, y el skill los menciona en una línea en vez de describirlos, apoyándose en la revelación progresiva.
- **Cortados** — `phase` y `acceptance_criteria`, eliminados del esquema. No son raros, son **redundantes**: `phase` no aporta nada que `status` no exprese ya, y `acceptance_criteria` es `verification` escrito antes de ejecutarlo. Un campo redundante se rellena *y* cuesta tokens; ningún renderizado dinámico arregla eso.

La regla que separa los dos primeros niveles del tercero: **omitir lo raro, cortar lo redundante.**

## Qué formato emite

**JSON compacto, en una sola línea.** El almacenamiento tiene que ser JSON porque RFC 7386 opera sobre JSON, pero el renderizado era una elección libre y se midió antes de tomarla, sobre el estado real de este repositorio y con los campos vacíos ya filtrados:

| Formato | Bytes | Frente a `indent=2` |
|---|---|---|
| JSON `indent=2` | 3.507 | — |
| YAML | 3.209 | −8,5 % |
| Markdown | 3.302 | −5,8 % |
| **JSON compacto** | **3.211** | **−8,4 %** |

JSON compacto ahorra **lo mismo que YAML sin ninguna de sus contrapartidas**: el modelo lee la misma sintaxis que escribe, de modo que el slug que ve en `facts` es literalmente la cadena que teclea en el parche, y no hay reglas de citado que se comporten distinto según el contenido del valor. La medición además reencuadra el problema: **el 71 % de Σ son las frases y sólo el 29 % es sintaxis**, así que ningún formato mueve mucho la aguja y la palanca real sobre el tamaño es escribir entradas concisas.

Dos matices que conviene tener escritos. En **tokens** el ahorro es menor que en bytes, quizá la mitad, porque los tokenizadores fusionan las rachas de espacio en blanco y una sangría entera suele costar un solo token. Y un blob de una línea enseña la forma del estado algo peor que uno indentado, que es parte de por qué el núcleo se serializa siempre; se asume, porque los modelos leen JSON minificado sin dificultad.

**El fichero en disco se queda indentado.** El Σ del orquestador está versionado ([ADR 0008](./0008-politica-de-git-para-el-estado.md)) y el diff de un JSON de una sola línea es inservible. Compacto es el renderizado, indentado es el almacenamiento — que es exactamente lo que significa que `get` no sea un `cat`. `--pretty` lo indenta para quien lo lea por pantalla.

## Qué cuenta como vacío

`null`, `[]`, `{}` y `""` se omiten. `0` y `false` **no**: son valores legítimos, no ausencias. La regla se fija aquí explícitamente para que no dependa del criterio de quien implemente el serializador.

## Consequences

- **`skillstate get` no devuelve el contenido del fichero.** Es una vista, y quien lo lea esperando un volcado se va a confundir. Por eso existe `skillstate get --raw`, que devuelve `state.json` íntegro para depurar y auditar.
- **El coste real que se ataca no es el de tokens.** Un campo vacío ocupa poco; lo caro es que **invita al modelo a rellenarlo** con contenido plausible pero inútil. Ese coste no aparecería en ninguna telemetría de tokens, así que no se podía medir para decidirlo — había que diseñarlo.
- **La revelación progresiva del skill es la otra mitad del mecanismo.** De poco sirve omitir los campos extendidos de Σ si el `SKILL.md` se los enumera con tres líneas de explicación cada uno en todas las sesiones.
- **Se puede crecer sin pagar.** Añadir un campo extendido nuevo cuesta cero para quien no lo use, lo que quita presión a la decisión de qué entra en el esquema.

## Considered Options

- **Amputar a los cinco campos del documento** (`objective`, `next_action`, `facts`, `files`, `verification`) — descartado: elimina `status`, y sin `status` un `next_action` nulo es **ambiguo**, porque no distingue «no hay siguiente paso porque el objetivo está terminado» de «el agente paró sin dejar ninguno» — que es exactamente lo que el hook `Stop` existe para detectar. `blocked` tampoco es reducible a `active`: una sesión fresca necesita saber si puede continuar o si espera algo externo. Cuesta un token.
- **Serializar dinámicamente también el núcleo** — descartado: ahorra unos pocos tokens más a cambio de que el modelo empiece cada sesión sin ver el esqueleto y de volver ambigua la ausencia de `next_action`.
- **Conservar `phase` y `acceptance_criteria` en el nivel extendido** por compatibilidad — descartado: el kit todavía no tiene usuarios, y `schema_version` existe precisamente para gestionar esa migración cuando los tenga.
- **No recortar y dejar que la instrumentación decida** — descartado: mediría lo que no importa. La telemetría capta tokens, no la tendencia a rellenar campos vacíos.
