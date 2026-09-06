---
status: accepted
---

# La v1 usa un cerrojo de fichero, no *compare-and-swap*

El paper señala los escritores concurrentes como problema abierto — *«multi-agent environments introduce concurrent writes, requiring deterministic conflict-resolution semantics»* — y durante buena parte del diseño se dio por hecho que la respuesta era un contador `state_version` con *compare-and-swap*. Al fijar la topología de [ADR 0005](./0005-estado-jerarquico-bajo-orquestacion.md) esa premisa dejó de sostenerse: los workers están aislados por worktree, el orquestador es escritor único de su Σ, y una sesión interactiva es una sola. El único caso concurrente real son **varias integraciones en paralelo escribiendo el Σ del orquestador**.

Y para ese caso el CAS es la herramienta equivocada. El CAS resuelve el patrón «leo, trabajo cinco minutos, escribo», donde el estado pudo cambiar entre medias. Pero con RFC 7386 el parche **no depende de lo que se leyó**: declara «pon `facts.auth-en-middleware` a X», no «incrementa lo que hubiera». Para claves distintas es conmutativo, y el `patch` del CLI es un lee-modifica-escribe que dura milisegundos. Se decide un **cerrojo exclusivo de fichero** durante esa ventana.

## Consequences

- **No hace falta `state_version` en Σ**, ni una bandera `--expect-version`, ni un protocolo de reintento que el modelo tenga que implementar correctamente. Ese coste era real y el caso que lo justificaba ya no existe.
- **Es aditivo.** Si algún día aparece un escritor concurrente de larga duración, el CAS se puede añadir sin romper nada.
- **No cubre el conflicto semántico**, que es otro problema y ya tiene solución propia: la colisión de clave idéntica con valor distinto la detecta `skillstate merge` y falla de forma ruidosa ([ADR 0006](./0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md)).
