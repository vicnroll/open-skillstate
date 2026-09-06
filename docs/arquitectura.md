# Arquitectura de OpenSkillState

Qué es el sistema, cómo funciona y qué garantiza. Las decisiones individuales y su porqué viven en [`docs/adr/`](./adr/); este documento describe el conjunto coherente que forman.

| | |
|---|---|
| **Base** | arXiv:2608.26263v3 (Badhe, Tiwari, Chung) |
| **Estado** | Diseño v1 cerrado, sin implementar |
| **Decisiones** | 20 ADR en [`docs/adr/`](./adr/) |
| **Verificación** | Comportamiento relevante de Claude Code y Git contrastado contra documentación oficial el 2026-09-06 |

---

## Qué es

`skstate` mantiene el **estado de ejecución** de un agente de código como un dato estructurado y validado en lugar de depender de historial conversacional acumulado. El agente deja de reescribir el estado a mano y pasa a proponer parches que un runtime determinista normaliza, fusiona, canonicaliza, valida y escribe.

Se distribuye como un binario compilado sin dependencias de runtime, acompañado de una skill que enseña al agente a usarlo ([ADR 0001](./adr/0001-cli-como-binario-compilado.md), [ADR 0009](./adr/0009-interfaz-y-garantia-portable.md)). OpenSkillState es agnóstico del cliente y del orquestador: una CLI, un MCP, un IDE o un runtime externo pueden consumir el mismo estado sin que el núcleo conozca al consumidor.

---

## Los dos planos

El patrón del paper combina dos mecanismos con costes y grados de control distintos:

```text
Σt+1 = Σt ⊕ ΔΣt
```

| Plano | Qué compra | Qué implementa OpenSkillState |
|---|---|---|
| **Estado** | precisión, reanudabilidad y rechazo de estado mal formado | esquema, validación, Merge Patch, canonicalización, persistencia, promoción |
| **Contexto** | evitar que cada paso cargue toda la trayectoria anterior | OpenSkillState aporta Σ; el runtime decide qué más entra en el prompt |

El contraste conceptual es:

```text
Runtime conversacional                  Ejecución basada en estado

paso 1  [ P + O₁ ]                       paso 1  [ P + Σ₁ + O₁ ]
paso 2  [ P + O₁R₁a₁ + O₂ ]              paso 2  [ P + Σ₂ + O₂ ]
paso 3  [ P + O₁R₁a₁ + O₂R₂a₂ + O₃ ]     paso 3  [ P + Σ₃ + O₃ ]

        ^ arrastra trayectoria                    ^ no necesita trayectoria previa
```

El plano de estado se implementa completo y es portable. El de contexto **no puede imponerse desde el repositorio**: una sesión interactiva puede mantener su transcripción y un cliente puede cargar memoria, instrucciones u otras fuentes propias.

Una ejecución fresca lanzada por un runtime externo sí puede eliminar la **dependencia de la transcripción anterior**, pero eso no significa que Σ sea literalmente lo único en el prompt. Por ejemplo, Claude Code carga Auto Memory al iniciar conversaciones y la comparte entre worktrees del mismo repositorio. OpenSkillState no la desactiva por defecto: es contexto externo al producto, potencialmente complementario o redundante con Σ ([ADR 0009](./adr/0009-interfaz-y-garantia-portable.md)).

---

## El plano de estado

### El CLI posee el esquema

El runtime es la autoridad sobre la forma persistida. El modelo emite `ΔΣ`; el CLI lo procesa y escribe de forma atómica.

El parche entra por **stdin**:

```bash
skstate patch --stdin <<'EOF'
{ "facts": { "auth-en-middleware": "Auth se aplica en middleware/auth.go, no en los handlers" } }
EOF
```

El CLI no acepta el contrato `{"state_patch", "action"}` del apéndice A.4 del paper como input propio. `action` pertenece al harness del modelo si ese harness la necesita; OpenSkillState sólo recibe el parche ([ADR 0004](./adr/0004-el-cli-acepta-solo-el-parche.md)).

### RFC 7386 y colecciones con clave

`⊕` es **RFC 7386 JSON Merge Patch sin extensiones**. Como el estándar reemplaza arrays enteros, las colecciones se representan como objetos keyed ([ADR 0002](./adr/0002-listas-como-objetos-con-clave-semantica.md)):

```jsonc
{ "facts": { "watcher-500ms": "El watcher va 500 ms por detrás de las escrituras" } }
{ "facts": { "watcher-500ms": null } }
```

Las claves son slugs derivados del contenido, **1 a 4 palabras**, kebab-case, `^[a-z0-9]+(-[a-z0-9]+){0,3}$`. Una palabra es válida cuando ya expresa el concepto (`readme`, `arquitectura`).

El CLI normaliza tipografía mecánica — minúsculas, `_` a `-`, acentos y guiones — y rechaza sólo lo que exige juicio, como superar cuatro palabras. Si dos claves de un mismo parche canonicalizan al mismo slug, rechaza el parche entero.

### Canonicalización después del merge

RFC 7386 interpreta `null` como eliminación de propiedad. Sin una segunda fase, un parche correcto como `{"next_action": null}` eliminaría físicamente parte del esqueleto core.

La tubería de `patch` queda fijada por [ADR 0003](./adr/0003-esquema-por-niveles-y-serializacion-dinamica.md):

```text
normalizar slugs
→ detectar colisiones
→ RFC 7386
→ canonicalizar documento
→ recalcular derivados
→ validar esquema
→ escritura atómica
```

La canonicalización restaura la representación vacía de campos core no requeridos:

```text
objective, next_action     → null
facts, decisions           → {}
files                      → {relevant:{}, modified:{}}
verification               → {checks:{}, overall:"not_run"}
```

`status`, `mode`, `schema_version` y `project_id` son requeridos: intentar eliminarlos falla; el runtime no inventa un estado operativo por defecto.

### El esquema y sus niveles

| Nivel | Campos | `get` |
|---|---|---|
| **Core** | `status`, `mode`, `objective`, `next_action`, `facts`, `decisions`, `files`, `verification` | **Siempre**, incluso vacíos |
| **Extended** | `hypotheses`, `blockers`, `constraints` | Sólo con contenido |
| **Meta** | `schema_version`, `project_id`, `project` | Nunca |

`skstate get` es una **vista**, no un `cat`: emite JSON compacto en una línea siguiendo `x-tier`. `--raw` devuelve el documento persistido y `--pretty` una vista indentada.

El formato se midió antes de elegirlo: JSON compacto ahorra aproximadamente lo mismo que YAML en el estado de referencia, pero mantiene una sola sintaxis entre lectura y escritura. La mayor parte de Σ es contenido humano, no puntuación JSON; el mecanismo importante es evitar contenido innecesario, no pelear por unos pocos separadores.

El esquema no se instala en el repositorio. El binario embebe todas las versiones que conoce y aplica la indicada por `schema_version` ([ADR 0015](./adr/0015-el-esquema-vive-dentro-del-binario.md)). `skstate schema` permite auditarlo o alimentar otra interfaz, por ejemplo MCP.

### Identidad estable

`project` deja de ser identidad. `init` genera una vez un `project_id` UUID v4 y lo hace inmutable ([ADR 0019](./adr/0019-identidad-estable-del-proyecto.md)):

```json
{
  "project_id": "550e8400-e29b-41d4-a716-446655440000",
  "project": "open-skillstate"
}
```

`project` es un nombre mutable. El histórico se relaciona por `project_id`, por lo que mover o renombrar el repositorio no parte la serie y dos proyectos homónimos no colisionan.

Como el ID debe ser único, `init` **construye** el estado inicial; ya no copia un `state.json` estático desde `skills/`.

### Verificación derivada

`verification.checks` pertenece al agente y es promovible. `verification.overall` pertenece al runtime y se deriva de los checks ([ADR 0006](./adr/0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md)):

```text
sin checks / todos not_run → not_run
algún failed               → failed
todos passed               → passed
passed + not_run           → partial
```

Un parche que intente escribir `verification.overall` se rechaza. Así no existen dos fuentes de verdad para el agregado.

---

## Garantías por nivel

La propiedad de «el runtime valida lo que persiste» no se puede imponer igual en todos los clientes:

| Nivel | Mecanismo | Dónde |
|---|---|---|
| **Prevención** | regla de permisos que bloquea la edición directa | Donde el cliente lo soporte; Claude Code en v1 |
| **Detección/validación** | hash externo + validación cuando cambia fuera del CLI | Portable |
| **Validación previa** | JSON Schema del input de herramienta | Donde exista MCP u otra tool tipada |

En Claude Code `init` instala:

```json
{
  "permissions": {
    "deny": ["Edit(/.openskillstate/**)"]
  }
}
```

La documentación actual indica que los permisos por path se consultan mediante `Edit(path)` y `Read(path)`, no `Write(path)`. La `/` inicial ancla la regla al *primary working directory* de los project settings, por lo que también funciona correctamente dentro de un worktree.

La prevención no es una frontera contra procesos arbitrarios. El threat model sigue siendo un agente tomando un atajo, no un adversario. Si `state.json` cambia fuera del CLI, la siguiente operación compara el hash:

- si el documento sigue validando, avisa y restablece la línea base;
- si no valida, se niega a continuar sobre él.

Eso **detecta corrupción, no procedencia**. Un cambio manual bien formado no puede distinguirse de un `git checkout` legítimo sin acoplar el producto a una historia de Git que tampoco sería concluyente ([ADR 0009](./adr/0009-interfaz-y-garantia-portable.md)).

### Hooks de Claude Code

`SessionStart` inyecta las instrucciones de continuidad y `skstate get` por stdout con código 0.

`Stop` comprueba que un estado `active` en modo `execution` deje un `next_action` concreto. Bloquea sólo el primer intento y cede si `stop_hook_active` indica reentrada ([ADR 0013](./adr/0013-el-hook-stop-empuja-una-vez-y-cede.md)). La reanudabilidad se **fomenta**, no se promete como barrera absoluta.

---

## Ejecución aislada y orquestación

OpenSkillState no conoce Syntony, Orca ni ningún orquestador. El patrón portable es simplemente:

```text
   Σ consolidado ── .openskillstate/state.json
        │
        ├── entorno A ── worker-state.json ──┐
        ├── entorno B ── worker-state.json ──┤── skstate merge ──> promoción
        └── entorno C ── worker-state.json ──┘
```

El aislamiento puede ser un worktree, contenedor, copia del repositorio u otro mecanismo. Cada worker se declara una sola vez con `skstate init --worker` y hereda el `project_id` del proyecto ([ADR 0016](./adr/0016-el-rol-de-worker-se-declara-al-crearlo.md)).

### `merge` promueve, no une

Sólo se promocionan los nodos `x-promote`:

- `facts`
- `decisions`
- `constraints`
- `files.modified`
- `verification.checks`

`objective`, `next_action`, `status`, `mode`, `blockers`, `hypotheses` y `files.relevant` mueren con la ejecución local. `verification.overall` se recalcula en el destino.

Ante una misma clave con valores distintos, `merge` falla y lista el conflicto. El juicio queda fuera del runtime: una persona o agente decide y escribe el resultado mediante un parche explícito.

### Git transporta estado, no lo fusiona

`worker-state.json` es efímero e ignorado. `state.json` sí se versiona, pero [ADR 0008](./adr/0008-politica-de-git-para-el-estado.md) prohíbe que Git intente fusionarlo textualmente:

```gitattributes
.openskillstate/state.json -merge
```

El atributo `merge` unset usa el driver binario integrado: conserva provisionalmente la versión actual y marca conflicto. No necesita un driver externo.

Esto cubre también ramas humanas normales. Durante un merge:

```bash
git show MERGE_HEAD:.openskillstate/state.json | skstate merge --stdin
git add .openskillstate/state.json
```

Git aporta la otra versión; `skstate` decide qué conocimiento puede sobrevivir según el esquema.

### Concurrencia

Con un fichero por ejecución aislada, la concurrencia deja de estar en el camino caliente. El único caso relevante son varias integraciones escribiendo simultáneamente el consolidado. Para el lee-modifica-escribe corto de `patch`/`merge` basta un cerrojo exclusivo de fichero; no se introduce CAS ni `state_version` ([ADR 0011](./adr/0011-cerrojo-de-fichero-en-lugar-de-compare-and-swap.md)).

---

## State budget

Las claves semánticas reducen duplicados, pero **no demuestran que Σ permanezca acotado**. Si el número de hechos útiles creciera linealmente con los pasos, el coste acumulado podría volver a aproximarse a O(T²) aunque la transcripción desapareciese.

Por eso la v1 incorpora el [ADR 0020](./adr/0020-presupuesto-de-estado.md). `skstate check` informa siempre de tamaño de la salida compacta y composición por colecciones:

```text
state: valid
state size: 14.2 KiB
facts: 87
  8.6 KiB
decisions: 23
  2.1 KiB
constraints: 12
  1.0 KiB
```

No existe un máximo global inventado. El usuario o CI puede declarar un presupuesto blando:

```bash
skstate check --budget-bytes 16384
```

Superarlo produce una advertencia, no invalida el estado ni dispara poda automática. El runtime mide; el agente decide qué información dejó de ser necesaria.

La arquitectura puede decir que el patrón permite coste O(T) **si `|Σ|` permanece acotado**, pero v1 no vende esa condición como garantía mecánica. La instrumenta desde el primer día para poder justificar más adelante un budget recomendado con evidencia.

---

## Superficie del CLI

| Comando | Qué hace |
|---|---|
| `get` | Renderiza Σ compacto según tiers. `--pretty` indenta; `--raw` devuelve el documento |
| `patch` | Aplica RFC 7386, canonicaliza, recalcula derivados, valida y escribe |
| `check` | Valida, comprueba integridad/versión y diagnostica tamaño. `--budget-bytes N` añade budget blando |
| `init` | Inicializa proyecto e instala integraciones. `--worker` declara un estado aislado |
| `uninstall` | Retira skills/hooks/instrucciones, conservando workspace e identidad |
| `deinit` | Retira el proyecto OpenSkillState y su workspace; el histórico sólo con `--purge-history` explícito |
| `merge` | Promueve desde un workspace/estado origen; `--stdin` acepta un estado completo |
| `migrate` | Migra explícitamente entre versiones, con backup |
| `schema` | Imprime el esquema embebido; `--version N` para uno anterior |

`history` y `hook` son superficies especiales. `history` no aparece en el help ni en la skill de trabajo; `hook` es invocado por el cliente (`skstate hook stop`, `skstate hook session-start`).

### `init`, `uninstall` y `deinit`

[ADR 0012](./adr/0012-init-completa-la-instalacion.md) separa integración de datos.

`init` crea el estado con UUID propio, instala skills, entradas de `.gitignore` y `.gitattributes`, regla/hooks de Claude y bloque de instrucciones. Cada modificación compartida se registra en `.openskillstate/installed.json`.

`uninstall` quita las integraciones de cliente, pero **conserva** `state.json`, `project_id` y la protección Git del estado. Permite cambiar de cliente o reinstalar más tarde sin perder continuidad.

`deinit` es destructivo: retira integración, política Git y workspace. En headless exige `--yes`. No borra el histórico local salvo `--purge-history`; un `init` posterior generará otro `project_id`.

---

## Migraciones

Las migraciones son explícitas ([ADR 0014](./adr/0014-migracion-explicita-de-schema-version.md)):

- un estado más nuevo que el binario se rechaza;
- uno más antiguo puede seguir operando mientras esa versión esté en la ventana de operación;
- `migrate` cambia la versión y deja backup;
- los saltos se implementan consecutivamente (`1→2`, `2→3`...), no como convertidores N→latest;
- la ventana de migración es indefinida aunque la de operación pueda cerrarse.

`project_id` se conserva a través de toda migración.

---

## Instrumentación e histórico

[ADR 0010](./adr/0010-se-instrumenta-el-estado-no-los-tokens.md) mide aquello que OpenSkillState puede observar portably: tamaño, uso de campos, casi duplicados y conflictos. No depende de telemetría de tokens de un cliente concreto.

El histórico vive en SQLite fuera del repositorio ([ADR 0017](./adr/0017-historico-local-inalcanzable-desde-el-agente.md)):

```text
   skstate ──escribe──▶ SQLite local ◀──lee── skstate history
```

El flujo normal del agente de trabajo **no lo recibe ni lo inyecta**. Eso es una política de no exposición, no una frontera de seguridad: un agente con shell podría descubrir un subcomando oculto, pero la skill, hooks, help y herramientas normales no lo publican.

Cada evento se relaciona por `project_id`, no por path ni nombre. El almacén puede contener contenido del proyecto y sobrevivir a borrar el repo, por lo que `history purge` y una separación futura entre exportar métricas y exportar contenido forman parte del diseño.

---

## Límites

**La disciplina de estado no es universalmente buena.** Depuración, auditoría o exploración pueden necesitar observaciones cuya relevancia sólo aparece más tarde. `mode: exploration` relaja la presión por dejar un `next_action` y estructurar prematuramente.

**La compactación no implementa el plano de contexto.** Resumir con pérdida para alcanzar un tamaño reintroduce riesgo de borrar precisamente lo relevante.

**OpenSkillState no monopoliza el contexto del cliente.** Puede sustituir la dependencia de la transcripción previa por Σ en ejecuciones frescas, pero memoria automática, instrucciones o contexto propio del cliente pueden coexistir.

**Σ no está matemáticamente acotado por el esquema.** V1 evita varias formas de crecimiento accidental y expone un state budget, pero el contenido útil todavía puede crecer. Esa condición se mide, no se oculta detrás de la palabra «bounded».

---

## Estado de verificación

Las afirmaciones que sostienen decisiones específicas de Claude Code se contrastaron contra documentación oficial; las de Git contra `gitattributes`.

| # | Afirmación | Veredicto | Consecuencia |
|---|---|---|---|
| 1 | Los permisos de path para escritura se consultan mediante `Edit(path)`, no `Write(path)` | **Confirmada, literal** | La regla instalada usa sólo `Edit` |
| 2 | `/path` en project settings se ancla al primary working directory | **Confirmada** | `Edit(/.openskillstate/**)` protege el workspace correcto también en worktrees |
| 3 | Bash y redirecciones reconocidas se someten a permisos; procesos arbitrarios no ofrecen la misma garantía | **Confirmada con límites documentados** | La prevención sigue siendo adicional, no universal |
| 4 | `SessionStart` puede inyectar contexto vía stdout | **Confirmada** | El hook usa stdout y código 0 |
| 5 | `Stop` tiene reentrada y límite de bloqueos | **Confirmada** | Empuja una vez y cede |
| 6 | `PostToolUse` no deshace acciones ya ejecutadas | **Confirmada** | No se usa como barrera de persistencia |
| 7 | Los subagentes normales tienen contexto aislado; un `fork` hereda el padre | **Confirmada** | Un consumidor que busque frescura debe evitar forks |
| 8 | Auto Memory se carga al inicio y se comparte entre worktrees del mismo repo | **Confirmada** | No afirmar que Σ es literalmente todo el contexto; `init` no la desactiva |
| 9 | `merge` unset en `.gitattributes` conserva nuestra versión y marca conflicto | **Confirmada, literal** | `state.json -merge` evita fusiones textuales silenciosas sin driver externo |

---

## Fuentes

Consultadas o revalidadas el 2026-09-06.

1. **SKILL.state: Scalable Long-Horizon Agent Skills** — Badhe, Tiwari, Chung.
   <https://arxiv.org/html/2608.26263v3>

2. **Claude Code — Hooks**.
   <https://code.claude.com/docs/en/hooks>
   <https://code.claude.com/docs/en/hooks-guide>

3. **Claude Code — Permissions** — reglas `Edit(path)`, anclaje de `/path`, Bash y redirecciones.
   <https://code.claude.com/docs/en/permissions>

4. **Claude Code — Memory** — Auto Memory, carga inicial y compartición entre worktrees.
   <https://code.claude.com/docs/en/memory>

5. **Claude Code — Subagents**.
   <https://code.claude.com/docs/en/sub-agents>

6. **RFC 7386 — JSON Merge Patch**.
   <https://datatracker.ietf.org/doc/html/rfc7386>

7. **Git — gitattributes** — semántica del atributo `merge` unset.
   <https://git-scm.com/docs/gitattributes>
