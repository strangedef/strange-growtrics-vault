## 8. Open Questions / Not Yet Decided

- [ ] Exact severity taxonomy for `Finding` (how fail vs. error map to severity levels).
- [x] Detailed testing/validation strategy for the generation and execution engines themselves (fixture specs, generation-quality evaluation approach).
- [ ] Error-handling policy beyond what's described per-stage above (e.g. partial-suite generation failures, retry limits beyond the single network retry in Stage 4).
- [ ] Whether knowledge-source refresh (re-fetching Confluence/Linear content) is on-demand only or also scheduled, even within Tier 0.

## 14. Open Questions / Not Yet Decided (Tier 1)

- [x] Exact heuristics for pagination auto-detection (which param-name patterns and response shapes count as "confident" vs. "ambiguous").
- [x] How `wrong-scope` is defined/verified when the client's auth scheme has no formal scope concept (e.g. a plain API key) — may only be meaningfully testable for OAuth2 clients.
- [ ] **Whether property-based fuzzing needs a cap on generated cases per parameter (to avoid combinatorial blow-up on endpoints with many parameters).**
- [ ] Severity mapping for negative/schema/auth findings specifically (Tier 0's open question on `Finding` severity taxonomy now needs to account for per-category differences, e.g. a BOLA failure is presumably higher severity than a missing pagination `hasNext` field).
- [ ] How do we eliminate what we don't know that we don't know

##  Deferred Items (Tier 1)
| Item | Deferred to | Current default |
|---|---|---|
| `create_via` auto-seeding + authz on mutating actions (PUT/DELETE) | **Tier 2** | Not generated; gated by Tier 1's GET/HEAD-only method scope |
| Expired-credentials mechanism | Later (non-vital) | Not generated; candidate mechanisms are a token-gen endpoint with a TTL/`exp` param or a client-supplied static expired token |
| Per-parameter fuzzing case cap | Later (no evidence yet) | Generate the full mutation set; add a cap only if combinatorial volume on wide endpoints proves a real problem |
| `Finding` severity mapping across categories | Later (no requirement yet) | No severity taxonomy; decide once real findings exist to calibrate against (shared with Tier 0's §7 deferral) |
- multiple agent design

# Knowledge base ingestion
### Generating rules/invariants
A list like:

- Event dates must be in the future
- Prices must be positive
- A completed order has a payment record
- Booking status can only be: pending, confirmed, cancelled

Each rule is one row in the database. That's it. That's the whole idea.

### One warning

Onboarding docs are full of real passwords and real customer data. Scan and strip that at ingest, before anything goes near a prompt.

### What to build in the spike

Upload docs → split into chunks → one LLM call per chunk to extract rules → tag each rule with its entity → match to endpoints by name.

### Why it's worth the extra table (fields)

Four things break without field-level links:

**Assertion construction.** "Prices must be positive" is useless unless generation knows to check `response.body.price`. A rule with no field attached can't become an assertion.

**Affected-only regeneration (Tier 2).** The spec changes `price` from integer to decimal. With field links: find rules touching `Booking.price`, regenerate just those tests. Without: you regenerate everything touching `Booking`, which is most of the suite. The whole "small, reviewable diff" promise depends on this precision.

**Conflict detection.** You can only spot "docs say X, code says Y" if both facts are keyed to the same field. No field key, no conflicts.

**Coverage attribution.** "Which rules aren't tested yet" needs to resolve to something concrete.

### But rules aren't one-field

Three shapes show up:

- **Single field** — "event_date must be in the future"
- **Cross-field** — "if status is `completed`, payment_id must be non-null" (two fields, different roles)
- **Entity or workflow level** — "a cancelled booking cannot be reactivated" (no field at all; it's about state transitions)