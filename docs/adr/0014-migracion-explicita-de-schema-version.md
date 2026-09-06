---
status: accepted
---

# Las migraciones de esquema son explícitas y las versiones desconocidas se rechazan

Σ lleva un `schema_version`, así que antes o después habrá estados escritos por una versión del CLI distinta de la que los lee. Se decide **no migrar automáticamente**:

- `skillstate check` detecta el desajuste y dice qué hacer.
- `skillstate migrate` migra de forma explícita, dejando copia de seguridad del estado anterior.
- Un `schema_version` **más nuevo** que el binario se rechaza siempre. No hay forma de saber qué significan campos que esta versión desconoce, y adivinar sería peor que parar.

La migración automática al leer sería más cómoda, pero cambiaría los datos del usuario en silencio, y una migración entre esquemas puede ser con pérdida — renombrar o eliminar campos es exactamente lo que ocurre entre versiones. Es el mismo criterio que rige el resto del diseño: cuando algo puede salir mal, que falle de forma ruidosa en lugar de resolverse solo.

## Consequences

- **Se escribe la política antes de necesitarla.** Hoy no hay usuarios ni una v2 del esquema, así que era tentador aplazarla; pero dejar escrito «rechaza lo que no entiendas» cuesta una línea ahora, y descubrir después que la v1.0 corrompía estados de la v1.1 cuesta bastante más.
