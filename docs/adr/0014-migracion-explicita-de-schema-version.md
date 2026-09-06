---
status: accepted
---

# Las migraciones de esquema son explícitas y las versiones desconocidas se rechazan

Σ lleva un `schema_version`, así que antes o después habrá estados escritos por una versión del CLI distinta de la que los lee. Se decide **no migrar automáticamente**:

- `skillstate check` detecta el desajuste y dice qué hacer.
- `skillstate migrate` migra de forma explícita, dejando copia de seguridad del estado anterior.
- Un `schema_version` **más nuevo** que el binario se rechaza siempre. No hay forma de saber qué significan campos que esta versión desconoce, y adivinar sería peor que parar.

La migración automática al leer sería más cómoda, pero cambiaría los datos del usuario en silencio, y una migración entre esquemas puede ser con pérdida — renombrar o eliminar campos es exactamente lo que ocurre entre versiones. Es el mismo criterio que rige el resto del diseño: cuando algo puede salir mal, que falle de forma ruidosa en lugar de resolverse solo.

## Un estado más antiguo opera, no bloquea

El caso simétrico — el estado en una versión **más antigua** que la que conoce el binario — no se trata igual, porque no tiene nada de ambiguo: el binario lleva ese esquema embebido ([ADR 0015](./0015-el-esquema-vive-dentro-del-binario.md)) y sabe validar contra él perfectamente. Negarse sería negarse a hacer algo que se sabe hacer bien. Se decide que **`get` y `patch` operan en la versión del proyecto**, avisando, y que sólo `migrate` la cambia.

Bloquear hasta migrar habría sido más estricto de lo que este mismo ADR pide. Lo que se rechazó arriba es cambiar los datos del usuario en silencio, y operar en la versión que el proyecto ya tiene no los cambia en absoluto: los deja donde están. Y bloquear rompería el camino orquestado, donde un worker headless recibe el error sin ningún humano que pueda ejecutar `migrate` — el mismo fallo que el [ADR 0012](./0012-init-completa-la-instalacion.md) evita en `init` con la detección de TTY.

## Dos ventanas, no una

«Dejar de soportar una versión» son en realidad dos cosas con vidas distintas, y confundirlas deja proyectos varados:

| Ventana | Qué cubre | Cuánto dura |
|---|---|---|
| **Operación** | `get` y `patch` funcionan sobre esa versión | Acotada; la política concreta se decidirá con datos de uso |
| **Migración** | `migrate` sabe leer esa versión y llevarla adelante | **Indefinida** |

La razón de que la segunda no caduque es que `migrate` necesita el esquema de **origen** para leer el estado viejo, no sólo el de destino. Si retirar una versión la borrara del binario, el proyecto que siguiera en ella no quedaría bloqueado sino **varado**: ni opera ni puede migrar, y la única salida sería descargar un binario antiguo, migrar con él y volver. Es exactamente el callejón sin salida que el resto del diseño evita.

Cuando toque fijar la ventana de operación, conviene medirla en **distancia de versiones** y no en tiempo. El binario sabe por sí mismo qué esquemas embebe; una caducidad por fecha le obligaría a comparar contra el calendario, y un binario con opiniones sobre el calendario envejece mal — alguien instala una versión de hace dos años y empieza a rechazarle cosas que ayer funcionaban.

## Las migraciones son una cadena

El usuario ejecuta **un solo `migrate`** y llega a la versión vigente, venga de donde venga. Por dentro, esa migración se compone de **saltos consecutivos** (`1→2`, `2→3`, `3→4`), no de convertidores directos a la última.

```text
✓  1→2, 2→3, 3→4      una versión nueva cuesta UN salto; 1→4 se compone
✗  1→4, 2→4, 3→4      una versión nueva obliga a reescribir uno por cada versión viva
```

Con convertidores directos, el trabajo de mantenimiento crece cuadráticamente y la ventana de migración indefinida se vuelve impagable justo cuando más versiones hay. Con la cadena, cada versión nueva cuesta lo mismo que la anterior. No cuesta nada adoptarlo desde el primer salto y es caro de reconvertir después, así que se fija antes de escribir la primera migración.

## Consequences

- **Se escribe la política antes de necesitarla.** Hoy no hay usuarios ni una v2 del esquema, así que era tentador aplazarla; pero dejar escrito «rechaza lo que no entiendas» cuesta una línea ahora, y descubrir después que la v1.0 corrompía estados de la v1.1 cuesta bastante más.
- **El aviso de versión antigua necesita un canal que llegue a una persona.** Bajo orquestación, un mensaje por `stderr` en un worker headless no lo lee nadie: el proceso termina y se pierde. El desajuste tiene que aparecer también en `skillstate check`, que es lo que ejecuta un humano o un CI. Si el único canal fuera `stderr`, la ventana de aviso no existiría precisamente donde más falta hace.
- **La cadena de migraciones sólo crece hacia adelante.** Ningún salto se reescribe al añadir una versión nueva, así que el coste de mantener la ventana indefinida es lineal y conocido.
