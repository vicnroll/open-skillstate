---
status: accepted
---

# El histórico vive en un almacén local que el agente de trabajo no alcanza

El [ADR 0010](./0010-se-instrumenta-el-estado-no-los-tokens.md) decidió qué medir pero no dónde ponerlo, y al concretarlo aparecieron dos cosas que el diseño no había mirado.

La primera: **las preguntas que la instrumentación debe responder no son del usuario, son nuestras**. ¿Es ruidosa la promoción por campo del [ADR 0006](./0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md)? ¿Se usan de verdad los campos extendidos que el [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md) supone poco frecuentes? Eso se contesta meses después y **a través de varios proyectos**. Un fichero por repositorio que nadie agrega no lo responde: sólo mide los repositorios que ejecutemos nosotros.

La segunda: **querer auditar pasos, decisiones y acciones para aprender de ellos es un objetivo legítimo** que el diseño parecía prohibir, y no lo prohíbe.

Se decide un **almacén local en SQLite, fuera del repositorio**, escrito de forma transparente por el binario principal en cada operación, y legible sólo por `skillstate history`, un subcomando oculto.

## Por qué guardar historia no contradice el patrón

La objeción evidente es que el [ADR 0005](./0005-estado-jerarquico-bajo-orquestacion.md) sostiene que Σ nunca es un histórico, porque crecer con la cronología reintroduciría el O(T²) que el patrón elimina. Eso **sigue siendo verdad de Σ y deja de serlo de skillstate**, porque la objeción aplicaba mal la tesis del paper: el O(T²) no es un problema de *almacenamiento*, es un problema de *contexto*. Lo que hace daño no es que la cronología exista, es que **entre en el prompt**. Un histórico en disco que nunca llega al modelo cuesta exactamente cero tokens.

La propiedad que hay que garantizar, por tanto, no es «no guardar historia» sino **«la historia no es alcanzable desde el modelo que trabaja»**.

## El invariante

> El histórico es **de escritura por el binario principal y de lectura por otra herramienta**. Ningún comando de `skillstate` devuelve datos del histórico: ni completos, ni resumidos, ni agregados, ni como contexto. Ni el `SKILL.md`, ni `references/`, ni el fragmento de instrucciones, ni `--help` lo mencionan.
>
> Si alguna vez alguien propone que `get` incluya un resumen de lo anterior, o que el hook `SessionStart` inyecte las últimas decisiones, la respuesta es **no**, y este ADR es el motivo: sería reintroducir por la puerta de atrás exactamente el O(T²) que el producto elimina por la principal.

## Por qué el mismo binario basta, y dónde estaba el riesgo de verdad

Se consideró un binario aparte, `skillstate-history`, que no se instalase con el kit: lo que no está en la máquina no se ejecuta, y eso es una barrera física en vez de una convención. Se descartó porque **el objetivo no es impedir todo acceso**, sino impedir que se contamine el agente que trabaja — de hecho se quiere explícitamente que un **agente analista dedicado**, independiente del que trabaja en el proyecto, pueda explotar los datos. Con ese objetivo, que la skill de trabajo no lo mencione cubre el caso real, porque el agente sólo llegaría al histórico si supiera que existe.

El riesgo serio está en otro sitio y es el que obliga a una decisión: **la skill del analista no puede instalarse en el proyecto**. Claude Code carga todas las skills del repositorio, y la *descripción* de una skill es precisamente lo que hace que un agente la invoque por su cuenta — la nuestra dice literalmente *«invoke autonomously whenever durable execution continuity would help»*. Una skill vecina que anuncie «puedo consultar todo lo registrado» se invocaría justo cuando el agente creyera que le vendría bien contexto previo. Sería un final irónico: construir todo el aislamiento y luego instalar el atajo al lado.

Además las dos skills **se contradicen si se cargan juntas**: una dice que el estado describe el presente y que nunca hace falta reconstruir la transcripción anterior; la otra ofrece la cronología completa.

Por eso son **dos instalaciones explícitas en ámbitos distintos**. El `init` normal no toca la del analista; `skillstate init --history-skill` la instala a nivel de usuario o en el espacio de trabajo del analista, nunca en el repositorio del proyecto.

## Decisiones técnicas que van con esto

- **Driver de SQLite en Go puro** (`modernc.org/sqlite`). El driver más común exige cgo, que rompe el enlazado estático y la compilación cruzada — es decir, el [ADR 0001](./0001-cli-como-binario-compilado.md) entero. Si por algún motivo acabáramos necesitando cgo, la decisión correcta sería volver a un fichero plano, no romper el 0001.
- **WAL**, porque aquí la concurrencia sí es el caso normal: varios workers escriben métricas a la vez. Contrasta a propósito con el [ADR 0011](./0011-cerrojo-de-fichero-en-lugar-de-compare-and-swap.md), donde un cerrojo simple bastaba porque la concurrencia era rara.
- **El proyecto se identifica por nombre**, en un campo `project` de nivel `meta` en `state.json`. No depende de rutas, así que mover el repositorio no parte la serie, y renombrar es un parche corriente. El campo vive en el estado porque es el único fichero que el CLI ya posee: no hacen falta ficheros nuevos.
- **`skillstate history <subcomando>`**, no `--history` ni `-h`. `-h` es `--help` en prácticamente toda CLI que existe, y una bandera que enruta subcomandos es una forma equivocada.

## Lo que se asume con los ojos abiertos

Guardar pasos, decisiones y acciones significa guardar los parches, y los parches **son contenido del proyecto**: objetivos, hechos sobre el código, rutas, decisiones. El almacén deja de ser un registro de recuentos y pasa a ser **un archivo local y duradero de todo lo que skillstate ha visto**, en todos los proyectos de la máquina, que sobrevive a borrar el repositorio. Eso exige tres cosas que antes no hacían falta:

- **Retención y borrado explícitos** — `skillstate history purge`, por proyecto y por antigüedad. Un archivo que sólo crece y que nadie puede vaciar es un problema, no una función.
- **La exportación OTLP deja de ser inocua.** Exportar recuentos a un backend de observabilidad no tiene consecuencias; exportar contenido es sacar de la máquina las notas de diseño del código. Cuando llegue esa fase tienen que ser dos caminos distintos, y conviene dejarlo escrito antes de que alguien lo construya como uno solo.
- **El aviso de no guardar secretos sube de categoría.** Hoy es un consejo del skill y lo que se ensucia es Σ, que se corrige con un parche. Con archivo permanente, un secreto escrito por error se queda hasta que alguien lo purgue.

## Consequences

- **El ADR 0010 queda absorbido en parte.** Lo que decidió —medir el estado y no los tokens— sigue en pie; dónde va el dato lo responde este ADR, y la respuesta resulta ser mejor que un fichero por repositorio porque permite la consulta que cruza proyectos, que era justo la que el 0010 necesitaba.
- **Aparece un segundo esquema con su propio versionado**, el del almacén. Se tratan a propósito de forma distinta: los datos de Σ son preciosos y su migración es explícita ([ADR 0014](./0014-migracion-explicita-de-schema-version.md)); los del histórico no lo son, y si su esquema cambia se puede recrear la tabla. Decirlo evita que alguien construya para las métricas la maquinaria que sólo Σ merece.
- **La forma de la fila ya es compatible con OTel** — un evento con marca temporal, nombre de operación y atributos escalares — sin necesidad de diseñar para ello. Basta con no diseñar en contra.
