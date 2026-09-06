---
status: accepted
---

# La instrumentación mide el estado, no el ahorro de tokens

El análisis previo proponía telemetría de tokens con una justificación explícita: decidir si despachar **un subagente por paso** dentro del cliente interactivo se pagaba a sí mismo, dado su coste fijo por invocación. Esa justificación deja de aplicar cuando existe un runtime externo capaz de arrancar workers frescos: el descarte de la transcripción deja de aproximarse mediante subagentes y pasa a ser una propiedad de cómo ese runtime despacha sesiones.

OpenSkillState no depende de un runtime concreto y, por tanto, no instrumenta APIs de telemetría propias de un cliente u orquestador. Se decide medir algo que el propio CLI conoce en todos los entornos: **el tamaño y la calidad de Σ a lo largo del tiempo** — cuánto crece, qué campos se usan de verdad, cuántos casi-duplicados aparecen y con qué frecuencia falla `merge` por conflicto de clave.

## Por qué esto y no el A/B del paper

El criterio es cuál de las mediciones **puede cambiar una decisión propia**. El ahorro de tokens del paper no está en duda y reproducirlo no alteraría el núcleo del producto. En cambio [ADR 0006](./0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md) acepta a sabiendas que la promoción por colección no distingue todos los elementos durables de los locales, y [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md) asume que los campos extendidos se usarán poco. Ambas hipótesis sí pueden resultar falsas en uso real.

Además, el [ADR 0020](./0020-presupuesto-de-estado.md) convierte el tamaño de Σ en una propiedad explícitamente observable desde v1. `check` informa de bytes de la representación compacta y distribución por colecciones; el histórico guarda esas métricas para estudiar tendencias entre proyectos.

Medir lo que confirma lo que ya se sabe es un trámite. Medir lo que puede refutar una decisión propia permite corregir arquitectura.

## Consequences

- **Dónde va el dato lo resuelve el [ADR 0017](./0017-historico-local-inalcanzable-desde-el-agente.md)**: un almacén local fuera del repositorio, y no un fichero por proyecto. Las preguntas relevantes se contestan cruzando proyectos.
- **Coste marginal bajo.** El CLI ya lee y escribe el estado en cada operación; registrar atributos escalares y tamaños no añade una dependencia del cliente.
- **Es portable.** Funciona igual con cualquier agente, CLI, MCP u orquestador porque observa la propia representación de OpenSkillState.
- **Si algún consumidor quiere medir tokens exactos**, esa telemetría pertenece al consumidor. OpenSkillState puede correlacionarla mediante `project_id` y marcas temporales, pero no la convierte en requisito del producto.
