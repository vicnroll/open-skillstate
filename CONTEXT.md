# OpenSkillState

Vocabulario del proyecto. Los cuatro primeros términos son la separación de identidad que sostiene todo lo demás: confundirlos hace que un lector no sepa si algo lo define el paper o lo decidimos nosotros.

## Identidad del producto

**SKILL.state**:
El **patrón**, definido en el paper arXiv:2608.26263v3. Es arquitectura, no producto: `Σt+1 = Σt ⊕ ΔΣt`, esquema fijo, validación determinista y reconstrucción del prompt por paso. No pertenece a nadie y no dicta ninguna disposición de ficheros.
_Avoid_: usarlo para referirse a esta implementación ni a la de nadie más.

**OpenSkillState**:
**Nuestra implementación** del patrón, con decisiones propias y extensiones donde el modelo del paper no encaja con el uso real de agentes de código — colecciones con clave, `mode`, jerarquía de ejecución, política Git, histórico separado y rechazo de parte del contrato A.4. Nombre humano del proyecto; el repositorio es `open-skillstate`.
_Avoid_: `skillstate` (es otro producto publicado por terceros), `kit` (condiciona el producto a una colección de herramientas cuando puede crecer a MCP, SDK o daemon).

**skstate**:
La **CLI** de OpenSkillState, y su nombre de binario. Una interfaz del producto, no el producto entero: un servidor MCP o SDK futuro serían otras interfaces.
_Avoid_: `skillstate` (colisiona con otro binario), `os` (se lee como *operating system*).

**`.openskillstate/`**:
El **workspace** del producto dentro de un repositorio. Es un namespace estable, no «la carpeta donde vive `state.json`»: puede alojar los datos que otras interfaces del producto necesiten persistir.
_Avoid_: `.skillstate/` (lo usa otra implementación y sugeriría falsamente que pertenece al patrón).

## Identidad del proyecto

**`project_id`**:
Identificador estable UUID v4 de una **instancia inicializada** de OpenSkillState. Lo genera `skstate init`, es inmutable, viaja en `state.json` y es la clave del histórico local. Workers y worktrees del mismo proyecto lo heredan.
_Avoid_: ruta del repositorio, remote Git o nombre humano como sustitutos de identidad.

**`project`**:
Nombre humano mutable del proyecto. Sirve para mostrarlo, no para identificarlo. Un rename cambia `project` y conserva `project_id`.
_Avoid_: usarlo como clave del histórico.

## Estado

**Σ**:
El estado de ejecución tal como llega al prompt, producido por `skstate get`. No es el fichero: `state.json` almacena el documento canónico completo y Σ es una vista según `x-tier`.
_Avoid_: `state.json` como sinónimo; «contexto» como sinónimo, porque el cliente puede añadir otras fuentes además de Σ.

**Campo de núcleo**:
Campo del esquema que se serializa en Σ **siempre, incluso vacío**, porque el modelo necesita ver el esqueleto que mantiene o porque su ausencia sería ambigua.
_Avoid_: «campo obligatorio»: algunos pueden contener su representación vacía aunque sigan emitiéndose.

**Campo extendido**:
Campo que valida igual que uno de núcleo, pero sólo aparece en Σ cuando tiene contenido y la skill describe por revelación progresiva.
_Avoid_: «campo opcional» como rasgo distintivo; lo importante es su política de renderizado.

**Campo meta**:
Campo que se valida y almacena pero que `get` **no emite** — `schema_version`, `project_id` y `project`.

**Campo derivado**:
Valor persistido para lectura/auditoría pero calculado por el runtime a partir de otros datos. En v1, `verification.overall`. El agente no lo parchea y `merge` no lo promociona.

**Canonicalización**:
Paso posterior a RFC 7386 que restaura la representación física canónica de los campos core eliminados por `null`, recalcula campos derivados y deja un único documento persistido antes de validar y escribir.

**State budget**:
Diagnóstico de tamaño y composición de Σ. En v1 es observable y puede compararse contra un budget blando explícito; no es un máximo universal ni una regla de validez.

## Ejecución aislada

**Σ consolidado**:
`.openskillstate/state.json`, versionado en Git. Representa la continuidad durable de la línea de trabajo que integra. Git lo transporta pero no lo fusiona semánticamente: `.gitattributes` lo marca `-merge`.

**Σ de worker**:
`.openskillstate/worker-state.json`, efímero e ignorado. Pertenece a una ejecución aislada, independientemente de que el aislamiento sea un worktree, contenedor, copia del repositorio u otro mecanismo.
_Avoid_: asociar «worker» con un orquestador concreto.

**Promoción**:
Operación de `skstate merge` que traslada sólo los nodos `x-promote` de un estado origen al consolidado. No es una unión de documentos ni un merge de Git.

## Histórico

**Histórico**:
Registro duradero local de eventos, métricas y contenido que OpenSkillState guarda **fuera del repositorio**. El flujo normal del agente de trabajo no lo recibe ni lo inyecta; `skstate history` existe para análisis explícito.
_Avoid_: «memoria» (invita a creer que forma parte del contexto operativo), «inalcanzable» (no es una frontera de seguridad; es una política de no exposición).
