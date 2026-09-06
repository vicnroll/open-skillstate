---
status: accepted
---

# El Σ de los workers no se versiona; el Σ consolidado del orquestador sí

La jerarquía de [ADR 0005](./0005-estado-jerarquico-bajo-orquestacion.md) crea un problema que no existía: si el estado está bajo control de versiones y cada worker trabaja en su propio worktree, al integrar **Git intentaría fusionar con su algoritmo de tres vías un fichero que `skillstate merge` ya sabe fusionar semánticamente**. Git no entiende nada de claves semánticas, promoción por campo ni escalares que no se fusionan: produciría conflictos de texto donde no hay conflicto semántico y, peor, fusiones limpias a nivel de texto que son incorrectas a nivel de estado.

Se decide que **el Σ de cada worker es efímero y nunca se versiona**, y que **el Σ consolidado del orquestador sí**, porque es el que representa conocimiento durable del repositorio. La política de Git coincide así con una frontera que ya existía por otras razones: lo que se promueve en [ADR 0006](./0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md) es exactamente lo que merece historial.

El problema se evita de raíz: si el Σ de los workers nunca entra en Git, integrar worktrees no puede producir conflictos sobre él, y el único camino de fusión sigue siendo `skillstate merge`.

## Consequences

- **Los dos Σ necesitan nombres distintos.** Los worktrees comparten el `.gitignore` del repositorio, así que una misma ruta relativa no puede estar ignorada en el worktree y versionada en la copia principal. El estado efímero del worker y el consolidado del orquestador tienen que vivir en ficheros con nombres diferentes dentro de `.skillstate/`.
- **El Σ de un worker no sobrevive a su worktree.** No se puede inspeccionar desde otro sitio ni reconstruir después de borrarlo. Es aceptable porque lo que importa ya se promovió al integrar; lo que no se promovió era estado local de una tarea terminada.

## Considered Options

- **Ignorado siempre** — descartado: elimina el conflicto con Git, pero también el historial del conocimiento durable, que es justo lo que sí merece versionarse.
- **Versionado con un *merge driver* propio** (`.gitattributes` con `merge=skillstate`) — descartado pese a ser el mecanismo que Git prevé para este caso: un driver no registrado en un clon hace que Git caiga **en silencio** en la fusión textual. Un fallo silencioso hacia el modo incorrecto es exactamente lo que se ha evitado en todas las demás decisiones.
- **Decidirlo por proyecto** — descartado: no es una decisión, es no tomarla. Era defendible mientras no existía mecanismo de fusión; ahora existe, y su corrección depende de que Git no toque el fichero.
