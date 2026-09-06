---
status: accepted
---

# El hook `Stop` empuja una vez y cede

El hook `Stop` bloquea el cierre del turno cuando `status` sigue en `active` sin un `next_action` concreto, que es la propiedad de reanudabilidad que sostiene el patrón. Pero se verificó contra la documentación que **el bloqueo no es incondicional**: está topado en ocho veces consecutivas y la entrada del hook incluye un campo `stop_hook_active` para detectar la reentrada. No puede garantizar nada; sólo puede insistir un número acotado de veces.

Se decide **un solo empujón**: bloquea la primera vez, y si vuelve a entrar con `stop_hook_active: true`, cede. Quien lo evalúa es el propio binario, invocado como `skillstate hook stop` ([ADR 0012](./0012-init-completa-la-instalacion.md)); no hay script intermedio con lógica propia.

El hook existe para atrapar un **olvido** — el modelo terminó y no dejó `next_action`. Un recordatorio arregla un olvido. Si tras el recordatorio el turno sigue queriendo cerrarse, ya no es un olvido sino una decisión, y pelearse con ella siete veces más agota el presupuesto de bloqueos, frustra al usuario y no arregla nada. Diseñarlo como barrera dura lo convierte en una trampa.

## Consequences

- **En `mode: exploration` la comprobación se salta entera.** Exigir un `next_action` concreto es justo lo que estorba al trabajo exploratorio, que es el motivo de que exista el campo `mode` ([ADR 0007](./0007-modo-de-operacion-dentro-del-estado.md)).
- **El mensaje debe poder leerlo un humano.** No está documentado qué recibe exactamente el modelo cuando un hook `Stop` sale con código 2 — si el `stderr` le llega como instrucción accionable —, así que el texto se redacta asumiendo que puede que sólo lo lea una persona.
- **La reanudabilidad queda como propiedad *fomentada*, no garantizada.** Conviene decirlo así en la documentación en vez de prometer un enforcement que el cliente no permite sostener.
