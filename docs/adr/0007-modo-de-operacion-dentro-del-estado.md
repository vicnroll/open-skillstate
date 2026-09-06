---
status: accepted
---

# El régimen de disciplina se declara dentro de Σ, en un campo `mode`

La sección 7 del paper reconoce tres escenarios donde su premisa falla: cuando no hay esquema fijo conocido de antemano, cuando la actualización correcta depende de una observación anterior cuya relevancia no se reconoció en su momento, y cuando el objetivo se define sobre la trayectoria misma. Depurar, auditar y explorar caen ahí, y en esos casos la disciplina estricta de estado **empeora** el resultado. El enforcement — la regla `deny` y el hook `Stop` — vuelve esa disciplina mecánica e inesquivable, así que hace falta una vía de escape explícita. Se decide un campo de núcleo `mode: "execution" | "exploration"`.

## Qué relaja el modo, y qué no

**La regla `deny` no se toca en ningún modo.** Sólo enruta las escrituras por el CLI; no obliga a escribir más ni a estructurar antes de tiempo. Lo que estorba al trabajo exploratorio es el hook `Stop` exigiendo un `next_action` concreto y la presión por destilar a Σ observaciones cuya relevancia todavía se desconoce. En `exploration`, el hook no exige `next_action` y el skill deja de presionar por estructurar.

## Por qué dentro de Σ y no en configuración

Un interruptor en `.skillstate/config.json` o en una variable de entorno resolvería lo mismo, pero quedaría invisible desde el estado: una sesión fresca no sabría en qué régimen opera, y en la práctica lo que se apaga se queda apagado. Dentro de Σ el modo viaja en todos los prompts, lo hereda cualquier sesión que reanude, el hook lo lee sin consultar configuración externa, y **cambiar de modo es un parche auditable** como cualquier otro.

## Consequences

- **El daño que este campo mitiga no es igual en los dos caminos.** En una sesión interactiva la transcripción sigue existiendo, así que una observación del paso 1 se puede rescatar en el paso 7. Bajo orquestación, donde cada worker arranca en frío, esa observación se pierde de verdad — la relevancia retroactiva es un problema bastante más serio ahí.
- **`mode` y `status` son ortogonales; se quedan los dos.** Se sospechó que se solapaban, pero todas las combinaciones significan algo distinto: `active`+`exploration` es depurar en mitad de una tarea, `blocked`+`exploration` es investigar por qué se está atascado, `completed`+`exploration` es auditar lo ocurrido. La intuición de que «explorar es lo que se hace al arrancar» queda cubierta por `active`+`exploration` sin necesidad de un estado nuevo. Fusionarlos obligaría a un producto cartesiano de valores (`active-exploration`, `blocked-exploration`…) que es el mismo par disfrazado de enumeración y más incómodo de consultar desde el hook.

## Considered Options

- **Sin escape, enforcement incondicional** — descartado: coherente, pero haría que el kit empeorase activamente una parte grande del trabajo real en un repositorio.
- **Interruptor fuera de Σ** (fichero de configuración o variable de entorno) — descartado: invisible desde el estado y propenso a quedarse apagado indefinidamente.
