# API Testing — Tier 0 (Foundation) Technical Design

**Status:** Draft — brainstormed design, pending write-up review
**Scope:** Tier 0 only (per `api-testing-capability-spec.pdf` §5 tiering table)
**Grounded in:** `api-testing-capability-spec.pdf`

---

## 1. Context & Positioning

From the capability spec (§0): running API tests is not the product — Postman, Karate, and Schemathesis already do that for free. TaloTrace's differentiation is:

1. **USP Pillar 1** — test case generation grounded in app/product knowledge, not mechanical schema fuzzing.
2. **USP Pillar 2** — ongoing maintenance so a client's suite stays accurate release over release without manual upkeep.

Tier 0's job is to stand up the minimum foundation both pillars are built on: a client can register an API, connect a spec and product knowledge, get a generated suite, run it, and see results — without yet building the maintenance/regeneration engine (Tier 2) or the deeper security/protocol coverage (Tier 3-4).

## 2. Tier 0 Scope

### In scope
- Target registration: base URL + auth + environment, **multiple environments per target from the start** (dev/staging/prod sharing one suite).
- Auth support: **API key, Bearer token, OAuth2 client-credentials**.
- Spec ingestion: **OpenAPI 3.x, Swagger 2.0, Postman collection, Insomnia collection**, normalized into one canonical representation.
- Product knowledge ingestion: **uploaded documents (PDF, DOCX)** and **connected cloud doc platforms (Confluence, Linear)** — extracted into candidate domain invariants with source citations.
- Generation v0: **single-call test cases only** (no chained CRUD flows yet) — 2xx status + core field assertions, enriched with knowledge-informed value-correctness assertions where a citation applies.
- Method scope: **safe/idempotent methods only (GET, HEAD)** — no mutating calls, no cleanup/teardown logic needed.
- Manual seed values: endpoints requiring a real resource ID (e.g. `/orders/{id}`) are flagged `needs-seed-value` and the value is supplied by the user, not guessed.
- Execution: **manual trigger only** — user selects Target + Environment + Suite version and runs it on demand.
- Reporting: Tier 0 emits a defined `Finding` data contract; the existing report/ticket pipeline is treated as an external consumer, not something Tier 0 implements.

### Out of scope (deferred to later tiers)
- Chained/stateful CRUD lifecycle test generation (create → list → get → update → delete) — planned for **Tier 2**, at which point `needs-seed-value` cases can be satisfied by a prior create step instead of manual input.
- Mutating methods (POST/PUT/PATCH/DELETE) and any associated safety/cleanup logic.
- Schema/contract validation, negative/edge-case fuzzing, full auth/authz matrix (BOLA/BFLA), pagination checks — Tier 1.
- Release-over-release change detection, automatic regeneration, human-review gate on auto-updates — Tier 2.
- OWASP API Top 10 coverage, idempotency/retry-safety testing, rate-limiting, consumer-driven contract testing — Tier 3.
- GraphQL/gRPC protocol support — Tier 4 (build only on real client demand, per spec).
- Auto-run-after-generation and CI/CD-triggered execution — deferred; Tier 0 is manual-trigger only.
- Codebase-aware (GitHub-connected) generation — not part of Tier 0's baseline per the spec's tiering table.

## 3. Architecture Overview

Tier 0 is built as **one service with five clearly-bounded stages**, not a microservices split — the capability spec explicitly calls for execution infrastructure to be "fast and minimal." Each stage produces a persisted, **versioned artifact** rather than transient state, which is the one architectural investment made now specifically to avoid rework when Tier 2 (release-over-release diffing, targeted regeneration) gets built.

```
Target + Environment Registration
        │
        ▼
[Stage 1] Spec Normalization ──────► SpecArtifact (versioned)
        │
[Stage 2] Knowledge Ingestion ─────► KnowledgeArtifact (versioned)
        │         (independent of Stage 1 — either can run first, either can be re-run alone)
        ▼
[Stage 3] Generation v0 (consumes latest SpecArtifact + latest KnowledgeArtifact)
        │
        ▼
    TestSuite (versioned, single-call GET/HEAD test cases)
        │
        ▼   (manual trigger — user picks Target + Environment + Suite version)
[Stage 4] Execution Engine ────────► SuiteRun + RunResults
        │
        ▼
[Stage 5] Reporting Boundary ──────► Findings (emitted to external report/ticket pipeline)
```

**Key property:** a `TestSuite` records exactly which `SpecArtifact` version and `KnowledgeArtifact` version it was generated from. Execution is decoupled from generation — the same suite can run against any registered environment without regenerating, since assertions check response shape/values, not which environment produced them.

## 4. Stage-by-Stage Design

### Stage 1 — Spec Normalization
- **Input:** raw uploaded spec (OpenAPI 3.x, Swagger 2.0, Postman collection, or Insomnia export).
- **Detect format** via structural markers (`openapi:`, `swagger:`, Postman's `info.schema`, Insomnia's `_type: export`).
- **Parse into a canonical representation**: endpoint list, each with method, path template, path/query params, request body schema, response schema per status code.
- **Validate** — reject malformed specs with actionable errors (e.g. "POST /orders missing response schema for 200"), never silently degrade.
- **Filter for Tier 0 scope**: mark endpoints as safe/idempotent (GET, HEAD, eligible now) vs. not (stored in canonical form but flagged out of scope, so later tiers don't need to re-parse).
- **Compute a structural diff** against the target's previous `SpecArtifact` version if one exists (added/removed/changed endpoints). Tier 0 doesn't act on this diff, but storing it now avoids retrofitting Tier 2's change-detection engine later.
- **Persist** as an immutable, versioned `SpecArtifact`.

### Stage 2 — Knowledge Ingestion
- **Input:** one or more `KnowledgeSource`s per target — uploaded PDF/DOCX, or a connected Confluence space / Linear project (credentials via the vault).
- **Fetch**: read uploads directly; pull Confluence/Linear content via their API.
- **Extract text**: PDF/DOCX → text extraction; Confluence/Linear content is already structured text.
- **Chunk** by section/heading into LLM-sized pieces.
- **Extraction pass**: an LLM call per chunk extracts candidate domain invariants relevant to API testing ("prices are positive," "event dates must be in the future," "a completed order has a payment record"), each tagged with a **source citation** — the basis for explainable findings later ("flagged because your onboarding doc says X"), not an unattributed LLM guess.
- **Graceful degradation**: a single source failure (expired Confluence token, unreadable file) logs a warning against that source and does not block ingestion of the rest.
- **Persist** as a versioned `KnowledgeArtifact`.

### Stage 3 — Generation v0
- **Input:** latest `SpecArtifact` + latest `KnowledgeArtifact` (knowledge enriches generation but is not a hard blocker if absent).
- For each **safe/idempotent endpoint**:
  1. Build the request: path/query param values from spec-provided examples where present, otherwise synthesized by type.
  2. **Base assertions**: expect 2xx; validate required response fields exist and match declared type.
  3. **Knowledge-informed assertions**: match extracted invariants to response fields by name/semantic similarity (e.g. an invariant about positive prices attaches to fields like `price`/`amount`) → adds value-correctness checks beyond shape.
  4. Attach **rationale metadata** to every assertion — which spec element or which cited knowledge invariant produced it.
- **Seed values**: endpoints needing a real resource ID (`/orders/{id}`) are marked `needs-seed-value` rather than guessed, and surfaced for the user to supply manually before first execution. (Tier 2's planned full CRUD lifecycle generation — create → list → get → update → delete — removes this manual step by chaining a create step's output into later reads.)
- **Forward-compatibility note**: `TestCase` reserves (but does not use) `depends_on` and `capture` fields so Tier 2 can introduce chained, stateful test steps without redesigning the entity.
- **Persist** as a versioned `TestSuite`.

### Stage 4 — Execution Engine
- **Input:** a `TestSuite` version + a chosen `Environment`.
- Resolve auth from the credentials vault (API key / bearer / OAuth2 client-credentials — auto-refresh the token if near expiry).
- For each test case: substitute the environment's base URL and any user-supplied seed values, issue the HTTP call, capture status/headers/body/latency.
- Evaluate assertions → per-assertion pass/fail with expected-vs-actual detail.
- Classify each test case's outcome as **pass**, **fail** (assertion violated), or **error** (network/timeout/unexpected exception) — kept distinct so a network blip isn't reported as a product bug.
- One retry on network-level errors before marking `error`, to reduce false failures (not full flaky-test detection — that's Tier 1+).
- **Persist** `SuiteRun` + per-test-case `RunResult`s.

### Stage 5 — Reporting Boundary
- Aggregate results into a per-endpoint pass/fail summary.
- For each fail/error, construct a `Finding`: endpoint, severity, human-readable summary, evidence (request/response snapshot), and the rationale/citation carried over from generation.
- Emit `Finding`s across a defined data contract to the external report/ticket pipeline — Tier 0 guarantees the contract, not the pipeline's internals.
- The full suite-run result (not just failures) remains queryable independent of the ticket flow.

## 5. Data Model

| Entity                | Purpose                            | Key fields                                                                                                                                                                    |
| --------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Target**            | A registered API to test           | `id`, `app_id`, `project_id`, `name`, `spec_source_type`                                                                                                                      |
| **Environment**       | One deployment of a Target         | `id`, `target_id`, `name` (dev/staging/prod/custom), `base_url`, `auth_config_ref` (type: api_key\|bearer\|oauth2_cc + vault ref)                                             |
| **KnowledgeSource**   | A connected doc source             | `id`, `target_id`, `type` (upload\|confluence\|linear), `config`                                                                                                              |
| **SpecArtifact**      | Versioned, normalized spec         | `id`, `target_id`, `version`, `canonical_endpoints[]`, `source_format`, `diff_from_previous`, `created_at`                                                                    |
| **KnowledgeArtifact** | Versioned extraction result        | `id`, `target_id`, `version`, `sources[]`, `extracted_invariants[]` (text, source_ref, confidence)                                                                            |
| **TestSuite**         | Versioned generated suite          | `id`, `target_id`, `spec_artifact_version`, `knowledge_artifact_version`, `version`, `test_cases[]`                                                                           |
| **TestCase**          | One generated request + assertions | `id`, `suite_id`, `endpoint_ref`, `request_template`, `assertions[]` (each with rationale), `needs_seed_value: bool`, *(reserved, unused in Tier 0: `depends_on`, `capture`)* |
| **SeedValue**         | User-supplied param value          | `id`, `test_case_id`, `param_name`, `value`, `provided_by`, `provided_at`                                                                                                     |
| **SuiteRun**          | One execution of a suite           | `id`, `suite_id`, `environment_id`, `triggered_by`, `started_at`, `finished_at`, `status`                                                                                     |
| **RunResult**         | One test case's outcome            | `id`, `suite_run_id`, `test_case_id`, `http_status`, `latency_ms`, `assertion_results[]`, `evidence`, `outcome` (pass\|fail\|error)                                           |
| **Finding**           | Emitted to report/ticket pipeline  | `id`, `suite_run_id`, `test_case_id`, `severity`, `summary`, `evidence_ref`, `rationale`                                                                                      |

Versioning is the load-bearing idea: `SpecArtifact`, `KnowledgeArtifact`, and `TestSuite` are immutable and versioned, and every `TestSuite` records exactly which spec/knowledge versions produced it — the traceability Tier 2 needs to say "this test came from spec v3 + knowledge v1, the spec is now v4, here's what's affected."

`Target.app_id` / `Target.project_id` scope every target to the existing multi-user/multi-project system, matching how the rest of the platform partitions data. Tier 0 just carries these as opaque foreign keys on `Target` — the multi-tenancy model itself (permissions, project-level settings) is out of scope here.

## 6. Example Data

A worked example across one target — a client's Orders API — showing one record per entity, in the order they'd actually get created.

**Target**
```json
{
  "id": "tgt_01h9x",
  "app_id": "app_modun",
  "project_id": "proj_bookings",
  "name": "Modun Orders API",
  "spec_source_type": "openapi3"
}
```

**Environment** (two, under the same Target)
```json
[
  {
    "id": "env_01a1",
    "target_id": "tgt_01h9x",
    "name": "staging",
    "base_url": "https://staging.modun.example.com/api",
    "auth_config_ref": { "type": "bearer", "vault_ref": "vault://modun/staging-token" }
  },
  {
    "id": "env_01a2",
    "target_id": "tgt_01h9x",
    "name": "production",
    "base_url": "https://api.modun.example.com",
    "auth_config_ref": { "type": "oauth2_cc", "vault_ref": "vault://modun/prod-oauth" }
  }
]
```

**KnowledgeSource**
```json
[
  {
    "id": "ks_01b1",
    "target_id": "tgt_01h9x",
    "type": "upload",
    "config": { "filename": "modun-business-rules.pdf" }
  },
  {
    "id": "ks_01b2",
    "target_id": "tgt_01h9x",
    "type": "confluence",
    "config": { "space_key": "MODUN", "vault_ref": "vault://modun/confluence-token" }
  }
]
```

**SpecArtifact**
```json
{
  "id": "spec_01c1",
  "target_id": "tgt_01h9x",
  "version": 1,
  "source_format": "openapi3",
  "canonical_endpoints": [
    {
      "method": "GET",
      "path": "/orders/{id}",
      "path_params": [{ "name": "id", "type": "string" }],
      "response_schema": {
        "200": { "id": "string", "status": "string", "total": "number" }
      },
      "tier0_eligible": true
    },
    {
      "method": "POST",
      "path": "/orders",
      "request_body_schema": { "items": "array", "customer_id": "string" },
      "response_schema": { "201": { "id": "string", "status": "string" } },
      "tier0_eligible": false
    }
  ],
  "diff_from_previous": null,
  "created_at": "2026-07-14T09:00:00Z"
}
```

**KnowledgeArtifact**
```json
{
  "id": "know_01d1",
  "target_id": "tgt_01h9x",
  "version": 1,
  "sources": ["ks_01b1", "ks_01b2"],
  "extracted_invariants": [
    {
      "text": "An order's total must always be a positive number.",
      "source_ref": "ks_01b1#section-3.2",
      "confidence": 0.91
    },
    {
      "text": "A completed order must have status one of: pending, paid, shipped, delivered, cancelled.",
      "source_ref": "ks_01b2#page-Order-Lifecycle",
      "confidence": 0.87
    }
  ]
}
```

**TestSuite**
```json
{
  "id": "suite_01e1",
  "target_id": "tgt_01h9x",
  "spec_artifact_version": 1,
  "knowledge_artifact_version": 1,
  "version": 1,
  "test_cases": ["tc_01f1"]
}
```

**TestCase**
```json
{
  "id": "tc_01f1",
  "suite_id": "suite_01e1",
  "endpoint_ref": "GET /orders/{id}",
  "request_template": { "path_params": { "id": "{{seed:order_id}}" } },
  "needs_seed_value": true,
  "assertions": [
    { "type": "status_code", "expected": "2xx", "rationale": "base assertion (spec)" },
    { "type": "field_exists", "field": "status", "rationale": "spec: required field" },
    {
      "type": "value_positive",
      "field": "total",
      "rationale": "knowledge invariant know_01d1#0: \"An order's total must always be a positive number.\""
    }
  ]
}
```

**SeedValue**
```json
{
  "id": "seed_01g1",
  "test_case_id": "tc_01f1",
  "param_name": "order_id",
  "value": "ord_8842",
  "provided_by": "hoang.nguyen@growtrics.ai",
  "provided_at": "2026-07-17T10:15:00Z"
}
```

**SuiteRun**
```json
{
  "id": "run_01h1",
  "suite_id": "suite_01e1",
  "environment_id": "env_01a1",
  "triggered_by": "hoang.nguyen@growtrics.ai",
  "started_at": "2026-07-17T10:16:00Z",
  "finished_at": "2026-07-17T10:16:04Z",
  "status": "completed"
}
```

**RunResult**
```json
{
  "id": "res_01i1",
  "suite_run_id": "run_01h1",
  "test_case_id": "tc_01f1",
  "http_status": 200,
  "latency_ms": 142,
  "assertion_results": [
    { "type": "status_code", "passed": true },
    { "type": "field_exists", "field": "status", "passed": true },
    { "type": "value_positive", "field": "total", "passed": false, "actual": -5 }
  ],
  "evidence": { "response_body": { "id": "ord_8842", "status": "paid", "total": -5 } },
  "outcome": "fail"
}
```

**Finding**
```json
{
  "id": "find_01j1",
  "suite_run_id": "run_01h1",
  "test_case_id": "tc_01f1",
  "severity": "high",
  "summary": "GET /orders/ord_8842 returned total=-5, expected a positive number.",
  "evidence_ref": "res_01i1",
  "rationale": "knowledge invariant know_01d1#0: \"An order's total must always be a positive number.\" (source: modun-business-rules.pdf, section 3.2)"
}
```

## 7. Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Architecture shape | Staged pipeline, single service, persisted versioned artifacts (not sync-only, not microservices) | Matches spec's "fast and minimal" execution-infra guidance while directly setting up Tier 2's diff/regeneration needs |
| Product knowledge input | Free-text docs (PDF/DOCX) + Confluence/Linear connectors | Matches how clients actually hold product knowledge; requires an extraction step, not structured rule entry |
| Auth support | API key, Bearer token, OAuth2 client-credentials | Covers the realistic majority of client APIs at this stage |
| Environments | Multiple environments per target from the start | Low added complexity now vs. retrofitting later; one suite naturally applies across dev/staging/prod |
| Spec formats | OpenAPI 3.x, Swagger 2.0, Postman collection, Insomnia collection | Covers clients without a formal OpenAPI doc but with an internal collection |
| Test generation depth | Single-call tests only, no chained CRUD | Keeps Tier 0 scoped to the MVP functional-testing bar; chaining is explicitly deferred to Tier 2 |
| HTTP method scope | Safe/idempotent only (GET, HEAD) | Removes all cleanup/teardown and destructive-call safety concerns from the foundation tier |
| Seed values for path params | User-supplied manually, flagged `needs-seed-value` | Avoids guessing IDs that produce meaningless 404s; superseded once Tier 2 adds create-then-use chaining |
| Execution trigger | Manual only | Matches the taxonomy's MVP bar for CI/CD ("manual trigger"); automation is Tier 1+ |
| Report/ticket integration | Tier 0 defines and emits a `Finding` contract; pipeline internals treated as an external boundary | Keeps Tier 0 buildable without dependency on undocumented pipeline internals |

## 7. Open Questions / Not Yet Decided

- Exact severity taxonomy for `Finding` (how fail vs. error map to severity levels).
- Detailed testing/validation strategy for the generation and execution engines themselves (fixture specs, generation-quality evaluation approach).
- Error-handling policy beyond what's described per-stage above (e.g. partial-suite generation failures, retry limits beyond the single network retry in Stage 4).
- Whether knowledge-source refresh (re-fetching Confluence/Linear content) is on-demand only or also scheduled, even within Tier 0.
