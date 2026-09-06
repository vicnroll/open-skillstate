---
status: accepted
---

# El histórico se identifica por `project_id`; `project` es sólo un nombre mutable

El [ADR 0017](./0017-historico-local-inalcanzable-desde-el-agente.md) necesita agrupar eventos del histórico aunque el repositorio se mueva. La primera versión usaba `project` como clave porque una ruta no es estable, pero **un nombre tampoco es una identidad**: dos repositorios pueden llamarse igual y un rename partiría la serie histórica en dos.

Se decide separar identidad y presentación:

```json
{
  "project_id": "550e8400-e29b-41d4-a716-446655440000",
  "project": "open-skillstate"
}
```

- `project_id` es un **UUID v4 opaco, generado una vez por `skstate init` e inmutable**.
- `project` es un nombre humano mutable. Puede cambiar con un parche normal sin afectar la identidad.
- Ambos son campos `meta`: se almacenan y validan, pero `skstate get` no los emite.
- El histórico SQLite se relaciona por `project_id`; `project` se guarda como atributo de presentación, no como clave.

## Por qué UUID v4

No hace falta orden temporal ni contenido semántico en el identificador. UUID v4 se genera con aleatoriedad criptográfica usando la librería estándar de Go, no requiere coordinación ni un servicio externo y evita introducir una dependencia sólo para crear IDs ordenables.

El identificador no pretende ser secreto. Su propiedad es ser estable y prácticamente único.

## Inmutabilidad

`project_id` es propiedad del runtime. Después de `init`, un `patch` que intente cambiarlo o borrarlo se rechaza aunque el valor resultante validase contra JSON Schema. La inmutabilidad es una regla de transición y por tanto no se puede expresar únicamente con el esquema del documento.

`migrate` conserva siempre el `project_id`. Un worker creado con `skstate init --worker` **hereda** el identificador del estado consolidado del proyecto; nunca genera uno nuevo.

## `init` construye el estado inicial

La identidad única impide copiar un `state.json` estático desde `skills/`: todas las instalaciones compartirían el mismo ID o el template tendría que contener un valor inválido.

Por tanto `init` **construye el documento inicial** a partir de la versión de esquema embebida, genera `project_id` y escribe la representación canónica definida en el [ADR 0003](./0003-esquema-por-niveles-y-serializacion-dinamica.md). La carga útil distribuible contiene skills, referencias y fragmentos de integración, no un estado pre-generado.

## Renames y movimientos

Mover el repositorio no cambia nada: el ID viaja dentro del `state.json` versionado. Renombrarlo sólo cambia `project`:

```bash
skstate patch --stdin <<'EOF'
{ "project": "new-name" }
EOF
```

Los eventos anteriores y posteriores siguen unidos por `project_id`.

## Relación con `deinit`

`uninstall` conserva el workspace y por tanto conserva la identidad. `deinit`, en cambio, elimina deliberadamente el workspace del proyecto ([ADR 0012](./0012-init-completa-la-instalacion.md)); un `init` posterior crea una **identidad nueva**. El histórico de la identidad anterior no se borra automáticamente: eso requiere `history purge` explícito.

Esta distinción es deliberada: desinstalar una integración no debe romper continuidad; desinicializar el proyecto sí declara que deja de ser la misma instancia de OpenSkillState.

## Consequences

- Dos repositorios con el mismo nombre nunca mezclan sus históricos.
- Un rename no parte una serie histórica.
- Los worktrees y workers pertenecen al mismo proyecto sin depender de rutas.
- El esquema v1 incorpora `project_id` antes de tener usuarios, evitando una migración futura puramente identificativa.
- `project` deja de cargar con dos responsabilidades incompatibles: identidad estable y nombre editable.

## Considered Options

- **Usar la ruta absoluta** — descartado: mover o clonar el repositorio cambia la identidad.
- **Usar `project` como clave** — descartado: colisiona entre repositorios homónimos y un rename parte la serie.
- **Derivar el ID del remote Git** — descartado: un proyecto puede no usar Git, puede tener varios remotes y un fork legítimo heredaría o cambiaría identidad según una regla difícil de explicar.
- **ULID/UUID v7** — válidos, pero su orden temporal no aporta nada a esta identidad y exigiría más implementación o dependencia que UUID v4.
