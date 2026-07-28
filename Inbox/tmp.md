-  is item_path and collection_path redundant as in each operation, we already have path
- does the requirement need teardown
- why don't group authz into operations like what we did in resouces[]?

```yaml
resources:
  - name: order
    owner_field: userId
    tenant_field: tenantId
    operations:
      create: { method: POST,   path: /orders,      allowed_roles: [user, admin], required_scope: "write:own" }
      read:   { method: GET,    path: /orders/{id}, allowed_roles: [user, admin], required_scope: "read:own" }
      list:   { method: GET,    path: /orders,      allowed_roles: [user, admin], required_scope: "read:own" }
      update: { method: PUT,    path: /orders/{id}, allowed_roles: [user, admin], required_scope: "write:own" }
      delete: { method: DELETE, path: /orders/{id}, allowed_roles: [admin],       required_scope: "write:own" }
```

- I want the system can automatically fill the ResourceMap via inference from the knowledge base
- ~~business logic assertions (cross-field)~~
- full testing workflow
### When Requirement change
- Keep a pointer from each rule back to the paragraph it came from so you can go fetch the full text when you need it.
B. Missing workflow pieces — these need decisions

B1. Nothing says when a suite actually runs. This is the biggest hole. The maintenance loop ends at "client consumes updated suite + report" (line 323), but:
- No trigger defined for execution beyond "manual, or maintenance-driven" (line 52).
- If maintenance-driven: against which Environment? A target can have staging and production registered. Nothing picks one, and picking wrong is the destructive case §11 already flags.
- No cadence entity — no Schedule, no "run nightly against staging". The pillar-2 promise ("client closes their eyes, gets reports") requires recurring execution, and it isn't in the design.

B2. Nothing says when the first suite is generated. Post-registration flow is undefined: does uploading a spec auto-generate, or is there an explicit action? Matters for whether a client with a spec and no knowledge base gets anything at all on day one.

B3. No rate limiting or concurrency control against the client's API. negative fuzzing is explicitly uncapped (§11, deliberate), schema walks every leaf, auth is a full truth-table cross-product. On a 200-endpoint spec that's plausibly tens of thousands of requests with no throttle, no max-concurrency, and no per-request timeout defined anywhere. Against a client's production API this is indistinguishable from an attack, and it can trip their WAF or rate limiter and produce mass false failures.

B4. Credential can't hold what oauth2_cc and login need. The entity is type, label, secret (line 399). Client-credentials needs client_id + client_secret + token_url + scopes; login needs an endpoint, a body shape, and a JSONPath to the token. Also undefined: token caching and refresh across a long run, and whether a mid-run 401 triggers re-auth or reports a failure.

B5. Do quarantined tests still execute? §7 says removals "auto-retire but quarantine-and-surface rather than silently drop" — but suites are immutable versions, so it's ambiguous whether a quarantined test is absent from the new version or present and still running. §7 also argues "was this removal a bug?" shows up as a normal test failure (old test → 404), which only works if it still runs. TestCase has no lifecycle field to express either.

B6. Nothing delivers identity_slots to the client. §4.2 generates a to-do list of secrets the client owes, and the only outbound channel in the design is the Finding contract to the ticket pipeline. So the list has nowhere to go.

---
C. Smaller gaps

C1. OpenAPI does carry some authz. §5.2's table says the spec is "Absent — OpenAPI has no authz semantics" (line 210). Not quite: securitySchemes + per-operation security give OAuth2 scope names and which endpoints require auth at all. That's a free third signal — it would let pass 3 validate inferred required_scope against the API's real scope vocabulary instead of trusting an LLM-invented string.

C2. StructuralClaim IDs aren't declared stable across versions. Invariants get an explicitly stable id carried by predicate/semantic match. Claims just have id (line 416) — but §7's row "structural claim changed, source chunk hash unchanged → suppress" requires matching a claim to its previous-version self. Without stable claim identity that rule can't execute.

C3. Pass 2 doesn't scale, and has no fallback. It's one LLM call holding the full claim set plus the entire SpecArtifact endpoint list. On a large API that exceeds context, and there's no batching or degradation path — unlike pass 1, which degrades per chunk.

C4. Confidence is self-reported and never calibrated. Auto-apply hinges on a threshold, §12 correctly defers the number — but nothing in the design measures whether the LLM's confidence tracks correctness, and §11 rules out an eval harness for now. That makes the threshold unfalsifiable in practice.

C5. §10 doesn't cover the data we now store. Two additions since it was written: chunks persist client documentation verbatim, and Finding.evidence holds real response bodies. Both plausibly contain PII or secrets, in the same unencrypted DB. §10 currently discusses only Credential.

C6. CoverageSnapshot has no category breakdown. §6 promises "coverage attribution" per capability, but covered[]/uncovered[] (line 418) carry no category.

C7. No concurrency rule for the maintenance loop. Two webhook pushes in quick succession, or maintenance regenerating while a manual run is executing — no locking or ordering defined, and MaintenanceRun.status alone doesn't resolve it.