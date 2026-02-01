# ADR-002: Transactional Event Log (Atomic Audit Trail)

## Status

**ACCEPTED** (2026-01-31)

---

## Context

PaySentry needs a complete audit trail of all financial operations to meet the trust metric: **"Audit trail completeness = 100%"**.

**Forces at play:**

1. **Critical trust metric:** If a single operation is missing from the log, the system is not auditable
2. **Idempotency:** To detect duplicates we need to query the event log (`idempotency_key`)
3. **Compliance:** In financial systems, audit trail is a legal requirement, not "nice to have"
4. **Availability vs Consistency:** We have a >99% uptime target, but also 100% log completeness

**The dilemma:**

When we execute an authorization or payment, what do we do if **the event log fails**?

```
POST /authorizations/{id}/capture
  ↓
Evaluate policy → approved
  ↓
[WRITE TO EVENT LOG] ← What if this fails?
  ↓
Execute payment via Payment Adapter
```

**Option A:** Reject the operation if the log fails (Consistency > Availability)
**Option B:** Execute anyway and log async (Availability > Consistency)

---

## Decision

**The event log will be transactional and atomic with state changes.**

No financial operation will be executed if its logging in the audit trail cannot be guaranteed.

### Implementation

All critical operations (create authorization, change state, execute payment) will be written in a **database transaction** that includes:

1. State change in the main table (e.g., `authorizations`)
2. Corresponding event in `event_log`

```python
# Example: Approve an authorization
@transactional
def approve_authorization(auth_id: str, user_id: str):
    # 1. Change state
    auth = db.query(Authorization).filter_by(id=auth_id).first()
    auth.status = "approved"

    # 2. Log event (same transaction)
    event = Event(
        event_type="authorization.approved",
        entity_id=auth_id,
        user_id=user_id,
        payload={"amount": auth.amount, "agent_id": auth.agent_id},
        timestamp=utcnow()
    )
    db.add(event)

    # 3. Atomic commit or full rollback
    db.commit()  # If this fails, NOTHING persists
```

If the `event_log` INSERT fails → transaction rolls back → state change doesn't occur → we return `500 Internal Server Error`.

### Consequence for Idempotency

The Mock Payment Adapter (and later BIND Adapter) **must** implement its own idempotency:

```python
class MockPaymentAdapter:
    def execute_transfer(self, amount, destination_cbu, idempotency_key):
        # Look for previous transfer with same key
        existing = self.db.query(MockTransfer).filter_by(
            idempotency_key=idempotency_key
        ).first()

        if existing:
            # Duplicate → return original result without re-executing
            return existing.result

        # First time → simulate and save
        result = self._simulate_transfer(amount, destination_cbu)
        self.db.add(MockTransfer(
            idempotency_key=idempotency_key,
            result=result
        ))
        self.db.commit()
        return result
```

The event log is still useful for **observability** (see all attempts, including duplicates), but the **non-duplication guarantee** is provided by the processor adapter.

---

## Consequences

### Positive

- ✅ **Consistency guarantee:** Impossible to have authorization without event, or event without authorization
- ✅ **Complete audit trail:** 100% of operations logged (trust metric met)
- ✅ **Simpler design:** We don't need to handle eventual consistency, complex retries, or dead letter queues
- ✅ **Easier debugging:** DB state and event log are always synchronized
- ✅ **Compliance out-of-the-box:** Any audit can trust the event log

### Negative

- ⚠️ **Event log becomes SPOF (Single Point of Failure):** If DB is down, entire system is down
- ⚠️ **Availability coupled to DB:** `Uptime(PaySentry) ≤ Uptime(PostgreSQL)`
- ⚠️ **Write latency:** Each operation pays the cost of writing to `event_log` (~10-20ms per INSERT)
- ⚠️ **Event_log table growth:** Fills up fast, will eventually need partitioning/archiving

### Neutral

- Event log must be in the same database as transactional tables (can't be an external service)
- We can't use async logging (e.g., send to CloudWatch via queue) as a replacement for transactional event log

---

## Alternatives Considered

| Alternative | Pros | Cons | Reason for Discarding |
|-------------|------|------|----------------------|
| **Transactional event log (chosen)** | Guaranteed consistency, 100% audit trail | DB becomes SPOF | ✅ In financial systems, consistency > availability |
| **Async event log with retry** | Higher availability, lower latency | Possibility of lost events (99.9%, not 100%) | ❌ Violates trust metric ("100% completeness") |
| **Event log in separate service** | Decouples availability | Requires 2-phase commit or complex saga | ❌ Excessive complexity for MVP |
| **No event log, only app logs** | Simpler | Zero auditability, doesn't meet compliance | ❌ Destroys value proposition ("OAuth for money") |

---

## References

- **DDIA Ch 7 - Transactions:** "Atomicity simplifies the programming model: if a transaction cannot be completed in full, the application can be sure that it didn't change anything."
- **DDIA Ch 9 - Consistency Guarantees:** "For systems that require strong consistency (financial transactions), you cannot compromise on this guarantee."
- **Martin Kleppmann - Event Sourcing:** https://martin.kleppmann.com/2015/03/04/turning-the-database-inside-out.html

---

## Notes

### Implications for Database Choice

This decision reinforces the choice of **PostgreSQL** (via Supabase) because:
- Native ACID transactions
- Excellent performance for short transactions (typically <50ms)
- Supabase free tier includes 500MB (sufficient for MVP with ~10K events)

### Availability Mitigation Strategy

While we accept that event log is SPOF, we can mitigate with:

1. **Aggressive monitoring:** Alert if DB response time > 200ms
2. **Health checks:** `GET /health` verifies DB connectivity before accepting requests
3. **Circuit breaker:** If DB is down, respond `503 Service Unavailable` immediately (don't wait for timeout)
4. **Document in SLA:** "PaySentry uptime depends on PostgreSQL uptime (Supabase: 99.9% SLA)"

### When to Reconsider This Decision

Reconsider if:
- Volume grows to >1M events/day (event_log becomes bottleneck)
- We need uptime >99.9% (would require PostgreSQL with replicas, outside MVP budget)
- We want to add real-time event streaming (e.g., WebSockets for live dashboard)

In that case, evaluate **Event Sourcing with Kafka/Redis Streams** as replacement for transactional event_log.

But for MVP: **keep it simple, keep it consistent**.
