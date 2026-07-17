# Tier 0
Stage 1 — Spec Normalization

- Input: raw uploaded spec (OpenAPI 3.x, Swagger 2.0, Postman collection, or Insomnia export).
- Detect format (via structural markers: openapi:, swagger:, Postman's info.schema, Insomnia's _type: export), then run a format-specific parser that converts everything into one canonical internal representation: a list of endpoints, each with method, path template, path/query params, request body schema (if any), response schema per status code.
- Validate — reject malformed specs with actionable errors (e.g. "POST /orders missing response schema for 200") rather than silently producing garbage.
- Filter for Tier 0 scope: mark which endpoints are safe/idempotent (GET, HEAD) — those are eligible for generation now; others are stored in canonical form but flagged "out of scope for Tier 0," so Tier 1 doesn't need to re-parse the spec later.
- Compute a structural diff against the target's previous SpecArtifact version, if one exists (added/removed/changed endpoints) — Tier 0 doesn't act on this diff, but computing and storing it now means Tier 2's change-detection engine has data to consume from day one instead of retrofitting it.
- Persist as an immutable, versioned SpecArtifact.

Stage 2 — Knowledge Ingestion

- Input: one or more KnowledgeSources per target — uploaded PDF/DOCX, or a connected Confluence space / Linear project (credentials via the vault).
- Fetch: read uploads directly; for Confluence/Linear, pull page/issue content via their API.
- Extract text: PDF/DOCX → text extraction; Confluence/Linear content is already structured text.
- Chunk by section/heading into LLM-sized pieces.
- Extraction pass: an LLM call per chunk pulls out candidate domain invariants relevant to API testing ("prices are positive," "event dates must be in the future," "a completed order has a payment record"), each tagged with a source citation (which doc/page it came from) — this citation is what makes findings explainable later ("we flagged this because your onboarding doc says X"), not just an LLM guess.
- Graceful degradation: if one source fails (expired Confluence token, unreadable file), log a warning against that source and continue with the rest — ingestion never hard-fails the whole target over one bad source.
- Persist as a versioned KnowledgeArtifact.

Stage 3 — Generation v0

- Input: latest SpecArtifact + latest KnowledgeArtifact (knowledge is optional — see note below).
- For each safe/idempotent endpoint:
  a. Build the request: path/query param values sourced from spec-provided examples where present; otherwise synthesized by type (string sample, valid enum value, etc.).
  b. Base assertions: expect 2xx; validate required response fields exist and match declared type.
  c. Knowledge-informed assertions: match extracted invariants to response fields by name/semantic similarity (an invariant about "prices are positive" attaches to fields like price/amount) → adds a value-correctness assertion, not just shape.
  d. Attach rationale metadata to every assertion — which spec element or which cited knowledge invariant produced it.
- Open problem worth flagging now: some GET endpoints need a real resource ID (/orders/{id}) that can't be safely synthesized — guessing one just produces a useless 404. Rather than guess, Tier 0 should mark that test case needs-seed-value and surface it for the user to supply one manually before first execution, instead of silently generating a test that always fails for the wrong reason.
- Persist as a versioned TestSuite.

Stage 4 — Execution Engine

- Input: a TestSuite version + a chosen Environment.
- Resolve auth from the vault (API key / bearer / OAuth2 client-credentials — auto-refresh the token if it's near expiry).
- For each test case: substitute the environment's base URL and any user-supplied seed values, issue the HTTP call, capture status/headers/body/latency.
- Evaluate assertions → per-assertion pass/fail with expected-vs-actual detail.
- Classify each test case as pass, fail (assertion violated), or error (network/timeout/unexpected exception) — kept as a distinct category so a network blip isn't reported as a product bug.
- One retry on network-level errors before marking error, to reduce false failures — not full flaky-test detection (that's Tier 1+).
- Persist SuiteRun + per-test-case RunResults.

Stage 5 — Reporting Boundary

- Aggregate results into a per-endpoint pass/fail summary.
- For each fail/error, construct a Finding: endpoint, severity, human-readable summary ("GET /orders/{id} returned price=-5, expected positive"), evidence (request/response snapshot), and the rationale/citation carried over from generation.
- Emit Findings across a defined data contract to the external report/ticket pipeline — Tier 0 guarantees the contract, not the pipeline's internals.
- The full suite-run result (not just failures) stays queryable on its own, independent of the ticket flow