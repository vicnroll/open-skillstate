---
status: accepted
---

# La v1 hace visible un presupuesto de estado sin imponer un límite universal

El patrón depende de que Σ permanezca pequeño respecto al trabajo que representa. Las claves semánticas del [ADR 0002](./0002-listas-como-objetos-con-clave-semantica.md) evitan duplicados y cronología disfrazada, pero **no garantizan que el número de hechos distintos deje de crecer**. Un estado puede ser perfectamente válido contra el esquema y, aun así, haber dejado de ser un buen estado de ejecución.

Por tanto «Σ es acotado» no puede presentarse como una garantía mecánica de la v1. Se decide incorporar desde la primera versión un **state budget observable**: el CLI mide el tamaño de la vista que realmente entra en contexto y hace visible su composición antes de que el crecimiento se convierta en una propiedad oculta.

## Qué mide `check`

`skstate check` informa siempre, además de la validez, de:

- bytes de la salida compacta de `skstate get` — el Σ que realmente se inyectaría, no el `state.json` indentado;
- número de entradas de las colecciones que pueden crecer (`facts`, `decisions`, `constraints`, `blockers`, `hypotheses`, `files.*`, `verification.checks`);
- tamaño por campo, para poder localizar dónde se concentra el crecimiento.

Ejemplo:

```text
state: valid
state size: 14.2 KiB
facts: 87
  8.6 KiB
decisions: 23
  2.1 KiB
constraints: 12
  1.0 KiB
verification.checks: 18
  1.4 KiB
```

Estas métricas también se registran en la instrumentación del [ADR 0010](./0010-se-instrumenta-el-estado-no-los-tokens.md), de modo que más adelante se pueda decidir con datos qué presupuestos son razonables.

## No hay un número mágico global

No se fija en v1 un máximo universal de 8 KiB, 16 KiB, 100 elementos ni otro número arbitrario. OpenSkillState es agnóstico del modelo, del tamaño de contexto y del tipo de repositorio; un umbral global sin evidencia convertiría una intuición en contrato.

La v1 sí permite **declarar un presupuesto blando en la invocación**:

```bash
skstate check --budget-bytes 16384
```

Si el Σ lo supera, el estado sigue siendo válido y el comando lo señala como advertencia:

```text
warning: state growth exceeds configured budget (14.2 KiB > 12.0 KiB)
```

El presupuesto es una **señal de calidad, no una frontera de validez**. `patch` no rechaza un cambio sólo porque lo supere y `get` nunca poda contenido por su cuenta. El CLI no puede decidir qué hecho deja de ser necesario sin introducir juicio semántico en el runtime.

No se persiste todavía un budget dentro del workspace. Quien quiera enforcement en CI puede pasar la misma bandera en su comando de comprobación; si la práctica demuestra que hace falta una política persistente, se añadirá explícitamente en vez de esconder configuración dentro de Σ.

## Qué ocurre cuando crece demasiado

La respuesta de v1 es deliberadamente humana/agéntica:

1. `check` muestra tamaño y concentración.
2. El agente o la persona revisa los campos grandes.
3. Elimina hechos recuperables del repositorio, entradas obsoletas, duplicados semánticos o conocimiento que ya no afecta al siguiente paso mediante un parche normal.
4. La cronología no se recupera desde Σ; si hace falta para análisis, vive separada según el [ADR 0017](./0017-historico-local-inalcanzable-desde-el-agente.md).

Una futura compactación automática sólo sería aceptable si mantiene las invariantes del patrón; resumir con pérdida el estado por debajo de un número sería reintroducir el mismo problema que OpenSkillState intenta evitar.

## Relación con complejidad O(T)

La arquitectura puede afirmar que el patrón **permite** mantener O(T) cuando el tamaño de Σ permanece acotado, pero OpenSkillState v1 no demuestra por construcción que `|Σt| = O(1)`. Lo que sí hace es evitar crecimiento accidental conocido y hacer medible el crecimiento residual.

Esta precisión importa: si `|Σt|` creciera linealmente con el número de pasos, el coste acumulado podría volver a ser cuadrático aunque no exista transcripción conversacional.

## Consequences

- `check` deja de ser sólo un validador binario y se convierte también en diagnóstico de salud de Σ.
- La primera versión recoge datos capaces de justificar un budget recomendado futuro sin inventarlo por adelantado.
- El tamaño se mide sobre la representación que consume contexto, no sobre bytes de almacenamiento.
- No aparece una segunda semántica de poda dentro del CLI: el runtime mide; el agente decide qué información sigue siendo necesaria.

## Considered Options

- **Límite duro global** — descartado: sería arbitrario y podría rechazar estados legítimos de dominios distintos.
- **Poda automática al superar el límite** — descartado: requiere juicio semántico y puede destruir precisamente la observación que un paso futuro necesita.
- **No medir nada hasta tener usuarios** — descartado: sin métricas no habrá datos para saber cuándo la premisa de estado acotado deja de cumplirse.
- **Medir tokens exactos** — descartado: depende del tokenizer/modelo. Bytes de la representación compacta son portables, baratos y comparables en el tiempo; el [ADR 0010](./0010-se-instrumenta-el-estado-no-los-tokens.md) ya fijó que la instrumentación base es sobre el estado.
