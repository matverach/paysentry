# ADR-003: Agent Token Storage Strategy

## Status

**ACCEPTED**

---

## Context

Agent tokens are long-lived credentials that allow AI agents to execute payment authorizations on behalf of the user. Unlike user tokens (JWT with exp=24h), agent tokens don't have automatic expiration and are only invalidated when the user revokes the agent.

**The problem:** How do we store these tokens in the database to balance security and usability?

**Forces at play:**
- **Security:** If the DB is compromised (SQL injection, accidental dump), we want to minimize damage
- **Usability:** Users may need to view/copy the token to reconfigure it
- **Simplicity:** Avoid complex key management infrastructure for an MVP
- **Compliance:** While we don't handle card data (PCI-DSS), we do handle access credentials

**Why now:** This decision impacts the design of the POST /agents endpoint and the `agents` table schema. Must be decided before implementation.

---

## Decision

**We will store agent tokens hashed in the database using bcrypt, and show them in plain text only once in the creation response.**

This means:
1. When the user creates an agent → we generate token with format `agt_{32_random_chars}`
2. We return `{agent_id, agent_token}` in the HTTP response (only once)
3. We store `bcrypt(agent_token)` in `agents.token_hash`
4. **There is no endpoint to retrieve the token later** — if lost, must create a new agent

**Pattern:** Exactly-once token delivery

---

## Consequences

### Positive
- **Security:** DB dump doesn't expose usable tokens → critical risk mitigation
- **Industry standard:** Same pattern as GitHub Personal Access Tokens, Stripe API Keys, AWS Access Keys
- **Simplicity:** Doesn't require encryption key management or additional infrastructure
- **Auditability:** Auth logs can verify token usage without exposing it

### Negative
- **UX friction:** If the user loses the token, must revoke old agent and create a new one
- **Not recoverable:** No way to "reset" or "show my token" — this may generate support overhead
- **Education:** Requires clear documentation in POST /agents explaining "copy now or lose it"

### Neutral
- Agent creation is relatively infrequent (1-5 agents per user typically)
- bcrypt cost in validation is negligible compared to network latency

---

## Alternatives Considered

| Alternative | Pros | Cons | Reason for Discarding |
|-------------|------|------|----------------------|
| Plain text storage | Simple, recoverable, can show tokens later | DB dump = total compromise of all tokens | Unacceptable security risk |
| Encrypted storage | Tokens recoverable, more secure than plain text | Requires key management, encryption keys in memory/config, if key is compromised = same as plain text | Unjustified complexity for MVP, doesn't eliminate fundamental risk |
| Hashing with bcrypt (chosen) | Secure, industry standard, simple | Not recoverable after creation | N/A — UX trade-off is acceptable to gain security |
| JWT with refresh tokens | Short tokens + refresh, known pattern | Agents need refresh logic, session management complexity | Over-engineering for MVP — agents aren't interactive users |

---

## References

- [OWASP: Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) - Also applies to long-lived tokens
- GitHub Personal Access Tokens - Same "show once" pattern
- Stripe API Keys - Identifiable prefixes (`sk_`, `pk_`) + hashed storage

---

## Notes

**Token format:** `agt_{32_chars_base62}`
- The `agt_` prefix allows identifying token type in logs (vs `usr_` for user tokens)
- Base62 (0-9, a-z, A-Z) avoids ambiguous characters and is URL-safe
- 32 chars = 190 bits of entropy (more than sufficient to prevent brute force)

**Validation implementation:**
```python
# On each request with agent_token
def validate_agent_token(token: str) -> Agent | None:
    # Extract first 8 chars as search hint (optional, optimization)
    # Do bcrypt.checkpw(token, agent.token_hash) for each active agent of user
    # If match → return agent
    # If no match → return None (401 Unauthorized)
```

**Future consideration (post-MVP):** If auth volume becomes an issue (bcrypt is CPU-intensive), we can:
1. Add cache of validated tokens (Redis with short TTL)
2. Use argon2 instead of bcrypt (more modern, better perf)
3. Implement periodic token rotation with notifications

But for MVP with <10 agents per user, direct bcrypt is sufficient.
