# skill-state-kit

Vocabulario del proyecto que traslada el patrón SKILL.state (arXiv:2608.26263v3) a un kit adoptable por repositorios que usan agentes de código.

## Language

**Kit**:
El producto completo — CLI `skillstate` + skills de agente + esquema de estado — que se instala en un repositorio mediante un paquete distribuible (binario compilado, instalador universal, wrapper npm), no copiando ficheros de texto a mano.
_Avoid_: Plantilla, drop-in (describen el prototipo anterior de sólo texto, ya superado)

**Σ**:
El estado de ejecución tal como llega al prompt, producido por `skillstate get`. No es el fichero: `state.json` almacena el esquema completo y Σ es la vista que omite lo vacío.
_Avoid_: state.json (es el almacén, no la vista), contexto (Σ es lo que sustituye al contexto conversacional, no una parte de él)

**Campo de núcleo**:
Campo del esquema que se serializa en Σ siempre, incluso vacío, porque el modelo necesita ver el esqueleto que mantiene o porque su ausencia sería ambigua.
_Avoid_: campo obligatorio (no lo es: puede estar vacío y seguir apareciendo)

**Campo extendido**:
Campo que el esquema valida igual que uno de núcleo, pero que sólo aparece en Σ cuando tiene contenido y que el skill menciona sin describir en detalle.
_Avoid_: campo opcional (todos lo son; lo que distingue al extendido es que no se serializa vacío)
