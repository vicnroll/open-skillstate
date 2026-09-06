---
status: accepted
---

# Git versiona el Σ consolidado pero nunca lo fusiona semánticamente

La jerarquía de [ADR 0005](./0005-estado-jerarquico-bajo-orquestacion.md) crea dos clases de estado: el Σ efímero de una ejecución aislada y el Σ consolidado que representa continuidad durable del repositorio. Git es un buen transporte e historial para el segundo, pero **no entiende la semántica de OpenSkillState**: no conoce promoción por campo, claves semánticas ni campos derivados.

El problema no existe sólo bajo orquestación. Dos ramas humanas pueden modificar `.openskillstate/state.json` y un `git merge` puede producir una fusión textual limpia que sea incorrecta a nivel de estado. Por tanto la regla debe ser general y no depender de quién creó la rama o el worker.

Se deciden dos cosas complementarias:

1. **El estado efímero de una ejecución aislada (`worker-state.json`) nunca se versiona.**
2. **El estado consolidado (`state.json`) sí se versiona, pero Git no puede resolver automáticamente una divergencia entre dos versiones.** `init` añade esta regla al `.gitattributes` del repositorio:

```gitattributes
.openskillstate/state.json -merge
```

`merge` unset activa el driver binario integrado de Git: conserva provisionalmente la versión de la rama actual y **marca el fichero como conflicto** cuando ambos lados necesitan fusión. No hay driver externo que instalar ni fallback silencioso a una fusión textual.

## La resolución pertenece a `skstate merge`

Cuando otra línea de trabajo tiene conocimiento que debe sobrevivir, se promociona mediante la semántica del [ADR 0006](./0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md), no mediante Git.

`skstate merge` acepta como origen un workspace/estado aislado y también un estado completo por stdin. Eso permite resolver una divergencia Git sin acoplar el CLI al índice de Git:

```bash
# durante un merge de Git, state.json contiene provisionalmente nuestra versión
# y Git lo mantiene marcado como conflicto
git show MERGE_HEAD:.openskillstate/state.json | skstate merge --stdin
git add .openskillstate/state.json
```

El CLI recibe el estado del otro lado, promueve únicamente `facts`, `decisions`, `constraints`, `files.modified` y `verification.checks`, recalcula derivados y falla ante conflictos semánticos. El estado operativo de la rama destino (`objective`, `next_action`, `status`, `mode`) sigue perteneciendo a la rama destino.

Este protocolo funciona igual si el origen es un worktree, una copia del repositorio, un contenedor o una rama convencional. Un orquestador externo puede automatizarlo, pero OpenSkillState no presupone ninguno.

## Por qué los dos ficheros necesitan nombres distintos

Los worktrees de Git comparten `.gitignore`. Una misma ruta relativa no puede estar ignorada en un worktree y versionada en la copia principal. Por eso el estado efímero vive en `.openskillstate/worker-state.json` y el consolidado en `.openskillstate/state.json`.

## Consequences

- **Git transporta y conserva historial, pero no decide semántica.** La línea `-merge` convierte una fusión potencialmente incorrecta en un conflicto visible.
- **No hace falta un merge driver propio.** La solución depende sólo de una capacidad nativa de Git y sigue funcionando en cualquier clon.
- **Las ramas normales quedan cubiertas**, no sólo los workers de un orquestador. Si ambos lados cambian `state.json`, la resolución es explícita.
- **El Σ de un worker no sobrevive a su espacio aislado.** Lo que debe sobrevivir se promueve antes de destruirlo; lo demás era estado local de una ejecución terminada.
- **`init` y `uninstall` gestionan también `.gitattributes`.** La entrada que OpenSkillState inserta se registra en `installed.json` para poder retirarla sin tocar reglas ajenas ([ADR 0012](./0012-init-completa-la-instalacion.md)).

## Considered Options

- **Ignorar siempre `state.json`** — descartado: elimina conflictos, pero también la continuidad compartida y el historial del conocimiento durable.
- **Dejar el three-way merge textual por defecto** — descartado: puede producir conflictos donde no los hay y, peor, fusiones limpias semánticamente incorrectas.
- **Merge driver propio** (`merge=skstate`) — descartado: un driver registrado en configuración local no viaja de forma fiable con el repositorio; un clon sin él puede caer en un comportamiento distinto. `-merge` es nativo y falla ruidoso.
- **Resolver sólo el caso de worktrees/orquestadores** — descartado: dos ramas convencionales reproducen exactamente el mismo problema y OpenSkillState debe ser agnóstico del consumidor.
