---
status: accepted
---

# La instrumentación mide el estado, no el ahorro de tokens

El análisis previo proponía telemetría de tokens con una justificación explícita: decidir si despachar **un subagente por paso** dentro del cliente interactivo se pagaba a sí mismo, dado su coste fijo por invocación. Esa justificación **ya no aplica**. El subagente por paso era un apaño para aproximar el descarte de `Rt` sin tener un bucle propio, y un orquestador como Syntony *es* ese bucle: despachar `claude -p` arranca en frío en cada invocación, que es el modelo por invocación completo y no su imitación. La pregunta que la medición iba a responder no se plantea.

Se decide instrumentar otra cosa: **el tamaño y la calidad de Σ a lo largo del tiempo** — cuánto crece, qué campos se usan de verdad, cuántos casi-duplicados aparecen, con qué frecuencia falla `merge` por conflicto de clave. Lo emite el propio CLI en cada operación, sin telemetría del cliente ni corpus de tareas.

## Por qué esto y no el A/B del paper

El criterio es cuál de las dos mediciones **puede cambiar una decisión ya tomada**. El ahorro de tokens del paper no está en duda, y reproducirlo no alteraría nada de lo diseñado: aunque saliera 8× en vez de 16,2×, se construiría exactamente lo mismo. En cambio [ADR 0006](./0006-merge-promueve-un-subconjunto-y-falla-ruidoso.md) acepta a sabiendas que la promoción por campo no distingue hechos durables de hechos locales dentro de `facts`, con la nota de que si resulta ruidoso habrá que marcar la durabilidad por ítem — y nadie sabe todavía si resulta ruidoso. [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md) asume que los campos extendidos se usarán poco; tampoco está comprobado.

Medir lo que confirma lo que ya se sabe es un trámite. Medir lo que puede refutar una decisión propia es lo otro.

## Consequences

- **Dónde va el dato lo resuelve el [ADR 0017](./0017-historico-local-inalcanzable-desde-el-agente.md)**: un almacén local fuera del repositorio, y no un fichero por proyecto. La diferencia importa porque las preguntas de arriba se contestan **cruzando proyectos**, y un registro por repositorio no las responde.
- **Coste marginal cero.** El CLI ya lee y escribe el estado en cada operación; registrar unos pocos números por llamada no añade trabajo.
- **No depende de Claude Code.** A diferencia de `claude_code.token.usage`, funciona igual en cualquier cliente y bajo orquestación — lo que importa dado que el enforcement ya es asimétrico por cliente ([ADR 0009](./0009-interfaz-y-garantia-portable.md)).
- **Si algún día hiciera falta el A/B de tokens**, conviene recordar la limitación verificada: `agent.name` colapsa todos los subagentes propios al valor `"custom"`, así que separa orquestador de subagentes pero no unos subagentes de otros.
