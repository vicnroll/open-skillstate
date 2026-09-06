# Arquitectura de skillstate

Qué es el sistema, cómo funciona y qué garantiza. Las decisiones individuales y su porqué viven en [`docs/adr/`](./adr/); este documento describe el conjunto coherente que forman.

| | |
|---|---|
| **Base** | arXiv:2608.26263v3 (Badhe, Tiwari, Chung) |
| **Estado** | Diseño cerrado, sin implementar |
| **Decisiones** | 14 ADR en [`docs/adr/`](./adr/) |
| **Verificación** | Comportamiento de Claude Code contrastado contra la documentación oficial el 2026-09-05; ver [Estado de verificación](#estado-de-verificación) |

---

## Qué es

`skillstate` mantiene el **estado de ejecución** de un agente de código como un dato estructurado y validado en lugar de como historial conversacional acumulado. El agente deja de reescribir el estado a mano y pasa a proponer parches que un runtime determinista valida y aplica.

Se distribuye como un binario compilado sin dependencias de runtime, acompañado de una skill que enseña al agente a usarlo ([ADR 0001](./adr/0001-cli-como-binario-compilado.md), [ADR 0009](./adr/0009-interfaz-y-garantia-portable.md)).

---

## Los dos planos

El patrón del paper no es un mecanismo sino dos, con costes y grados de dificultad muy distintos. Separarlos es lo que hace viable implementarlo.

```text
Σt+1 = Σt ⊕ ΔΣt
```

| Plano | Qué compra | Evidencia en el paper | Qué se implementa |
|---|---|---|---|
| **Estado**<br>esquema, validación determinista, operador de fusión | Precisión y robustez | 0,53 → 0,98 ante ruido<br>0 pasos de recuperación | **Completo y portable** |
| **Contexto**<br>reconstrucción del prompt por paso, descarte de `Rt` | Coste de tokens | 16,2× menos tokens<br>O(T) frente a O(T²) | **Depende del cliente** |

Lo que cambia entre un runtime y otro no es el contenido de un paso, sino qué arista alimenta el siguiente:

```text
Runtime conversacional                   skillstate

paso 1  [ P + O₁ ]                       paso 1  [ P + Σ₁ + O₁ ]
paso 2  [ P + O₁R₁a₁ + O₂ ]              paso 2  [ P + Σ₂ + O₂ ]
paso 3  [ P + O₁R₁a₁ + O₂R₂a₂ + O₃ ]     paso 3  [ P + Σ₃ + O₃ ]

        ^ crece con el historial                 ^ tamaño acotado
          O(T²) acumulado                          O(T) acumulado
```

El plano de estado se implementa entero y funciona igual en cualquier cliente. El de contexto no se puede forzar desde un repositorio: una sesión interactiva acumula transcripción y nadie puede impedirlo desde fuera. Donde sí se cumple de verdad es **bajo orquestación**, porque un worker lanzado con `claude -p` arranca en frío y Σ es literalmente todo lo que recibe.

---

## El plano de estado

### El CLI posee el esquema

El paper es explícito sobre dónde vive la autoridad: *«schema ownership and validation reside in the deterministic runtime rather than the model, malformed outputs cannot corrupt persistent state»*. El modelo emite `ΔΣ`; el CLI valida, fusiona y escribe de forma atómica.

El parche se acepta por **stdin**, lo que evita que el modelo tenga que escapar un JSON anidado dentro de una cadena de shell:

```bash
skillstate patch --stdin <<'EOF'
{ "facts": { "auth-en-middleware": "Auth se aplica en middleware/auth.go, no en los handlers" } }
EOF
```

### El operador de fusión

`⊕` es **RFC 7386 (JSON Merge Patch) sin extensiones**: fusión recursiva de objetos y borrado por `null`. El estándar reemplaza los arrays enteros, lo que encarecería cada parche en los campos que más crecen, así que **las colecciones se modelan como objetos con clave** ([ADR 0002](./adr/0002-listas-como-objetos-con-clave-semantica.md)):

```jsonc
// Añadir un hecho — el parche completo, sin reenviar nada más
{ "facts": { "watcher-500ms": "El watcher va 500 ms por detrás de las escrituras" } }

// Borrar uno
{ "facts": { "watcher-500ms": null } }
```

Las claves son **slugs semánticos derivados del contenido**, generados por el modelo con una convención mecánica que el CLI valida (`^[a-z0-9]+(-[a-z0-9]+){0,3}$`). Eso hace la escritura idempotente — reescribir un hecho conocido sobrescribe su clave en vez de duplicarlo — y evita que dos escritores concurrentes colisionen salvo cuando escriben sobre lo mismo.

El CLI **no acepta** el contrato de dos claves del apéndice A.4 del paper. `{"state_patch", "action"}` es un contrato de salida del *modelo*, no de entrada del CLI: en cualquier arquitectura donde el modelo tenga sus propias herramientas, `action` duplica `next_action` y puede desincronizarse sin que nada lo detecte ([ADR 0004](./adr/0004-el-cli-acepta-solo-el-parche.md)).

### El esquema

Tres niveles, en lugar de un recorte ([ADR 0003](./adr/0003-esquema-por-niveles-y-serializacion-dinamica.md)):

| Nivel | Campos | Serialización |
|---|---|---|
| **Núcleo** | `status`, `mode`, `objective`, `next_action`, `facts`, `decisions`, `files`, `verification` | Siempre, aunque estén vacíos |
| **Extendido** | `hypotheses`, `blockers`, `constraints` | Sólo con contenido |
| **Metadatos** | `schema_version` | No cuenta como contenido |

`skillstate get` **es una función de renderizado, no un `cat`**: omite lo vacío (`null`, `[]`, `{}`, `""`; nunca `0` ni `false`). Un campo raro cuesta cero tokens cuando no se usa, y sobre todo deja de **invitar al modelo a rellenarlo**, que es el coste caro y el que ninguna telemetría capta. `skillstate get --raw` devuelve el fichero íntegro para depurar.

`status` es `idle` | `active` | `blocked` | `completed`. `mode` es `execution` | `exploration`, y son ortogonales: `active`+`exploration` es depurar en mitad de una tarea ([ADR 0007](./adr/0007-modo-de-operacion-dentro-del-estado.md)).

---

## Garantías, por nivel

La propiedad que el paper pide — *«malformed outputs cannot corrupt persistent state»* — no se puede cumplir igual en todos los clientes, y conviene decirlo sin adornos.

| Nivel | Mecanismo | Dónde |
|---|---|---|
| **Prevención** | `deny: ["Edit(./.skillstate/...)"]` impide que el modelo escriba el fichero a mano | Sólo Claude Code |
| **Detección** | El CLI guarda fuera del fichero un hash de lo último que escribió y lo compara en cada operación | **En todas partes** |
| **Validación previa** | JSON Schema del *tool input* validado por el cliente | Donde haya servidor MCP |

La prevención se apoya en una asimetría que la documentación de permisos afirma literalmente: las reglas `deny` alcanzan las herramientas integradas, los comandos bash reconocidos y los destinos de redirecciones, pero **no los subprocesos arbitrarios** — que es exactamente lo que el CLI es.

La detección basta porque **el modelo de amenaza no es un adversario, es un atajo**. Contra un adversario no serviría; contra un atajo sí, porque el atajo se toma justo cuando se supone que nadie mira. Cubre además un agujero que abre la propia skill: un script empaquetado en `scripts/` es un subproceso arbitrario y `deny` no lo alcanza.

```text
                ┌── Edit · sed · > fichero ──✗  bloqueado (sólo Claude Code)
   Modelo ──ΔΣ──┤
                └── skillstate patch ──→ valida esquema
                                         ⊕ RFC 7386 ──→ estado + hash de integridad
```

### El hook `Stop` empuja, no bloquea

Rechaza cerrar el turno si `status` sigue en `active` sin un `next_action` concreto. Pero el bloqueo está topado en ocho veces y existe `stop_hook_active` para detectar la reentrada, así que **empuja una vez y cede** ([ADR 0013](./adr/0013-el-hook-stop-empuja-una-vez-y-cede.md)). En `mode: exploration` no comprueba nada. La reanudabilidad es una propiedad *fomentada*, no garantizada.

### El hook `SessionStart`

Inyecta el procedimiento y la salida de `skillstate get` **por stdout con código 0**, que es la vía documentada. Convierte la reanudación en sesión nueva de instrucción a mecanismo: la sesión empieza literalmente siendo (P, Σ).

---

## Bajo orquestación

Un orquestador como [Syntony](https://github.com/vicnroll/syntony) despacha varios workers en paralelo, aislados en worktrees de Git. Ahí el plano de contexto se cumple de verdad, y aparecen problemas que el uso interactivo no tiene.

```text
   Orquestador ── Σ consolidado (versionado en Git)
        │
        ├── worktree A ── Σ efímero (ignorado)  ──┐
        ├── worktree B ── Σ efímero (ignorado)  ──┤── skillstate merge ──> promoción
        └── worktree C ── Σ efímero (ignorado)  ──┘
```

**Jerarquía** ([ADR 0005](./adr/0005-estado-jerarquico-bajo-orquestacion.md)) — cada worker escribe sólo su Σ; el orquestador sólo el suyo. Nadie comparte fichero, así que no hay contención.

**`merge` promueve, no une** ([ADR 0006](./adr/0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md)) — el Σ de un worker mezcla lo que muere con la tarea (`objective`, `next_action`, `status`) y lo que la sobrevive (`facts`, `decisions`, `files.modified`, `verification`). Sólo se promueve lo segundo, declarado por campo en el esquema. Ante clave idéntica con valor distinto, **falla y lista los conflictos**; el Integrator resuelve emitiendo un parche explícito, de modo que el juicio lo pone el modelo y la escritura sigue siendo determinista.

**Git no toca el estado de los workers** ([ADR 0008](./adr/0008-politica-de-git-para-el-estado.md)) — si se versionara, Git intentaría fusionar con su algoritmo de tres vías un fichero que `merge` ya sabe fusionar semánticamente, produciendo conflictos donde no los hay y, peor, fusiones limpias que son incorrectas. Los dos Σ necesitan nombres de fichero distintos, porque los worktrees comparten el `.gitignore`.

**Cerrojo, no *compare-and-swap*** ([ADR 0011](./adr/0011-cerrojo-de-fichero-en-lugar-de-compare-and-swap.md)) — con esta topología el único caso concurrente son varias integraciones simultáneas sobre el Σ del orquestador, y el `patch` del CLI es un lee-modifica-escribe de milisegundos. Con RFC 7386 el parche no depende de lo que se leyó, así que un cerrojo basta y se ahorra el campo, la bandera y el protocolo de reintento que el CAS exigiría al modelo.

---

## Superficie del CLI

| Comando | Qué hace |
|---|---|
| `get` | Renderiza Σ omitiendo lo vacío. `--raw` devuelve el fichero íntegro |
| `patch` | Aplica un parche RFC 7386 por stdin, validando contra el esquema |
| `check` | Valida el estado, detecta desajuste de `schema_version` y escritura directa |
| `init` | Instala el kit en el repositorio |
| `merge` | Promueve el Σ de un worker al del orquestador |
| `migrate` | Migra el estado entre versiones de esquema, con copia de seguridad |

`init` **completa la instalación**, incluida la sección en `CLAUDE.md`/`AGENTS.md` dentro de un bloque delimitado e idempotente ([ADR 0012](./adr/0012-init-completa-la-instalacion.md)). Con TTY pregunta; sin TTY exige `--write-instructions` o `--no-write-instructions` y falla si no recibe ninguna, porque un `init` headless que decide en silencio es peor que uno que se detiene diciendo qué falta.

`migrate` es siempre explícito, y un `schema_version` más nuevo que el binario se rechaza siempre ([ADR 0014](./adr/0014-migracion-explicita-de-schema-version.md)).

---

## Instrumentación

Se mide **el estado, no los tokens** ([ADR 0010](./adr/0010-se-instrumenta-el-estado-no-los-tokens.md)): tamaño de Σ en el tiempo, qué campos se usan de verdad, cuántos casi-duplicados aparecen, con qué frecuencia falla `merge`. Lo emite el propio CLI, sin telemetría del cliente ni corpus de tareas.

El criterio es cuál de las dos mediciones puede **cambiar una decisión ya tomada**. El ahorro de tokens del paper no está en duda y reproducirlo no alteraría nada de lo diseñado. En cambio la promoción por campo del ADR 0006 no distingue hechos durables de hechos locales dentro de `facts`, y nadie sabe todavía si eso resulta ruidoso.

---

## Límites

**No hay disciplina de estado universalmente buena.** La sección 7 del paper reconoce tres escenarios donde su premisa falla: sin esquema fijo conocido de antemano, cuando la relevancia de una observación anterior no se reconoció en su momento, y cuando el objetivo se define sobre la trayectoria misma. Depurar, auditar y explorar caen ahí, y forzar la disciplina **empeora** el resultado. Para eso existe `mode: exploration`.

**La compactación no es una implementación del plano de contexto.** Un resumen con pérdida reintroduce exactamente el envenenamiento que el patrón ataca. Es lo que ocurre cuando el plano de contexto falla, no una forma de conseguirlo.

**El plano de contexto no se puede forzar en una sesión interactiva.** Ninguna skill ni hook puede obligar a un cliente a descartar su transcripción. Bajo orquestación sí se cumple, porque cada worker arranca en frío — y ahí la pérdida de relevancia retroactiva pasa a ser un riesgo real, no teórico.

---

## Estado de verificación

Las afirmaciones sobre el comportamiento de Claude Code sostienen decisiones de ingeniería, así que se contrastaron una a una contra la documentación oficial el 2026-09-05.

| # | Afirmación | Veredicto | Consecuencia |
|---|---|---|---|
| 1 | `deny` alcanza herramientas integradas, comandos bash reconocidos y redirecciones, pero no subprocesos arbitrarios | **Confirmada, literal** | La prevención se sostiene sobre texto citable |
| 2 | Orden de evaluación `deny` → `ask` → `allow`, primera coincidencia decide | **Confirmada** | Un `allow` posterior no puede ablandar el `deny` |
| 3 | `deny` tiene precedencia sobre los **hooks** y aplica sin *workspace trust* | **No documentada** | Retirada como argumento; nada depende de ella |
| 4 | `SessionStart` inyecta contexto | **Confirmada vía stdout**; `additionalContext` por evento, no documentado | Se implementa con stdout y salida 0 |
| 5 | `Stop` puede bloquear con salida 2 | **Matizada**: tope de 8 bloqueos y campo `stop_hook_active` | Empuja una vez y cede |
| 6 | `PostToolUse` no puede deshacer la acción | **Confirmada, literal** | La receta de Codex CLI no se traslada |
| 7 | `PreCompact` es informativo y no bloquea | **No documentada** | Suposición, no hecho; no se construye nada sobre ella |
| 8 | Subagentes aislados, sin historial, sólo devuelven su resumen | **Confirmada**, con excepción: un `fork` **sí** hereda el historial | Un orquestador no debe despachar forks |
| 9 | Cada subagente recarga CLAUDE.md y los skills precargados | **Confirmada** (salvo `Explore`/`Plan`); coste **no cuantificado** | Hay coste fijo por invocación, sin cifra publicada |
| 10 | `claude_code.token.usage` con `type`, `session.id` y `agent.name` | **Confirmada**, con límite: los subagentes propios colapsan a `"custom"` | Separa orquestador de subagentes, no unos de otros |

Cinco confirmadas (dos literales), tres matizadas con consecuencia directa sobre el diseño, dos no documentadas retiradas como argumento. Ninguna resultó falsa.

---

## Fuentes

Consultadas y verificadas el 2026-09-05. Las de Claude Code se citan separando **referencia** de **guía**: buena parte de los detalles operativos sólo aparecen en la guía, y consultar la página equivocada produce falsos «no documentado».

1. **SKILL.state: Scalable Long-Horizon Agent Skills** — Badhe, Tiwari, Chung. Formalismo `⊕`, contrato de dos claves del apéndice A.4, esquema de cinco campos, limitaciones de la sección 7.
   <https://arxiv.org/html/2608.26263v3>

2. **Claude Code — Hooks (referencia)** — Eventos, formato de salida JSON, códigos de salida.
   <https://code.claude.com/docs/en/hooks>

   **Hooks (guía)** — Inyección de contexto en `SessionStart` vía stdout, `stop_hook_active` y el tope de ocho bloqueos, imposibilidad de deshacer en `PostToolUse`.
   <https://code.claude.com/docs/en/hooks-guide>

3. **Claude Code — Subagents** — Aislamiento de contexto, qué recibe el subagente (CLAUDE.md salvo `Explore`/`Plan`), la excepción del tipo `fork`, el campo `skills:`.
   <https://code.claude.com/docs/en/sub-agents>

4. **Claude Code — Configure permissions** — Sintaxis `Edit(ruta)`, alcance sobre bash y redirecciones, exclusión de subprocesos arbitrarios, orden de evaluación.
   <https://code.claude.com/docs/en/permissions>

5. **Claude Code — Monitoring usage** — `claude_code.token.usage`, atributo `type`, `session.id`, `agent.name` con los subagentes propios colapsados a `"custom"`.
   <https://code.claude.com/docs/en/monitoring-usage>

6. **RFC 7386 — JSON Merge Patch** — Borrado por null, fusión recursiva, reemplazo completo de arrays, algoritmo de referencia.
   <https://datatracker.ietf.org/doc/html/rfc7386>

7. **SKILL.state: O(T) Agent Memory — Codex CLI** — Daniel Vaughan. Análisis previo del mismo traslado; origen de la receta `PostToolUse` que no aplica en Claude Code.
   <https://codex.danielvaughan.com/2026/08/29/skill-state-ot-agent-memory-structured-execution-state-codex-cli-long-horizon/>
