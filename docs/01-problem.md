# PaySentry - Problem Statement

---

## 1. Market Context

Autonomous AI agents are moving from experimental to production. OpenAI launched Operator (January 2025), Google is developing Project Jarvis, and Visa announced Intelligent Commerce for autonomous payments.

The fundamental shift: **agents no longer just query information, they now execute actions** — including financial transactions.

**Why NOW?**
- AI agents have sufficient reliability for specific tasks
- Fintechs seek differentiation with "AI-powered" features
- Users want automation, but not at the cost of losing control

The problem is that **the authorization infrastructure doesn't exist**. OAuth solved "delegate access to data". No one has solved "delegate access to money" in a granular and auditable way.

---

## 2. The Specific Problem

### What problem exists TODAY?

**Users want to delegate payments to AI agents, but don't trust giving them total access to their funds.**

### Who suffers from it?

1. **Fintechs and wallets** that want to offer "AI assistant for payments"
   - Don't have granular authorization infrastructure
   - Risk/Compliance won't approve giving total access to an agent

2. **End users** who see value in automation but:
   - Don't trust that the agent won't make mistakes
   - Want limits: "only spend up to $X", "only at merchant Y"
   - Need to revoke access immediately

3. **AI agent developers** who:
   - Don't have a standard for "request permission to pay"
   - Each integration is custom and without traceability

### How much does it hurt?

- **Lost opportunity:** Fintechs can't launch AI agent features → lose competitiveness
- **Regulatory risk:** Giving unrestricted access → compliance problems
- **User friction:** Each payment requires manual confirmation → kills the "autonomous" value proposition

### How do they solve it today?

**Workaround 1:** Give full credentials to the agent
- Problem: No limits, no audit trail, no granular revocation

**Workaround 2:** Manual confirmation on each transaction
- Problem: Kills autonomy, high friction

**Workaround 3:** Don't offer the functionality
- Problem: Lose market share vs competitors who do innovate

**Why are they insufficient?**
Because no one solved the problem of **programmable trust**. We want to automate, but with guardrails.

---

## 3. Proposed Solution

**PaySentry is an authorization gateway that allows fintechs and wallets to delegate payments and transfers to AI agents with granular policies and complete traceability, unlike giving direct card access which has no configurable limits or audit trail.**

**Analogy:** OAuth, but for money.

---

## 4. System Actors

### Actor 1: End User (money owner)

**Who is it?**
A person who uses a fintech/wallet and wants to automate recurring payments or delegate them to an AI assistant.

**What do they need from the system?**
- Define spending policies for the agent ("maximum $50k/month", "only at Edenor", "transfers only to my savings account")
- See complete history of what the agent did
- Revoke permissions immediately if something goes wrong
- Manually approve transactions that exceed certain thresholds

**What problem does PaySentry solve for them?**
Gives them **control and visibility**. They can automate without fear that the agent will go rogue.

### Actor 2: AI Agent (delegate)

**Who is it?**
An autonomous system (e.g., payment assistant, expense bot) that executes financial actions on behalf of the user.

**What do they need from the system?**
- Request authorization to execute a payment or transfer
- Know if it was approved/denied and why
- Execute the payment if authorized
- Receive confirmation that the payment was processed

**What problem does PaySentry solve for them?**
Gives them **a standard protocol** to "ask for permission". No need to integrate directly with the payment processor, only with PaySentry.

### Actor 3: Fintech/Wallet (integrator)

**Who is it?**
The company that offers financial services and wants to add AI agent capabilities.

**What do they need from the system?**
- Ready infrastructure to offer "AI-powered payments"
- Compliance: complete audit trail, configurable policies
- Not build everything from scratch

**What problem does PaySentry solve for them?**
Gives them **fast time-to-market** and **compliance out-of-the-box**. Instead of building all the authorization logic, policies and audit, they integrate an API.

---

## 5. MVP Scope

### 5.1 WHAT'S IN (Must Have)

**Minimum end-to-end flow:**

1. **Agent registration:** User registers an agent and receives an access token
2. **Policy definition:** User creates policy with per-transaction limit and daily limit
3. **Authorization request:** Agent requests authorization for transfer of $X to account Y
4. **Automatic evaluation:** System validates against policies
5. **Manual approval (threshold):** If amount > threshold, user must approve manually
6. **Execution:** System executes the transfer against a Mock Adapter (simulated)
7. **Audit trail:** All operations are logged in immutable event log

**Essential features:**
1. JWT authentication with separate scopes (user_token, agent_token)
2. CRUD of policies (per-tx limit, daily limit, manual approval threshold)
3. Real-time policy evaluation
4. Authorization states (pending, approved, denied, captured)
5. Mock Adapter to simulate payment processor
6. Append-only event store for audit
7. Documented RESTful API (OpenAPI)

### 5.2 WHAT'S OUT (Out of Scope)

**Deferred to v2:**
- Multiple payment processors (only Mock for MVP)
- Advanced policies (merchant categories, schedules, geolocation)
- Webhooks for async notifications
- Web dashboard (API only)
- Multi-currency (ARS only)
- Refunds/chargebacks
- Sophisticated rate limiting
- Multi-tenancy

---

## 6. Success Metrics

### 6.1 Technical Metrics

| Metric | MVP Target | How to measure |
|--------|------------|----------------|
| p99 authorization latency | < 500ms | Load test with k6 |
| Uptime | > 99% | Monitoring (UptimeRobot) |
| Error rate (system errors, not denials) | < 1% | APM logs |
| Idempotency | 100% duplicate captures return same result | Integration tests |

### 6.2 Trust Metrics (Trust Indicators)

> These metrics demonstrate the system is ready to generate trust, without needing real users.

| Metric | MVP Target | Why it indicates trust |
|--------|------------|------------------------|
| Audit trail completeness | 100% operations logged | Total transparency = precondition for trust. If ONE operation is missing from the log, the system is not auditable. |
| Policy enforcement accuracy | 0% policy violations | Absolute consistency = system does what it promises. A single violation case destroys credibility. |
| Policy update propagation | < 5 seconds | Immediate control = user feels they have the power. High latency = frustration. |
| Idempotency guarantee | 100% duplicates handled | Zero double charges = trust in financial reliability. Can't be 99.9%, must be 100%. |
| Complete documentation | 100% endpoints with OpenAPI + examples | Portfolio quality: demonstrates professionalism and architectural thinking. |

---

## 7. Known Risks (Preview)

| Risk | Impact | Probability | Initial Mitigation |
|------|--------|-------------|-------------------|
| Double payment execution (not idempotent) | High | Medium | Idempotency keys in captures, duplication tests |
| Inconsistency between policy cache and DB | Medium | Medium | Short TTL in cache (60s), explicit invalidation on updates |
| Mock Adapter hides real integration issues | Medium | High | Document assumptions about real API, design with Hexagonal Architecture |
| Scope creep (adding "easy" features) | Medium | High | Maintain explicit Won't Have list, weekly review |
| Inconsistent state if process dies mid-flight | High | Low | Use DB transactions, event sourcing to rebuild state |

---

## Concrete Use Case: Automatic Expense Payment

**Character:** Martín has $45,000/month expenses due on the 10th.

**Without PaySentry:**
- Martín receives notification on the 10th
- Must open the app, enter CVV, confirm payment
- If he forgets → late fee

**With PaySentry:**
1. Martín registers "Expense Agent"
2. Defines policy:
   - Per-transaction limit: $60,000
   - Only transfers to building management account
   - Manual approval threshold: $50,000
3. On the 10th, the agent:
   - Requests authorization for $45,000 to building account
   - PaySentry validates: ✅ Within limit, ✅ Allowed account, ✅ No manual approval needed
   - Approves automatically
   - Executes transfer
4. Martín receives notification: "Your agent paid $45,000 in expenses"

**Failure scenario:**
- Next month, agent requests $55,000 (unexpected increase)
- PaySentry detects: ⚠️ Exceeds $50k threshold
- Status: `pending_approval`
- Martín receives push: "Approve $55k payment?"
- Martín reviews and approves manually

**Value demonstrated:**
- Automation of happy path (zero friction)
- Control in exceptional cases (requires approval)
- Complete audit trail (every request logged)

---

## Completeness Checklist

- [x] Each section has concrete content (no placeholders)
- [x] The problem is specific and measurable
- [x] MVP scope is realistic for 8-12 weeks
- [x] Metrics have numbers, not just "high" or "good"
- [x] I can explain this in 3 minutes to a non-technical person

---

## Next Step

**Next artifact:** `02-requirements.md` - Translate this problem into verifiable functional and non-functional requirements.
