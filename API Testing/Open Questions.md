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
