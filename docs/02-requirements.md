# PaySentry - Requirements Specification

---

## 1. Constraints

> **What they are:** Externally imposed limitations that you can't change. They define the solution space.

### 1.1 Technical

| ID | Constraint | Reason |
|----|------------|--------|
| CON-T01 | Payment processor: MercadoPago (Mock Adapter for MVP) | Argentine market, decision already made (ADR-001) |
| CON-T02 | Cloud budget: $0-200/month | Personal project, not commercial |
| CON-T03 | Backend: Python (FastAPI) | Known stack, fast prototyping |
| CON-T04 | Database: PostgreSQL | ACID required, already decided |

### 1.2 Business

| ID | Constraint | Reason |
|----|------------|--------|
| CON-B01 | No banking license required | Simplifies regulation, acts as middleware |
| CON-B02 | Argentine market only (ARS, MercadoPago) | Limited scope for MVP |

### 1.3 Time

| ID | Constraint | Reason |
|----|------------|--------|
| CON-TIME01 | Design phase: ~6-8 weeks | Side project alongside full-time work |
| CON-TIME02 | MVP implementation: ~4-6 weeks | Core functionality only |

---

## 2. Critical Use Cases

> **Why first:** All functional requirements derive from these concrete use cases.

### Use Case 1: Automatic Expense Payment (Happy Path)

**Actor:** User (Martín) + Agent (Expense Bot)

**Flow:**
1. Martín registers "Expense Bot" → receives `agent_token`
2. Martín creates policy: `{max_per_transaction: 60000, daily_limit: 100000, approval_threshold: 50000}`
3. Day 10 of month: Bot requests authorization for $45,000 to building account
4. PaySentry evaluates:
   - ✅ Amount < max_per_transaction (45k < 60k)
   - ✅ Amount < daily_limit (45k < 100k, no prior spending)
   - ✅ Amount < approval_threshold (45k < 50k) → **auto-approved**
5. PaySentry executes payment against Mock Adapter
6. Returns `{status: "captured", payment_id: "..."}` to agent
7. Martín sees notification: "Your agent paid $45,000 in expenses"

**Derived requirements:** FR-A01, FR-P01, FR-P02, FR-P03, FR-AUTH01, FR-AUTH02, FR-PAY01, FR-HIST01

---

### Use Case 2: Amount Exceeds Threshold → Manual Approval

**Actor:** User (Martín) + Agent (Expense Bot)

**Flow:**
1. Next month: expenses rise to $55,000 (building increase)
2. Bot requests authorization for $55,000
3. PaySentry evaluates:
   - ✅ Amount < max_per_transaction (55k < 60k)
   - ✅ Amount < daily_limit (55k < 100k)
   - ⚠️ Amount > approval_threshold (55k > 50k) → **requires manual approval**
4. PaySentry returns `{status: "pending_approval", authorization_id: "auth_123"}`
5. Martín receives push notification (simulated with GET /authorizations?status=pending)
6. Martín reviews and approves: `POST /authorizations/auth_123/approve`
7. Bot captures: `POST /authorizations/auth_123/capture`
8. PaySentry executes payment

**Derived requirements:** FR-P04, FR-APR01, FR-APR02, FR-AUTH03

---

### Use Case 3: Policy Violation → Automatic Rejection

**Actor:** Agent (Expense Bot)

**Flow:**
1. Bot requests authorization for $70,000 (configuration error or attack)
2. PaySentry evaluates:
   - ❌ Amount > max_per_transaction (70k > 60k)
3. PaySentry returns `{status: "denied", reason: "exceeded_max_transaction_limit"}`
4. Bot cannot capture
5. Event is logged in audit log

**Derived requirements:** FR-P02, FR-AUTH02, FR-HIST01

---

## 3. Non-Functional Requirements (NFR)

> **Why before FRs:** NFRs define the architecture. A change here changes ALL the design.

### 3.1 Performance

| ID | Requirement | Target | Justification | Verification Method |
|----|-------------|--------|---------------|---------------------|
| NFR-PERF01 | Authorization latency (p95) | < 2000ms | Acceptable limit for payment flow UX. Balances responsiveness vs implementation complexity for MVP. | Manual testing, optional load test with k6 |
| NFR-PERF02 | Minimum throughput | > 1 req/s | Sufficient for concept validation with limited load | No specific testing required for MVP |

**Rationale:** For MVP, focus is on correct architecture and reliability over premature performance optimization. Targets scale linearly by adding resources.

---

### 3.2 Reliability

| ID | Requirement | Target | Justification | Verification Method |
|----|-------------|--------|---------------|---------------------|
| NFR-REL01 | Zero double charges | 100% idempotency guaranteed | Critical for financial systems. Implements at-least-once delivery + idempotency = exactly-once semantics (DDIA Ch 7) | Integration tests with retries, manual testing with duplicate captures |
| NFR-REL02 | Complete audit trail | 100% operations logged | Compliance requirement. Implements event sourcing for state reconstruction (ADR-002) | Test: verify every POST /authorizations generates event in event_log |
| NFR-REL03 | Policy enforcement accuracy | 0% policy violations | Core value proposition of the system - reliable authorization is fundamental requirement | Unit tests + e2e tests with different policies |

**Rationale:** These NFRs define the system architecture. They are measurable, have absolute targets (not "best effort"), and are non-negotiable for a financial authorization gateway.

---

### 3.3 Availability

| ID | Requirement | Target | Justification | Verification Method |
|----|-------------|--------|---------------|---------------------|
| NFR-AVAIL01 | Uptime during demo hours | Best-effort, no SLA | It's a POC, downtime has no commercial impact. No fallback or HA for MVP. | Manual monitoring, optional UptimeRobot |

**Rationale:** For a single-tenant MVP, best-effort availability is acceptable. Architecture allows adding redundancy when scaling.

---

### 3.4 Security

| ID | Requirement | Target | Justification | Verification Method |
|----|-------------|--------|---------------|---------------------|
| NFR-SEC01 | User tokens JWT with expiration | exp = 24h, signed with HS256 | Short-lived session tokens, security/UX balance | Code review, validation tests |
| NFR-SEC02 | Hashed agent tokens | Storage with bcrypt (ADR-003) | Long-lived credentials, protection against DB dumps | Code review, verify GET /agents doesn't expose tokens |
| NFR-SEC03 | No card data storage | 0 PAN/CVV stored | PCI-DSS compliance - MercadoPago handles tokenization | DB schema audit |
| NFR-SEC04 | HTTPS in production | TLS 1.2+ | Token protection in transit | Verify deploy on Railway/Render |

**Rationale:** Security must be "correct by design", not added later. These practices are standard in systems handling credentials.

---

### 3.5 Maintainability

| ID | Requirement | Target | Justification | Verification Method |
|----|-------------|--------|---------------|---------------------|
| NFR-MAINT01 | Test coverage | > 70% for core logic (policy engine, idempotency) | Guarantees reliability in critical system components | Coverage report in CI |
| NFR-MAINT02 | Structured logs | All requests with correlation_id | Debuggability and traceability - critical for financial systems | Code review, verify JSON format |
| NFR-MAINT03 | API documentation | Complete OpenAPI 3.0 with examples | Facilitates client integration and API contract maintainability | Validate openapi.yaml covers all endpoints |

---

## 4. Functional Requirements (FR)

> **Derived from the 3 critical use cases**

### 4.1 User Management

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| FR-U01 | System must allow registering a user with email and password | Must | Given valid email and password >= 8 chars, when POST /users/register, then returns {user_id, user_token} and status 201 |
| FR-U02 | System must authenticate users with email/password | Must | Given valid credentials, when POST /auth/login, then returns JWT with exp=24h and scopes={policies:*, agents:*, authorizations:*} |
| FR-U03 | System must reject authentication with invalid credentials | Must | Given incorrect password, when POST /auth/login, then returns 401 Unauthorized |
| FR-U04 | System must validate email format in registration | Should | Given invalid email (no @), when POST /users/register, then returns 400 Bad Request with descriptive message |

---

### 4.2 Agent Management

| ID | Requirement | Priority | Acceptance Criteria | Use Case |
|----|-------------|----------|---------------------|----------|
| FR-A01 | System must allow a user to register a new agent | Must | Given valid user_token, when POST /agents with {name, description}, then returns {agent_id, agent_token} in plain text once only | UC1, UC2, UC3 |
| FR-A02 | System must store agent_token hashed (ADR-003) | Must | Agent_tokens are stored hashed in DB with bcrypt, making later recovery impossible | UC1 |
| FR-A03 | System must generate agent_tokens with identifiable format | Must | Generated tokens have format "agt_" + 32 random chars (base62), implicit scopes: {authorizations:create, authorizations:read} | UC1 |
| FR-A04 | System must allow listing user's agents | Should | Given valid user_token, when GET /agents, then returns list with {id, name, created_at, status} without exposing tokens | UC1 |
| FR-A05 | System must allow revoking an agent | Must | Given valid user_token, when DELETE /agents/{id}, then marks agent.status=revoked and rejects future authorizations with that token | UC3 |
| FR-A06 | System must validate ownership in revocation | Must | Given user_token of user X, when DELETE /agents/{id} of user Y, then returns 403 Forbidden | UC3 |

---

### 4.3 Policy Management

| ID | Requirement | Priority | Acceptance Criteria | Use Case |
|----|-------------|----------|---------------------|----------|
| FR-P01 | System must allow creating a policy associated with an agent | Must | Given valid user_token and existing agent_id, when POST /policies with {agent_id, max_amount_per_transaction, daily_limit, approval_threshold}, then returns {policy_id} and status 201 | UC1, UC2 |
| FR-P02 | System must validate maximum amount per transaction | Must | Given policy.max_amount_per_transaction=60000, when agent requests auth for 70000, then status=denied, reason="exceeded_max_transaction_limit" | UC3 |
| FR-P03 | System must validate cumulative daily limit | Must | Given policy.daily_limit=100000 and agent already spent 80000 today, when requests auth for 30000, then status=denied, reason="exceeded_daily_limit" | UC1 |
| FR-P04 | System must allow defining threshold for manual approval | Must | Given policy.approval_threshold=50000, when agent requests auth for 55000, then status=pending_approval (not automatically denied) | UC2 |
| FR-P05 | System must apply policy changes immediately | Should | Given updated policy with new max_amount, when next auth arrives <5 seconds later, then uses new limits | UC1 |
| FR-P06 | System must allow updating an existing policy | Must | Given valid user_token and existing policy_id, when PUT /policies/{id} with new values, then updates and returns 200 OK | UC2 |
| FR-P07 | System must validate that only owner can modify policies | Must | Given user_token of user X, when PUT /policies/{id} of agent of user Y, then 403 Forbidden | Security |

---

### 4.4 Authorization Flow

| ID | Requirement | Priority | Acceptance Criteria | Use Case |
|----|-------------|----------|---------------------|----------|
| FR-AUTH01 | System must validate agent token before processing | Must | Given invalid or revoked agent_token, when POST /authorizations, then 401 Unauthorized | UC1, UC3 |
| FR-AUTH02 | System must evaluate agent policy against requested amount | Must | Given policy with limits and auth request with amount, when evaluates, then returns status ∈ {approved, denied, pending_approval} according to policy rules | UC1, UC2, UC3 |
| FR-AUTH03 | System must return specific status according to evaluation result | Must | Status must clearly indicate: approved (auto-approved), denied (policy violation), pending_approval (requires manual intervention) | UC1, UC2, UC3 |
| FR-AUTH04 | System must generate unique authorization_id for each request | Must | Each POST /authorizations generates a unique authorization_id used for later capture | UC1, UC2 |
| FR-AUTH05 | System must validate authorization request structure | Must | Given request without required fields (amount, destination_cbu), then 400 Bad Request with detail of missing fields | UC1 |

---

### 4.5 Payment Execution

| ID | Requirement | Priority | Acceptance Criteria | Use Case |
|----|-------------|----------|---------------------|----------|
| FR-PAY01 | System must execute payment only if authorization is approved | Must | Given auth.status=denied or pending_approval, when POST /authorizations/{id}/capture, then 400 Bad Request | UC1, UC2 |
| FR-PAY02 | System must be idempotent in capture (NFR-REL01) | Must | Given capture already executed with authorization_id, when POST /authorizations/{id}/capture is repeated, then returns same result (same payment_id) without re-executing | UC1 |
| FR-PAY03 | System must execute against Mock Adapter in MVP | Must | For MVP, POST /authorizations/{id}/capture simulates execution and returns fake payment_id without real MercadoPago integration (ADR-001) | UC1 |
| FR-PAY04 | System must update daily_spent after successful capture | Must | Given successful capture, when updating state, then increments policy.daily_spent for future validations | UC1, UC3 |

---

### 4.6 Manual Approval

| ID | Requirement | Priority | Acceptance Criteria | Use Case |
|----|-------------|----------|---------------------|----------|
| FR-APR01 | System must allow user to approve a pending authorization | Must | Given auth.status=pending_approval, when POST /authorizations/{id}/approve with valid user_token, then auth.status → approved | UC2 |
| FR-APR02 | System must allow user to reject a pending authorization | Should | Given auth.status=pending_approval, when POST /authorizations/{id}/reject with valid user_token, then auth.status → rejected | UC2 |
| FR-APR03 | System must allow listing pending authorizations | Should | Given valid user_token, when GET /authorizations?status=pending_approval, then returns list of auths requiring approval | UC2 |

---

### 4.7 History and Audit

| ID | Requirement | Priority | Acceptance Criteria | Use Case |
|----|-------------|----------|---------------------|----------|
| FR-HIST01 | System must log each authorization attempt in event log (NFR-REL02) | Must | Every POST /authorizations generates an immutable record in event_log with timestamp, agent_id, amount, result | UC1, UC2, UC3 |
| FR-HIST02 | System must allow querying transaction history | Should | Given valid user_token, when GET /transactions, then returns list of authorizations with filters by date_range, agent_id, status | UC1 |
| FR-HIST03 | System must log authorization state changes | Must | Each state transition (pending → approved → captured) generates event in log | UC2 |

---

## 5. Prioritization (MoSCoW)

### Must Have (MVP doesn't work without this)

**Core flow (UC1 - Happy path):**
- FR-U01, FR-U02, FR-U03
- FR-A01, FR-A02, FR-A03, FR-A05
- FR-P01, FR-P02, FR-P03, FR-P04, FR-P06
- FR-AUTH01, FR-AUTH02, FR-AUTH03, FR-AUTH04
- FR-PAY01, FR-PAY02, FR-PAY03, FR-PAY04
- FR-APR01
- FR-HIST01, FR-HIST03

**Critical NFRs:**
- NFR-REL01, NFR-REL02, NFR-REL03
- NFR-SEC01, NFR-SEC02, NFR-SEC03

---

### Should Have (Important but not blocking)

- FR-U04 (email validation - nice to have)
- FR-A04 (list agents - useful but not critical)
- FR-P05 (immediate propagation - can have acceptable latency)
- FR-APR02, FR-APR03 (reject + listing - flow complement)
- FR-HIST02 (history query - demo value)

---

### Could Have (Nice to have for post-MVP)

- Pagination in GET /transactions
- Advanced filters in history
- Webhooks for notifications
- Usage metrics per agent

---

### Won't Have (Explicitly out of scope for MVP)

**Functionality:**
- Multi-currency (ARS only)
- Multiple payment processors (Mock Adapter only)
- Merchant categories in policies
- Allowed schedules in policies
- Geolocation
- Refunds/chargebacks
- Sophisticated rate limiting (beyond basic validation)
- Real push notifications (only polling with GET)
- Web dashboard (API + CLI/Postman only)

**Architecture:**
- Multi-tenancy (single-tenant POC only)
- Horizontal scaling (single instance sufficient)
- Multi-region deployment
- Distributed cache (Redis only for deduplication if needed)
- Distributed queue (synchronous processing sufficient)

**Ops:**
- Advanced monitoring (only logs + optional UptimeRobot)
- Automatic alerting
- Auto-scaling
- Disaster recovery
- Automated backup

---

## 6. Assumptions

> **What we assume is true, but could be false**

| ID | Assumption | Risk if false | Mitigation |
|----|------------|--------------|------------|
| ASM-01 | MercadoPago API is stable and well documented | Post-MVP integration delays | Use Mock Adapter in MVP (ADR-001), design with Hexagonal Architecture allows changing processor |
| ASM-02 | AI agents will send well-formed requests | More validation needed, frequent errors | Strict API validation, good error messages |
| ASM-03 | Bcrypt performance is acceptable for token validation | High auth latency if many agents | Cache recent validations (post-MVP), or switch to argon2 |
| ASM-04 | PostgreSQL on free/low-tier hosting supports expected MVP load | Downtime or throttling | Acceptable for MVP, upgrade plan documented for production |
| ASM-05 | Limited initial load (single-tenant, low concurrency) | Doesn't need multi-tenancy, scaling, HA in MVP | Architecture allows adding multi-tenancy later if project scales |

---

## 7. Success Criteria ("Done" Criterion)

The MVP is complete when:

### Functional
- ✅ The 3 use cases (UC1, UC2, UC3) work end-to-end
- ✅ E2E test demonstrates complete flow: register → policy → authorization → capture
- ✅ Executable demo script shows working system

### Non-Functional
- ✅ NFR-REL01: Test with duplicate captures returns same payment_id
- ✅ NFR-REL02: Audit log has 100% of operations
- ✅ NFR-REL03: Tests demonstrate policy enforcement

### Documentation
- ✅ Complete and validated OpenAPI spec
- ✅ README allows another dev to run project locally
- ✅ ADRs document major decisions (ADR-001, ADR-002, ADR-003)

### Deployment
- ✅ System deployed and accessible via public URL
- ✅ Functional health check endpoint
- ✅ Aggregated and accessible logs

---

## References

- **DDIA Ch 1:** Reliability, Scalability, Maintainability - Base for NFRs
- **DDIA Ch 7:** Transactions - Idempotency and exactly-once semantics
- **DDIA Ch 9:** Consistency and Consensus - Read-your-writes for policies
- **DDIA Ch 11:** Stream Processing - Event sourcing for audit trail
- **ADR-001:** Mock Adapter for MVP
- **ADR-002:** Transactional Event Log
- **ADR-003:** Agent Token Storage Strategy
