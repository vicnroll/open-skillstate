---
status: accepted
---

# El CLI de OpenSkillState se distribuye como binario compilado, no como script interpretado

El kit debe adoptarse en repositorios de cualquier stack (Node, Python, .NET, Rust, Go...), así que un script en un lenguaje de aplicación concreto (Python, Node) impondría ese runtime como dependencia dura a repos que no lo usan — la misma asimetría que ya se rechazó para Node, sólo que con otro lenguaje. Se decide compilar el CLI a un binario estático (Go) sin dependencias de runtime, distribuido mediante un instalador universal (`curl | sh`) como canal canónico, más un wrapper npm como canal de conveniencia — elegido por ser el ecosistema más probable hoy entre usuarios de agentes de código — que apunta al mismo binario en lugar de reimplementarlo.

## Considered Options

- **Script vendorizado (Python o Node)** — descartado: impone un runtime de aplicación como dependencia dura a repos que no lo usan.
- **Paquete npm exclusivo instalado con `-g`** — descartado: acopla la versión del CLI a la máquina en vez de al repositorio, de modo que dos repos que necesiten versiones de esquema distintas no pueden convivir; y excluye por completo los stacks no-Node.
- **Wrappers para todos los ecosistemas desde el día uno (npm + pip + cargo + brew)** — descartado por ahora: sobre-invierte en mantenimiento antes de tener evidencia de demanda. Añadir un wrapper más tarde es barato porque todos apuntan al mismo binario.
