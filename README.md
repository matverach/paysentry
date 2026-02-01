# PaySentry

> Authorization Gateway para Pagos de Agentes IA

[![Status](https://img.shields.io/badge/status-design%20phase-yellow)]()

## 🎯 Qué es PaySentry

PaySentry es infraestructura de autorización para pagos delegados a agentes IA. Pensalo como **OAuth pero para dinero**: un usuario define políticas (límites, categorías, merchants) y PaySentry valida cada transacción antes de ejecutarla.

```
┌────────┐      ┌─────────────┐      ┌────────────┐      ┌─────────┐
│ Agente │  ->  │  PaySentry  │  ->  │ Procesador │  ->  │ Compra/ │
│  IA    │      │  (valida)   │      │ de Pagos   │      │   Trx   │
└────────┘      └─────────────┘      └────────────┘      └─────────┘
                       |
                       v
                 ┌───────────┐
                 │ Políticas │
                 │ del User  │
                 └───────────┘
```

## 🚧 Estado del Proyecto

Este proyecto está en **fase de diseño de arquitectura**. El objetivo es explorar y documentar patrones de arquitectura para sistemas financieros distribuidos.

### Progreso

| Fase | Estado | Documento |
|------|--------|-----------|
| Problem Statement | ✅ Completado | [01-problem.md](docs/01-problem.md) |
| Requirements | ✅ Completado | [02-requirements.md](docs/02-requirements.md) |
| Data Model | ⏳ Siguiente | [03-data-model.md](docs/03-data-model.md) |
| ADRs | 🔄 En progreso (3/?) | [docs/adr/](docs/adr/) |
| System Context (C4) | ⏳ Pendiente | docs/architecture/ |
| API Spec | ⏳ Pendiente | docs/api/ |
| Implementation | ⏳ Pendiente | src/ |

### Architecture Decision Records (ADRs)

| ADR | Decisión | Estado |
|-----|----------|--------|
| [ADR-001](docs/adr/001-mock-adapter-for-mvp.md) | Mock Adapter para MVP | ✅ Accepted |
| [ADR-002](docs/adr/002-transactional-event-log.md) | Event Log Transaccional | ✅ Accepted |
| [ADR-003](docs/adr/003-agent-token-storage.md) | Agent Token Storage (bcrypt) | ✅ Accepted |

## 📚 Documentación

- **[docs/](docs/)** - Documentación de diseño
- **[docs/adr/](docs/adr/)** - Architecture Decision Records

## 🔑 Conceptos Clave

### Actores

| Actor | Descripción | Permisos |
|-------|-------------|----------|
| **Usuario** | Owner del dinero, define políticas | CRUD políticas, aprobar transacciones |
| **Agente** | IA que ejecuta compras | Solo pedir autorización y capturar |
| **Integrador** | Fintech/wallet que integra PaySentry | API access |

### Flujo de Autorización

1. Usuario configura política para un agente
2. Agente solicita autorización (`POST /v1/authorizations`)
3. PaySentry evalúa contra política
4. Si aprobado → agente captura (`POST /v1/authorizations/{id}/capture`)
5. PaySentry ejecuta pago contra procesador
6. Transacción queda en audit log

## 🛠️ Stack Técnico (Propuesto)

| Componente | Tecnología |
|------------|------------|
| Backend | Python (FastAPI) |
| Database | PostgreSQL |
| Cache | Redis |
| Procesador de Pagos | MercadoPago |
| Infra | Railway/Render |

## 📖 Referencias

- [Designing Data-Intensive Applications](https://dataintensive.net/) - Martin Kleppmann
- [MercadoPago API Docs](https://www.mercadopago.com.ar/developers)
- [C4 Model](https://c4model.com/) - Arquitectura de software

## 👤 Autor

Proyecto arquitectónico para explorar patrones de sistemas de autorización financiera.

---

## Changelog

Ver [CHANGELOG.md](CHANGELOG.md) para historial de cambios.
