# ADR-002: Event Log Transaccional (Atomic Audit Trail)

## Status

**ACCEPTED** (2026-01-31)

---

## Context

PaySentry necesita un audit trail completo de todas las operaciones financieras para cumplir con la métrica de confianza: **"Audit trail completeness = 100%"**.

**Fuerzas en juego:**

1. **Métrica de confianza crítica:** Si falta una sola operación del log, el sistema no es auditable
2. **Idempotencia:** Para detectar duplicados necesitamos consultar el event log (`idempotency_key`)
3. **Compliance:** En sistemas financieros, el audit trail es requisito legal, no "nice to have"
4. **Availability vs Consistency:** Tenemos target de uptime > 99%, pero también 100% completitud del log

**El dilema:**

Cuando ejecutamos una autorización o pago, ¿qué hacemos si **el event log falla**?

```
POST /authorizations/{id}/capture
  ↓
Evaluar política → approved
  ↓
[WRITE TO EVENT LOG] ← ¿Qué pasa si esto falla?
  ↓
Ejecutar pago via Payment Adapter
```

**Opción A:** Rechazar la operación si el log falla (Consistency > Availability)
**Opción B:** Ejecutar igual y loguear async (Availability > Consistency)

---

## Decision

**El event log será transaccional y atómico con los cambios de estado.**

No se ejecutará ninguna operación financiera si no puede garantizarse su registro en el audit trail.

### Implementación

Todas las operaciones críticas (crear autorización, cambiar estado, ejecutar pago) se escribirán en una **transacción de base de datos** que incluye:

1. El cambio de estado en la tabla principal (ej: `authorizations`)
2. El evento correspondiente en `event_log`

```python
# Ejemplo: Aprobar una autorización
@transactional
def approve_authorization(auth_id: str, user_id: str):
    # 1. Cambiar estado
    auth = db.query(Authorization).filter_by(id=auth_id).first()
    auth.status = "approved"

    # 2. Registrar evento (misma transacción)
    event = Event(
        event_type="authorization.approved",
        entity_id=auth_id,
        user_id=user_id,
        payload={"amount": auth.amount, "agent_id": auth.agent_id},
        timestamp=utcnow()
    )
    db.add(event)

    # 3. Commit atómico o rollback completo
    db.commit()  # Si esto falla, NADA se persiste
```

Si el `event_log` INSERT falla → la transacción hace rollback → el cambio de estado no ocurre → retornamos `500 Internal Server Error`.

### Consecuencia para Idempotencia

El Mock Payment Adapter (y luego BIND Adapter) **debe** implementar su propia idempotencia:

```python
class MockPaymentAdapter:
    def execute_transfer(self, amount, destination_cbu, idempotency_key):
        # Buscar transferencia previa con misma key
        existing = self.db.query(MockTransfer).filter_by(
            idempotency_key=idempotency_key
        ).first()

        if existing:
            # Duplicado → retornar resultado original sin re-ejecutar
            return existing.result

        # Primera vez → simular y guardar
        result = self._simulate_transfer(amount, destination_cbu)
        self.db.add(MockTransfer(
            idempotency_key=idempotency_key,
            result=result
        ))
        self.db.commit()
        return result
```

El event log sigue siendo útil para **observabilidad** (ver todos los intentos, incluyendo duplicados), pero la **garantía de no duplicación** la da el adapter del procesador.

---

## Consequences

### Positivas

- ✅ **Garantía de consistencia:** Imposible tener autorización sin evento, o evento sin autorización
- ✅ **Audit trail completo:** 100% de operaciones logueadas (métrica de confianza cumplida)
- ✅ **Diseño más simple:** No necesitamos manejar eventual consistency, retries complejos, o dead letter queues
- ✅ **Debugging más fácil:** El estado de la DB y el event log siempre están sincronizados
- ✅ **Compliance out-of-the-box:** Cualquier auditoría puede confiar en el event log

### Negativas

- ⚠️ **Event log se vuelve SPOF (Single Point of Failure):** Si la DB está caída, todo el sistema está caído
- ⚠️ **Availability acoplada a la DB:** `Uptime(PaySentry) ≤ Uptime(PostgreSQL)`
- ⚠️ **Latencia de escritura:** Cada operación paga el costo de escribir a `event_log` (+ ~10-20ms por INSERT)
- ⚠️ **Crecimiento de tabla event_log:** Se llena rápido, eventualmente necesitará particionado/archivado

### Neutras

- El event log debe estar en la misma base de datos que las tablas transaccionales (no puede ser un servicio externo)
- No podemos usar logging async (ej: enviar a CloudWatch vía queue) como reemplazo del event log transaccional

---

## Alternatives Considered

| Alternativa | Pros | Cons | Razón de Descarte |
|-------------|------|------|-------------------|
| **Event log transaccional (elegida)** | Consistencia garantizada, audit trail 100% | DB se vuelve SPOF | ✅ En sistemas financieros, consistency > availability |
| **Event log async con retry** | Mayor availability, menor latencia | Posibilidad de eventos perdidos (99.9%, no 100%) | ❌ Viola métrica de confianza ("100% completitud") |
| **Event log en servicio separado** | Desacopla availability | Requiere 2-phase commit o saga compleja | ❌ Complejidad excesiva para MVP |
| **No tener event log, solo logs de aplicación** | Más simple | Cero auditabilidad, no cumple compliance | ❌ Destruye propuesta de valor ("OAuth para dinero") |

---

## References

- **DDIA Cap 7 - Transactions:** "Atomicity simplifies the programming model: if a transaction cannot be completed in full, the application can be sure that it didn't change anything."
- **DDIA Cap 9 - Consistency Guarantees:** "For systems that require strong consistency (financial transactions), you cannot compromise on this guarantee."
- **Martin Kleppmann - Event Sourcing:** https://martin.kleppmann.com/2015/03/04/turning-the-database-inside-out.html

---

## Notes

### Implicaciones para la Elección de Base de Datos

Esta decisión refuerza la elección de **PostgreSQL** (vía Supabase) porque:
- ACID transactions nativas
- Performance excelente para transacciones cortas (< 50ms típicamente)
- Free tier de Supabase incluye 500MB (suficiente para MVP con ~10K eventos)

### Estrategia de Mitigación de Availability

Si bien aceptamos que el event log es SPOF, podemos mitigar con:

1. **Monitoring agresivo:** Alerta si DB response time > 200ms
2. **Health checks:** `GET /health` verifica conectividad a DB antes de aceptar requests
3. **Circuit breaker:** Si DB está caída, responder `503 Service Unavailable` inmediatamente (no esperar timeout)
4. **Documentar en SLA:** "Uptime de PaySentry depende de uptime de PostgreSQL (Supabase: 99.9% SLA)"

### Cuándo Reconsiderar Esta Decisión

Reconsiderar si:
- El volumen crece a >1M eventos/día (event_log se vuelve cuello de botella)
- Necesitamos uptime > 99.9% (requeriría PostgreSQL con réplicas, fuera de presupuesto MVP)
- Queremos agregar event streaming en tiempo real (ej: WebSockets para dashboard live)

En ese caso, evaluar **Event Sourcing con Kafka/Redis Streams** como reemplazo del event_log transaccional.

Pero para MVP: **keep it simple, keep it consistent**.
