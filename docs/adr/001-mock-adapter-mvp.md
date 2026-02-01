# ADR-001: Mock Adapter for Payment Processor in MVP

## Status

**ACCEPTED** (2026-01-31)

---

## Context

PaySentry needs to execute bank transfers to CBU/Alias as part of the payment authorization flow. For the MVP, we need to decide which payment processor to use.

**Forces at play:**

1. **Main MVP use case:** Outgoing transfers (e.g., expense payment to building account)
2. **Time constraint:** MVP in 8-12 weeks, side project
3. **Budget constraint:** $0-200/month
4. **Project objective:** Architectural learning + SA portfolio, not immediate production product
5. **Design requirement:** System must demonstrate solid architecture, not just functional features

**Research conducted:**

### Option 1: MercadoPago
- **Pros:**
  - Free sandbox
  - Spanish documentation
  - Familiarity with Argentine ecosystem
- **Cons:**
  - ❌ **Doesn't support outgoing transfers via API**
  - Only allows collections (payments IN), not sending money (payouts OUT)
  - API limited to: create payments, save cards, refunds

### Option 2: BIND APIBank
- **Pros:**
  - ✅ Supports outgoing transfers (POST /transfers)
  - Complete banking API (accounts, CBU/CVU, immediate transfers)
  - Designed for fintechs
- **Cons:**
  - ❌ Requires onboarding process (KYC, compliance)
  - No instant public sandbox
  - Unknown approval time (days/weeks)

**Problem:**
The main MVP use case (automatic transfers) is not supported by the most accessible option (MercadoPago), and the option that does support it (BIND) introduces an indeterminate time blocker.

---

## Decision

**We will use a Mock Adapter for the payment processor during the MVP.**

The system will implement **Hexagonal Architecture (Ports & Adapters)** with:

1. **Port (Interface):** `PaymentProcessorPort`
   ```python
   class PaymentProcessorPort(ABC):
       @abstractmethod
       def execute_transfer(self,
                           amount: Decimal,
                           destination_cbu: str,
                           idempotency_key: str) -> TransferResult:
           pass
   ```

2. **Mock Adapter (MVP):** `MockPaymentAdapter`
   - Simulates successful/failed transfers based on configurable rules
   - Persists operations in local DB for audit trail
   - Generates realistic transaction IDs
   - Simulates network latencies (100-300ms)

3. **Real Adapter (Future):** `BindPaymentAdapter`
   - Will be implemented when BIND onboarding is complete
   - **Zero changes in business logic** when swapping adapters

---

## Consequences

### Positive

- ✅ **Unblocks MVP immediately** - We don't depend on external approvals
- ✅ **Demonstrates hexagonal architecture** - Key concept for SA (dependency inversion, testability)
- ✅ **100% controllable for testing** - Can simulate network failures, timeouts, duplicates
- ✅ **Allows fast iteration on business logic** - The core (policies, authorizations, audit) can be developed without waiting for integrations
- ✅ **Low cost** - $0 in processor fees during development

### Negative

- ⚠️ **Doesn't validate assumptions about real API** - We might discover integration issues late
- ⚠️ **Requires explicitly documenting assumptions** - The mock must be based on how we *assume* BIND works
- ⚠️ **Risk of overly simplistic mock** - If we don't properly model failures, real code could be fragile

### Neutral

- The mock must stay updated if we learn more about the real API
- When we migrate to BIND, we'll need exhaustive integration tests

---

## Alternatives Considered

| Alternative | Pros | Cons | Reason for Discarding |
|-------------|------|------|----------------------|
| **MercadoPago** | Immediate sandbox, good docs | ❌ Doesn't support outgoing transfers | Core use case is not possible |
| **BIND APIBank** | Supports real use case | ❌ Onboarding blocks MVP start | Indeterminism in timeline (incompatible with 8-12 weeks) |
| **Use payments IN instead of transfers OUT** | Unblocks MercadoPago | ❌ Completely changes use case (no longer "pay expenses", would be "charge user") | Invalidates the problem value proposition |
| **Mock Adapter (chosen)** | Unblocks MVP, demonstrates architecture | Doesn't validate real integration | ✅ It's the only one that meets: time, scope, and learning objective |

---

## References

- **DDIA Ch 1 - Maintainability:** "Good abstractions can help keep different parts of the system decoupled, making it easier to change one part without affecting another."
- **Hexagonal Architecture (Alistair Cockburn):** https://alistair.cockburn.us/hexagonal-architecture/
- **BIND APIBank Docs:** https://docs.bind.com.ar/reference/transferencias
- **MercadoPago API Reference:** https://www.mercadopago.com.ar/developers/es/reference

---

## Notes

### Assumptions about BIND's real API (to validate)

The mock will be designed assuming BIND:
- Accepts `idempotency_key` to avoid duplicates
- Returns states: `pending`, `completed`, `failed`
- Validates CBU/CVU before executing
- Can fail due to: insufficient funds, invalid CBU, timeout
- Typical latency: 200-500ms

### Criteria for migrating from Mock to BIND

We will migrate when:
1. BIND onboarding is complete
2. Core logic (policies, authorizations, audit) is stable
3. We have integration tests that validate the port contract

### Mock Implementation

The `MockPaymentAdapter` must:
- Persist each operation in `mock_transfers` table (for audit)
- Simulate random latency (100-300ms)
- Configure simulated failure rate (e.g., 5% of transfers fail with "timeout")
- Generate realistic transaction_id (e.g., `MOCK-TX-{timestamp}-{random}`)
- Log all operations as if they were real

This guarantees that business code behaves the same with the mock and with BIND.
