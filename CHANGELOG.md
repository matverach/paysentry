# Changelog

Registro de progreso del proyecto PaySentry.

## [2026-02-01] - Sesión 3

### Added
- Data Model completo (`docs/03-data-model.md`)
  - Schema PostgreSQL con 7 entidades (users, agents, policies, agent_stats, authorizations, transactions, events)
  - Diagrama ER con Mermaid
  - Queries de validación contra casos de uso
  - Índices diseñados según patrones de acceso
  - Estimación de storage para MVP
- ADR-004: Atomic Aggregate Updates
  - Decisión sobre agregados materializados vs cálculo on-the-fly
  - Lazy reset pattern para agent_stats
  - Trade-off de write amplification por O(1) en validaciones
- README en inglés como main con diagrama del data model
- README.es.md (versión en español)

### Changed
- Traducidos a inglés: `docs/01-problem.md`, `docs/02-requirements.md`, todos los ADRs
- Actualizado progreso a 3/8 artefactos core (37.5%)
- Actualizado state.md con decisiones del data model
- Actualizado portfolio-highlights.md con ADR-004 y Data Model

---

## [2026-02-01] - Sesión 2

### Added
- Requirements Specification completo (`docs/02-requirements.md`)
  - Constraints, NFRs, casos de uso críticos, FRs
  - Priorización MoSCoW con justificación técnica
  - Assumptions documentados
  - Success criteria concreto
- ADR-003: Agent Token Storage Strategy
  - Decisión de hashear tokens con bcrypt
  - Exactly-once token delivery pattern

### Changed
- Actualizado progreso a 2/8 artefactos core (25%)
- Definido approach de requirements: Constraints → NFRs → Use Cases → FRs

---

## [2026-01-31] - Sesión 1

### Added
- Problem Statement (`docs/01-problem.md`)
  - Caso de uso concreto (pago automático de expensas)
  - Métricas de éxito técnicas y de confianza
  - Delimitación clara de alcance MVP
- ADR-001: Mock Adapter para MVP
  - Investigación de procesadores de pago (MercadoPago, BIND)
  - Decisión de Hexagonal Architecture
- ADR-002: Event Log Transaccional
  - Análisis de consistency vs availability
  - Decisión de atomicidad en audit trail
- Estructura inicial del repositorio
- CLAUDE.md con contexto del proyecto
- Template de ADR (`docs/adr/000-template.md`)
- README del proyecto

---

## Formato

Basado en [Keep a Changelog](https://keepachangelog.com/).

- **Added** - Nuevas features o documentos
- **Changed** - Cambios en features existentes
- **Deprecated** - Features que serán removidas
- **Removed** - Features removidas
- **Fixed** - Bug fixes
- **Security** - Vulnerabilidades

---

## Tip para LinkedIn

Cada entrada mayor en este changelog puede ser un post de LinkedIn:
- "Definí el problem statement de PaySentry. El challenge fue..."
- "Tomé la decisión de usar X sobre Y. Mi razonamiento..."
- "Primera versión del data model. Aprendí sobre..."
