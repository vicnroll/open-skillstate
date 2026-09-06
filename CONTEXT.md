# OpenSkillState

Vocabulario del proyecto. Los cuatro primeros términos son la separación de identidad que sostiene todo lo demás: confundirlos es lo que hace que un lector no sepa si algo lo define el paper o lo decidimos nosotros.

## Identidad

**SKILL.state**:
El **patrón**, definido en el paper arXiv:2608.26263v3. Es arquitectura, no producto: `Σt+1 = Σt ⊕ ΔΣt`, el esquema fijo, la reconstrucción del prompt por paso. No pertenece a nadie y no dicta ninguna disposición de ficheros.
_Avoid_: usarlo para referirse a esta implementación, ni a la de nadie más.

**OpenSkillState**:
**Nuestra implementación** del patrón, con decisiones propias y extensiones donde el modelo del paper no encaja con el uso real de agentes de código — colecciones con clave, `mode`, jerarquía de orquestación, rechazo de parte del contrato A.4. Nombre humano del proyecto; el repositorio es `open-skillstate`.
_Avoid_: skillstate (es otro producto, publicado en npm por terceros), kit (condiciona el producto a ser una colección de herramientas cuando puede crecer a MCP, SDK o daemon).

**skstate**:
La **CLI** de OpenSkillState, y su nombre de binario. Una interfaz del producto, no el producto entero: el servidor MCP y los SDK futuros son otras.
_Avoid_: skillstate (colisiona en el `PATH` con el binario homónimo ya publicado), os (se lee como *operating system*).

**`.openskillstate/`**:
El **workspace** del producto dentro de un repositorio. Es un namespace, no «la carpeta donde vive `state.json`»: puede alojar lo que otras interfaces del producto necesiten persistir.
_Avoid_: `.skillstate/` (lo escribe también la otra implementación, y sugeriría falsamente que el directorio pertenece al patrón y que dos herramientas pueden compartirlo).

## Estado

**Σ**:
El estado de ejecución tal como llega al prompt, producido por `skstate get`. No es el fichero: `state.json` almacena el esquema completo y Σ es la vista que omite lo vacío.
_Avoid_: state.json (es el almacén, no la vista), contexto (Σ es lo que sustituye al contexto conversacional, no una parte de él)

**Campo de núcleo**:
Campo del esquema que se serializa en Σ siempre, incluso vacío, porque el modelo necesita ver el esqueleto que mantiene o porque su ausencia sería ambigua.
_Avoid_: campo obligatorio (no lo es: puede estar vacío y seguir apareciendo)

**Campo extendido**:
Campo que el esquema valida igual que uno de núcleo, pero que sólo aparece en Σ cuando tiene contenido y que la skill menciona sin describir en detalle.
_Avoid_: campo opcional (todos lo son; lo que distingue al extendido es que no se serializa vacío)

**Campo meta**:
Campo que se valida y se almacena pero que `get` **no emite** — `schema_version` y `project`. Al modelo no le sirven, y serían bytes en todos los prompts.

**Histórico**:
El registro duradero de pasos, decisiones y acciones que OpenSkillState guarda **fuera del repositorio**, inalcanzable desde el agente que trabaja. No es Σ y no es parte del workspace.
_Avoid_: memoria (invita a creer que el agente puede consultarla, que es exactamente lo que no puede)
