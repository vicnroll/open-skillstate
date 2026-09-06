---
status: accepted
---

# El histórico vive fuera del repositorio y no se expone al agente de trabajo

El [ADR 0010](./0010-se-instrumenta-el-estado-no-los-tokens.md) decidió qué medir pero no dónde ponerlo. Las preguntas de instrumentación son transversales — crecimiento de Σ, uso de campos, conflictos de `merge` — y se contestan mejor cruzando proyectos que creando un fichero de métricas independiente en cada repositorio.

Además, auditar pasos, decisiones y acciones para aprender de ellos es compatible con SKILL.state siempre que esa cronología **no vuelva a entrar automáticamente en el contexto del agente que ejecuta**.

Se decide un **almacén local en SQLite, fuera del repositorio**, escrito de forma transparente por el binario en cada operación y consultable mediante `skstate history`, un subcomando deliberadamente fuera de la superficie normal del agente de trabajo.

## Guardar historia no contradice el patrón

El O(T²) que el patrón combate es un problema de *contexto*, no de *almacenamiento*. Una cronología en disco cuesta cero tokens mientras no se inyecte en cada paso.

La propiedad que OpenSkillState necesita proteger es:

> El flujo normal del agente de trabajo **no recibe, no enlaza y no inyecta** el histórico. `get`, `patch`, `check`, los hooks y la skill de trabajo operan sobre Σ actual, nunca reconstruyen conversación pasada desde SQLite.

Si alguien propusiera que `get` incluya «las últimas decisiones» o que `SessionStart` recupere automáticamente pasos anteriores, este ADR es la razón para rechazarlo: sería reintroducir por la puerta de atrás el canal cronológico que el producto separa.

## No es una frontera de seguridad

La formulación inicial decía que el histórico era «inalcanzable» desde el agente. Es demasiado fuerte. `skstate history` vive en el mismo binario y un agente con acceso general a shell podría descubrir o adivinar el comando.

La garantía real es de **no exposición y no inyección**, no de aislamiento de seguridad:

- `skstate --help` no anuncia `history`;
- la skill de trabajo y sus referencias no lo mencionan;
- ningún hook lo consulta;
- ningún comando normal devuelve ni resume datos históricos;
- no existe una herramienta MCP de trabajo que lo publique.

Esto cubre el riesgo relevante — contaminar accidentalmente el contexto operativo — sin fingir que ocultar un subcomando es un sandbox.

## La skill analista vive en otro ámbito

Se quiere permitir un **agente analista dedicado** que explote el histórico. Su skill se instala explícitamente a nivel de usuario o en un workspace de análisis, nunca dentro del repositorio del proyecto.

Una skill vecina que anunciase «puedo consultar todo lo registrado» competiría con la skill de trabajo y podría ser invocada precisamente cuando el agente quisiera recuperar contexto previo. Las dos responsabilidades deben permanecer separadas.

`skstate init` normal no instala esa skill. Una instalación específica como `skstate init --history-skill` puede hacerlo en un ámbito no asociado al proyecto operativo.

## Identidad del proyecto

El histórico **no se agrupa por ruta ni por nombre**. Cada evento lleva el `project_id` estable definido en el [ADR 0019](./0019-identidad-estable-del-proyecto.md); `project` es sólo una etiqueta de presentación.

Esto evita tres fallos de la propuesta anterior:

- mover el repositorio no parte la serie;
- renombrarlo no crea otra serie;
- dos repositorios llamados igual no mezclan eventos.

Los workers heredan el mismo `project_id`, de modo que la instrumentación de ejecuciones paralelas sigue perteneciendo al mismo proyecto sin depender del mecanismo de aislamiento.

## Decisiones técnicas

- **SQLite en Go puro** (`modernc.org/sqlite`). Un driver que exija cgo rompería la distribución estática y compilación cruzada del [ADR 0001](./0001-cli-como-binario-compilado.md).
- **WAL**, porque varios procesos pueden registrar eventos al mismo tiempo aunque sus Σ estén aislados.
- **Eventos compatibles conceptualmente con OTel**: timestamp, operación y atributos escalares. Eso no implica exportar contenido ni acoplarse a OTLP en v1.
- **`skstate history <subcomando>`**, no una bandera de `skstate`; es una superficie distinta con su propio propósito.

## Contenido sensible y retención

Guardar parches, objetivos, decisiones y acciones significa guardar **contenido del proyecto**, no sólo contadores. El almacén puede sobrevivir a borrar el repositorio y puede contener información que el usuario no esperaba conservar indefinidamente.

Por eso forman parte del diseño:

- `skstate history purge`, por `project_id` y por antigüedad;
- separación entre exportar **métricas** y exportar **contenido** cuando exista OTLP u otro backend;
- la recomendación de no guardar secretos en Σ, reforzada porque un secreto escrito una vez puede persistir también en el histórico hasta ser purgado.

`uninstall` y `deinit` no purgan historia por defecto ([ADR 0012](./0012-init-completa-la-instalacion.md)). `deinit --purge-history` puede hacerlo de forma explícita antes de destruir el workspace y perder su `project_id`.

## Consequences

- El histórico permite análisis longitudinal sin convertirlo en memoria automática del agente de trabajo.
- La garantía se expresa correctamente como **no exposición**, no como inaccesibilidad física.
- La identidad estable permite consultas cruzadas sin depender de paths o nombres.
- Aparece un segundo esquema local, el de SQLite, con un ciclo de vida distinto del esquema de Σ. Sus migraciones pueden ser más pragmáticas porque no forma parte del contrato portable del proyecto.
- El [ADR 0020](./0020-presupuesto-de-estado.md) puede usar el histórico para estudiar crecimiento real de Σ y justificar futuros budgets recomendados.

## Considered Options

- **Histórico dentro de `.openskillstate/`** — descartado: dificulta análisis cruzado y lo acerca innecesariamente al contexto del agente.
- **No guardar ninguna cronología** — descartado: elimina la posibilidad de evaluar decisiones del propio diseño sin mejorar el coste de contexto.
- **Binario separado como frontera física** — descartado: el objetivo es evitar exposición accidental, no impedir que un analista autorizado consulte los datos.
- **Usar `project` como clave** — descartado y sustituido por `project_id` en el ADR 0019.
