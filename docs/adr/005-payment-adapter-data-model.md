# ADR-005: Payment Adapter Data Model

## Status

**ACCEPTED** (2026-02-03)

---

## Context

PaySentry needs to execute payments through external providers (MercadoPago, BIND, etc.). ADR-001 established that the MVP will use a mock adapter with a Strategy pattern in code (Ports & Adapters / Hexagonal Architecture).

However, the **data model** for adapters was not defined. We need to decide:

1. Where to store provider configuration and credentials
2. How agents reference their payment provider
3. How to handle adapter lifecycle (creation, revocation)

**Forces at play:**

- Each user connects their own payment provider account (not PaySentry's account)
- An agent must know which adapter to use when executing payments
- MVP uses mock adapter, but should be easy to swap to real providers
- Users may have multiple providers (e.g., MercadoPago for payments, BIND for transfers)
- Credentials are sensitive and vary by provider

---

## Decision

**We will create a separate `adapters` entity with a many-to-one relationship from agents.**

### Schema

```sql
CREATE TABLE adapters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,        -- 'mock', 'mercadopago', 'bind'
    credentials JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    revoked_at TIMESTAMP,

    CONSTRAINT uq_user_provider UNIQUE(user_id, provider)
);

-- Agents reference a specific adapter
ALTER TABLE agents ADD COLUMN adapter_id UUID NOT NULL REFERENCES adapters(id) ON DELETE RESTRICT;
```

### Relationships

```
users ||--o{ adapters : "connects"
agents }o--|| adapters : "uses"
```

- One user can have multiple adapters (one per provider)
- One agent uses exactly one adapter
- Multiple agents can share the same adapter

### Strategy Pattern Integration

The `provider` field acts as a discriminator for the Strategy pattern defined in ADR-001:

```python
def get_adapter(adapter_row) -> PaymentAdapter:
    if adapter_row.provider == "mock":
        return MockAdapter()
    elif adapter_row.provider == "mercadopago":
        return MercadoPagoAdapter(adapter_row.credentials)
    elif adapter_row.provider == "bind":
        return BindAdapter(adapter_row.credentials)
```

---

## Consequences

### Positive

- ✅ **Clean separation** — Provider config is not mixed with user or agent data
- ✅ **Flexible credentials** — JSONB allows different fields per provider without schema changes
- ✅ **Easy to add providers** — Just INSERT a new adapter row + implement the adapter class
- ✅ **Supports multiple providers per user** — User can have both MercadoPago and BIND
- ✅ **Agent-level control** — Each agent can be restricted to a specific provider
- ✅ **Audit trail** — Adapter lifecycle tracked via events table

### Negative

- ⚠️ **Extra JOIN** — Queries for agent validation now include adapters table
- ⚠️ **Credential storage** — JSONB credentials must be encrypted at rest (application-level or DB-level)
- ⚠️ **Orphan prevention** — Must validate adapter is active before agent creation

### Neutral

- UNIQUE(user_id, provider) limits one adapter per provider per user — acceptable for MVP, can relax later
- Soft delete via `revoked_at` requires runtime validation (adapter not revoked) before payment execution

---

## Alternatives Considered

| Alternative | Pros | Cons | Reason for Discarding |
|-------------|------|------|----------------------|
| **Credentials in users table** | Simpler, fewer tables | Mixes auth with provider config; hard to support multiple providers | Violates single responsibility |
| **Credentials in agents table** | Each agent has its own config | Duplicates credentials if multiple agents use same provider; harder to revoke provider-wide | Data redundancy, management complexity |
| **Hardcoded single provider** | Simplest | Can't support user-connected accounts; no flexibility | Doesn't match business model |
| **Separate adapters table (chosen)** | Clean separation, flexible, scalable | Extra JOIN, slightly more complexity | ✅ Best balance of flexibility and simplicity |

---

## References

- **ADR-001:** Mock Adapter for Payment Processor in MVP (Strategy pattern in code)
- **DDIA Ch 4 - Encoding and Evolution:** Schema flexibility with JSONB for varying credential formats
- **Hexagonal Architecture:** Adapters as the bridge between domain and external systems

---

## Notes

### Credential Security

The `credentials` JSONB field will contain sensitive data (access tokens, API keys). Security measures:

1. **Application-level encryption** — Encrypt before INSERT, decrypt on SELECT
2. **Never log credentials** — Redact in all logging
3. **Principle of least privilege** — Only payment execution service reads credentials

### Runtime Validation

Before executing a payment, the system must verify:

```sql
SELECT ad.revoked_at
FROM agents a
JOIN adapters ad ON a.adapter_id = ad.id
WHERE a.id = :agent_id
  AND a.revoked_at IS NULL
  AND ad.revoked_at IS NULL;
```

If either is revoked, reject the authorization request.

### MVP Seeding

For MVP, each user gets a mock adapter on registration:

```sql
INSERT INTO adapters (user_id, provider, credentials)
VALUES (:user_id, 'mock', '{}');
```
