# API Testing — Unified Technical Design

**Status:** Draft — brainstormed design, pending review
**Scope:** The full API-testing system through the maintenance engine (Tier 0–2 of `api-testing-capability-spec.pdf` §5). Organized by capability; build order is in Appendix A.
**Grounded in:** `api-testing-capability-spec.pdf`

---

## 1. Positioning

Running API tests is not the product — Postman, Karate, and Schemathesis do that for free. TaloTrace's differentiation is two USP pillars:

1. **Grounded generation** — test cases informed by the client's app/product knowledge (and, later, codebase), not mechanical schema fuzzing.
2. **Continuous maintenance** — the suite stays accurate release over release with zero client upkeep. The client "closes their eyes"; TaloTrace watches the spec, regenerates affected tests, and surfaces results.

Everything below serves one or both. Execution/reporting is necessary plumbing — built fast and minimal, not the sell.

## 2. Core Principles

- **Versioned immutable artifacts.** `SpecArtifact`, `KnowledgeArtifact`, `ResourceInference`, `TestSuite`, and `CoverageSnapshot` are versioned and immutable; a suite records the exact spec + knowledge (+ inference) versions it came from. This traceability is the backbone the whole maintenance engine is built on — not a retrofit.
- **Coverage contract vs. best-effort enrichment.**
  - A `TestSuite`/`SpecArtifact` is a *coverage contract* — never silently incomplete, so building one is **all-or-nothing** (one failure → nothing persisted).
  - A `KnowledgeArtifact` and a `ResourceInference` are *best-effort enrichment* — partial is useful and not misleading, so both **degrade gracefully** (ingestion per source and per chunk; inference per claim and per operation).
  - At *execution* time, test cases are independent atomic units, so a run **continues past** individual failures and reports a partial result.
- **Automatic maintenance, client hands-off.** Change detection, diffing, and regeneration fire with zero client action. The only human in the loop is a **TaloTrace reviewer**, pulled in solely on low confidence or a capability gap.
- **Explainability, down to the source sentence.** Every assertion carries a `rationale` naming the spec element or cited knowledge invariant that produced it, and a knowledge-sourced assertion resolves further — invariant → chunk → the client's **verbatim prose** (§5.1). "Your doc says *X*" is defensible to a client; "our extractor believes *X*" is not. Same chain is the basis for precise change-mapping.

## 3. Architecture

One service. A generation/execution pipeline feeds a reporting boundary; a maintenance loop wraps the whole thing and re-drives generation when the client's spec changes.

```
                 ┌──────────────────────── MAINTENANCE LOOP ───────────────────────┐
                 │  SpecWatch detects spec change (GitHub webhook / poll)           │
                 │      → re-ingest → diff → regenerate affected → gate → snapshot   │
                 ▼                                                                   │
  Registration & Inputs                                                             │
  (Target, Environment, Credential, KnowledgeSource, ResourceMap, AuthProfile,      │
   SLAConfig, PaginationOverride, SpecWatch)                                        │
        │                                                                           │
        ▼                                                                           │
  [1] Spec Normalization ─────► SpecArtifact (versioned, + diff_from_previous) ─────┘
        │
  [2] Knowledge Ingestion ────► KnowledgeArtifact (versioned)
        │   (independent of [1] — either runs/re-runs alone)
        ▼
  [2b] Resource Inference ────► ResourceInference (versioned) ──► merged into
        │   (spec + knowledge → resource model)  ResourceMap / AuthProfile
        ▼
  [3] Generation ─────────────► TestSuite (versioned): TestCases + TestScenarios
        │
        ▼   (trigger: manual, or maintenance-driven)
  [4] Execution ──────────────► SuiteRun → RunResults / ScenarioRuns
        │
        ▼
  [5] Reporting Boundary ─────► Findings → external report/ticket pipeline
                                CoverageSnapshot (per suite version)
```

Execution is decoupled from generation: one suite runs against any registered `Environment` unchanged, since assertions check response shape/values, not which environment produced them.

## 4. Registration & Inputs

Client-declared, per target. All optional except the spec — the system tests whatever it's given and notes what it couldn't.

`ResourceMap` and `AuthProfile` are the two inputs the system can also **infer for itself** from the client's knowledge base (§5.2), so "client-declared" means *may be* declared: anything the client leaves out, inference tries to fill, and anything the client does declare is never overwritten.

- **Target** — the API under test, scoped to the platform's multi-tenant system via `app_id`/`project_id` (opaque foreign keys; tenancy model itself out of scope).
- **Environment** — a deployment (dev/staging/prod); base URL + a `Credential`. Multiple per target, all sharing one suite.
- **Credential** — a stored client API secret (see §10 security note). Types: `api_key | bearer | oauth2_cc | login`.
- **KnowledgeSource** — uploaded PDF/DOCX or connected Confluence/Linear. Auth + fetch is existing platform infra; the system consumes already-retrieved content.
- **Spec** — OpenAPI 3.x, Swagger 2.0, Postman collection, or Insomnia export. Required.
- **SpecWatch** — where to auto-fetch the spec from for maintenance: `github_webhook` (instant on push/merge) or `poll_url` (scheduled hash+diff).
- **ResourceMap** — the shared resource model (§4.1). Drives both auth checks and CRUD-lifecycle generation.
- **AuthProfile** — roles + credentials-per-role (§4.2). Source of the authz truth table.
- **SLAConfig** — per-endpoint latency threshold. No config → no SLA test, no hardcoded default.
- **PaginationOverride** — client-declared pagination style for an endpoint the detector marked ambiguous.

### 4.1 `ResourceMap` — shared resource model

Each resource declared once. Per CRUD verb, `operations` carries the endpoint (`method` + `path`) **and** its authz rule (`allowed_roles` + `required_scope`) inline. CRUD-lifecycle generation reads the endpoint fields; the auth capability reads the authz fields. `owner_field`/`tenant_field` support BOLA ownership checks.

```yaml
resources:
  - name: order
    owner_field: userId
    tenant_field: tenantId          # omit if single-tenant
    operations:
      create: { method: POST,   path: /orders,      allowed_roles: [user, admin], required_scope: "write:own" }
      read:   { method: GET,    path: /orders/{id}, allowed_roles: [user, admin], required_scope: "read:own" }
      list:   { method: GET,    path: /orders,      allowed_roles: [user, admin], required_scope: "read:own" }
      update: { method: PUT,    path: /orders/{id}, allowed_roles: [user, admin], required_scope: "write:own" }
      delete: { method: DELETE, path: /orders/{id}, allowed_roles: [admin],       required_scope: "write:own" }
```

**Declaration is optional.** A `ResourceMap` the client never writes is inferred from their knowledge base and validated against the spec — full mechanism in §5.2. Every field carries `provenance` (`client_declared` | `inferred_knowledge` | `inferred_spec`) so the two origins never blur, and client-declared values always win.

### 4.2 `AuthProfile` — roles + credentials

```yaml
roles:
  - { name: admin, scopes: ["*"] }
  - { name: user,  scopes: ["read:own", "write:own"] }
  - { name: guest, scopes: ["read:public"] }

credentials:
  admin: [ { id: adminA, credential: <token / api-key / login details> } ]
  user:  [ { id: userA, credential: <...> },
           { id: userB, credential: <...> } ]   # 2nd same-role identity → enables BOLA/BFLA replay
  guest: [ { id: guestA, credential: <...> } ]
```

The generator joins `AuthProfile` (roles × credentials) with `ResourceMap.operations` (allowed_roles + required_scope) to compute an **authz truth table**: the expected allow/deny for every (identity, operation) pair. Identity count per role is unbounded and optional — the system tests what the provided identities allow and skips-with-note the rest.

**Roles and scopes are inferable; credentials are not.** Docs that describe authorization name the roles ("admins may cancel an order") — so §5.2 infers `roles[]` and their scopes, and must, since an inferred `allowed_roles: [admin]` is meaningless unless `admin` exists here. Credentials are secrets only the client holds. Inference therefore emits **`identity_slots[]`** — a generated to-do list naming what the client still owes:

```yaml
identity_slots:
  - { role: admin, needed: 1, reason: "delete is admin-only" }
  - { role: user,  needed: 2, reason: "2nd same-role identity enables BOLA/BFLA replay" }
```

Until a slot is filled the rule above applies unchanged — test what the provided identities allow, skip-with-note the rest. A slot is guidance, never a blocker.

## 5. Pipeline Stages

**[1] Spec Normalization**
- Detect format via structural markers; parse into a canonical endpoint list (method, path template, params, request-body schema, response schema per status), preserving the **full nested schema tree** (objects, arrays of objects).
- Validate. **All-or-nothing:** one malformed endpoint blocks the whole `SpecArtifact` until fixed.
- Flag each endpoint's methods; compute `diff_from_previous` vs. the prior version (raw material for maintenance).
- Persist as an immutable, versioned `SpecArtifact`.

**[2] Knowledge Ingestion** (chunking detail in §5.1)
- Extract text → chunk by section/heading → one LLM call per chunk extracts candidate domain invariants, each with a **structured `source_ref`** (`chunk_id` + quote span, §5.1) and a **stable `id`** (carried across `KnowledgeArtifact` versions by predicate/semantic match, so a rule is trackable release to release). Two forms: **single-field** ("prices are positive") and **cross-field / conditional** ("if `status` is `completed`, `payment_id` must be non-null"). Where possible the LLM emits a **structured predicate** (`field`/`op`/`value`, or `when → then`) alongside the text, so downstream evaluation is deterministic code, not an LLM call per assertion.
- **Graceful degradation** per source and per chunk — a failed chunk drops its candidates, never the artifact. Every drop is recorded in `ingestion_notes[]` so a thinner artifact is visible, not silent.
- Persist chunks **and** invariants as one versioned `KnowledgeArtifact`.

**[2b] Resource Inference** (detail in §5.2)
- Input: latest `SpecArtifact` + latest `KnowledgeArtifact`. Runs when either changes; skipped entirely if the client declared a complete `ResourceMap` **and** `AuthProfile`.
- Three passes: per-chunk **structural-claim** extraction → cross-chunk **reconciliation** binding claims to real endpoints → deterministic **spec validation** dropping any operation the spec doesn't back.
- **Graceful degradation**, like [2] and unlike [3]: a failed chunk drops its claims, a rejected operation drops that operation — never the run. Recorded in `inference_notes[]`.
- Persist the proposal as a versioned, immutable `ResourceInference`; merge its confident fields into the live `ResourceMap`/`AuthProfile`, never overwriting client-declared values.

**[3] Generation** (detail per capability in §6)
- Input: latest `SpecArtifact` + latest `KnowledgeArtifact` (knowledge enriches, not required) + `ResourceMap`/`AuthProfile`/`SLAConfig` where present — declared, inferred, or a mix; generation reads one merged view and does not care which.
- Produces `TestCase`s (single-call, capability-tagged) and `TestScenario`s (chained CRUD lifecycles). Every assertion carries a human-readable `rationale` **and** a structured `source` ref (spec element, or `invariant_id` + `knowledge_version`) — the machine link that lets maintenance map a spec/knowledge change to exactly the assertions it affects. Every case/suite gets an editable `name`.
- **All-or-nothing:** any generation failure → no partial suite. Persist as a versioned `TestSuite`; record `generation_notes[]` (what was skipped and why).

**[4] Execution**
- Input: a `TestSuite` version + a chosen `Environment`. Resolve auth from the referenced `Credential`(s) per identity.
- Single-call cases: substitute base URL / seed / identity, issue, evaluate assertions. **Request dedup** — identical `functional`/`schema`/`sla` requests issue once, evaluate all three against the one response.
- Scenarios: run steps as a **DAG by `depends_on`**; on a step fail/error, mark transitive dependents `skipped`, continue other branches.
- Outcome per unit: `pass | fail | error | skipped`. One retry on network error before `error`. **Continue on failure** — a partial report beats aborting.
- Persist `SuiteRun` + `RunResult`s / `ScenarioRun`s.

**[5] Reporting Boundary**
- Build `Finding`s (endpoint/scenario, category, severity, summary, evidence, carried-over rationale) and emit across a defined contract to the external report/ticket pipeline (an external consumer, not built here).
- `generation_notes` + `CoverageSnapshot` surface alongside results. Full run results stay queryable independent of the ticket flow.

> **LLM-call retries (stages 2–3):** follow the codebase's shared LLM-client retry/backoff convention — not a policy invented here.

> **Why [2b] sits between ingestion and generation:** it consumes both artifacts and produces no tests, only the resource model that generation needs. Placing it inside generation would make an all-or-nothing stage depend on a best-effort one.

### 5.1 Chunking & the audit chain

Detail for stage [2]. Chunking is not just an ingestion mechanic — the chunk is the unit that makes a knowledge-sourced assertion auditable and makes knowledge-diff noise separable from real change.

**Chunking is by section/heading boundary** — not fixed-token, not sliding window. Business rules live in prose sections, and a heading boundary keeps a rule together with its qualifiers in one LLM window; a cross-field conditional ("if `status` is `completed`, `payment_id` must be non-null") is one sentence in one section, and a token window that splits it across two calls loses either the `when` or the `then`. Nesting is preserved via `heading_path[]` (e.g. `["Orders", "3.2 Totals"]`), so a citation reads the way a human would cite the doc.

**Chunks are first-class and persisted.** A chunk is not a transient ingestion buffer — it is the evidence a rule was actually written down:

| Field | Purpose |
|---|---|
| `id` | Stable handle an invariant cites |
| `source_id` | Which `KnowledgeSource` it came from |
| `heading_path[]` | Human-readable location (`modun-rules.pdf § Orders › 3.2 Totals`) |
| `char_start` / `char_end` | Exact span in the extracted text — survives heading renumbering |
| `text` | Verbatim source prose, unparaphrased |
| `text_hash` | Content fingerprint — the fuzzy-diff discriminator (§7) |

**The audit chain — a finding back to the sentence that justifies it.** Four hops, every one a structured ref, no text search anywhere:

```
Finding.rationale
  └─► TestCase.assertions[].source { kind: knowledge, invariant_id, knowledge_version }
        └─► KnowledgeArtifact vN .extracted_invariants[id]
              └─► invariant.source_ref { chunk_id, quote_span }
                    └─► KnowledgeArtifact vN .chunks[chunk_id].text  ← verbatim client prose
```

Because `knowledge_version` is pinned on the assertion (and `knowledge_artifact_version` on the suite), the chain is **reproducible**: an old finding resolves against the exact artifact version that produced it, not against today's docs. `quote_span` narrows the citation from "somewhere in §3.2" to the precise sentence. The spec side has the mirror chain — assertion `source.endpoint_ref` + field path into the pinned `SpecArtifact` version.

**Why the verbatim text, not just the paraphrase.** The invariant's `text` is LLM-written; the chunk's `text` is the client's. Three consumers need the original:
- the **review gate** (§7) — a reviewer judging a fuzzy knowledge diff needs the source prose to compare against, not two LLM paraphrases of it;
- the **client-facing finding** — "your doc says *X*" is defensible; "our extractor believes *X*" is not;
- the **fuzzy-diff guard** (§7) — `text_hash` is what separates extraction jitter from a real doc edit, and it cannot be computed after the fact.

**Conflicting invariants across chunks.** Two chunks can emit contradicting predicates over the same field (one section updated, another left stale). Detection is deterministic — same field, incompatible `op`/`value`. Resolution: **never silently pick one**. Both are retained on the artifact with `status: conflicted`, **neither generates an assertion**, and the pair becomes a `ReviewRequest` citing both chunk texts. A contradiction in the client's own docs is a finding about the docs — surfacing it beats guessing which section is current.

### 5.2 Resource Inference — building the `ResourceMap` from the knowledge base

Detail for stage [2b]. The `ResourceMap` (§4.1) and `AuthProfile` (§4.2) are the highest-value inputs and the ones clients are least likely to fill in: writing them means hand-transcribing a resource model and an authorization matrix the client already documented in prose. Inference reads that prose instead.

**What each input can and cannot tell us.** The division of labor is not arbitrary — it follows from what each source structurally contains:

| Question | Spec | Knowledge base |
|---|---|---|
| Which endpoints exist? | **Authoritative** — enumerated | Implied at best |
| Which endpoints form one resource? | Guessable by path shape | Stated by name |
| Who may call an operation (`allowed_roles`, `required_scope`) | **Absent** — OpenAPI has no authz semantics | Stated in prose |
| Which field marks ownership (`owner_field`, `tenant_field`) | A field name with no meaning attached | Stated: "each order belongs to a user" |
| Do the roles even exist, and what can they do? | Absent | Stated |

The spec knows *what exists*; the knowledge base knows *what it means and who may touch it*. Neither alone produces a `ResourceMap`. So the knowledge base proposes the model and **the spec is the validator** — a proposed operation that names no real endpoint is dropped, not guessed at.

#### Three passes

**Pass 1 — per-chunk structural-claim extraction.** Rides the same §5.1 chunk loop as invariant extraction: one additional LLM call per chunk, emitting `StructuralClaim`s, each citing `chunk_id` + `quote_span` exactly as an invariant does. A claim is one atomic assertion about the API surface:

| `kind` | Example claim from prose |
|---|---|
| `resource` | "order" is a resource |
| `operation` | orders can be cancelled (→ a delete-like operation) |
| `ownership` | an order belongs to a user (→ `owner_field`) |
| `role` | `admin`, `user`, `guest` exist |
| `scope` | `user` may read and write their own data |
| `authz_rule` | only `admin` may cancel an order |

Per-chunk degradation is inherited: a chunk that fails extraction drops its claims and is logged in `inference_notes[]`; the run continues.

**Pass 2 — cross-chunk reconciliation.** One call with the full claim set plus the `SpecArtifact` endpoint list. This is the step that needs a global view: a resource is introduced in §2, its ownership rule appears in §5, its delete permission in §7, and only a pass that sees all three can assemble one resource. It groups claims into resources, binds each operation to a concrete `method` + `path`, resolves role names to a single vocabulary, and assigns a `confidence` per field.

**Pass 3 — deterministic spec validation.** No LLM. Every proposed `method` + `path` must exist in the `SpecArtifact`, and every `allowed_roles` entry must name a role in the (declared or inferred) `AuthProfile`. Failures drop that **operation**, never the resource, with an `inference_notes` entry. This pass is what makes the two-signal rule below enforceable in code rather than judged by a model.

#### The confidence gate

Every inferred field carries a `confidence`. Three outcomes:

| Condition | Outcome |
|---|---|
| `confidence` ≥ threshold | **Auto-apply** — field lands in the live map tagged `inferred_knowledge` |
| `confidence` < threshold | Field left **unset**; `ReviewRequest` to the TaloTrace reviewer, quoting the citing chunk |
| Claims conflict across chunks | Both retained, field unset, `ReviewRequest` citing both — same rule as conflicting invariants (§5.1) |

An unset field never blocks its neighbours: a resource with a confident `read` and an unclear `delete` generates read tests now and gains the delete lifecycle after review. Review routes to TaloTrace's internal console (§7), never to the client — the hands-off promise holds.

#### Two-signal rule for mutating operations

Pass 3 requires spec backing for *every* operation. Mutating operations require a **second, independent signal** on top:

- **`read` / `list`** — spec-shape symmetry suffices. `GET /orders` + `GET /orders/{id}` is adequate evidence the resource is readable.
- **`create` / `update` / `delete`** — the knowledge must **explicitly describe the operation**. `DELETE /orders/{id}` merely existing in the spec does *not* license a delete test; a chunk saying "admins may cancel an order" does. Structural symmetry alone → operation dropped with a note.

Same shape as the pagination two-signal rule (§6): one request-side signal, one content-side signal, never one alone.

**Why auto-applying a destructive operation is survivable.** A `delete` only ever runs inside a CRUD `TestScenario`, whose delete step targets an ID captured from that same scenario's `create` step (`create_via`, §6). An auto-applied delete therefore removes **a row the suite just created**, not pre-existing client data. Two consequences follow, both worth stating plainly:

- If inference finds no `create` for a resource, there is no `create_via`, so **no delete test is generated at all** — the dangerous case excludes itself.
- The residual risk is a wrongly-inferred delete *path* colliding with another resource's. Reaching it requires the spec-existence check **and** the explicit-description rule to fail simultaneously.
- §11's accepted mutating-environment risk (a client pointing a run at production) is unchanged by this feature. Inference does not widen it beyond self-created rows.

#### Merge: how declared and inferred coexist

Two objects, one concept, deliberately:

- **`ResourceInference`** — versioned, immutable. The *proposal*: what was inferred, from which chunk, at what confidence, and what was dropped. Gives the maintenance loop something to diff and a reviewer something to read.
- **`ResourceMap` / `AuthProfile`** — the single *live view* generation reads, with `provenance` on every field.

Merge rule, in order: **`client_declared` > `inferred_knowledge` > `inferred_spec`**. Client values are never overwritten, only marked as superseding an inferred claim (kept for audit). Inference fills holes at field granularity, so a client who declares `owner_field` and nothing else gets exactly the rest filled in.

Without the immutable half, a re-run silently rewrites the resource model and nothing can diff it. Without the merged half, every generator grows merge logic.

#### Audit chain, extended

Inferred fields reuse §5.1's chain wholesale — no new machinery:

```
ResourceMap.operations.delete.allowed_roles = [admin]
  └─► claim_ref
        └─► ResourceInference vK .claims[id].source_ref { chunk_id, quote_span }
              └─► KnowledgeArtifact vN .chunks[chunk_id].text  ← "Only admins may cancel an order."
```

Every inferred field answers *"why do you think that?"* with a sentence the client wrote. This is the difference between an inferred resource model and a guessed one, and it is what makes reviewing a low-confidence proposal a ten-second job rather than an investigation.

## 6. Test Capabilities

All capabilities generate capability-tagged `TestCase`s (or `TestScenario`s) so results and coverage read per category.

- **`functional`** — happy-path: 2xx + required fields exist and match type, enriched with **knowledge-informed value-correctness** assertions (an invariant about positive prices attaches to `total`/`amount`).
- **`business_logic`** — cross-field / conditional invariants: relationships between two or more fields, e.g. *"if `status` is `completed`, `payment_id` must be non-null"*, *"`ends_at` > `starts_at`"*, *"a refunded order has `refund_id` set"*. This is the core "business-logic assertions, not just schema shape" differentiator — the value a schema alone can never express. Represented as a `conditional` assertion (`when → then`, both referencing fields); evaluated by **deterministic code** (no LLM at run time), so runs stay fast and reproducible. If the antecedent doesn't hold on a given response, the assertion is **not-applicable** (vacuous pass, reported N/A — not a fail). Applies to any response carrying the referenced fields — single-call reads and scenario step responses alike (e.g. after an `update` sets `status=completed`, the read-back step asserts `payment_id` is non-null).
- **`schema`** — full nested conformance: recursive tree walk, one presence+type assertion per leaf at any depth, array-wildcard paths for list items (`line_items[].quantity`), evaluated per-array-item with per-index failure evidence.
- **`negative`** — property-based, schema-driven fuzzing: mutate each parameter and (for mutating ops) each request-body field against its own constraint (wrong type, out-of-range, invalid enum, oversized, missing-required) → expect a safe 4xx, never a 5xx or silent accept. No per-parameter cap (revisit only on evidence of blow-up). Standalone, not chained.
- **`auth`** — from the authz truth table:
  - no-creds / invalid-creds (unconditional → expect 401);
  - wrong-scope / role-denied incl. **BFLA** (every should-deny pair → 403/401);
  - **BOLA** (same-role identity B requests identity A's object → 403/404).
- **`pagination`** — LLM-assisted detection needing **two agreeing signals** (a request param set: offset/limit, page/size, or cursor; **and** a response marker: total-count, `has_next`/`next_cursor`, page-metadata). Confident → generate boundary-page + metadata-consistency cases; ambiguous → skip-with-note + offer `PaginationOverride`; not-paginated → nothing.
- **`sla`** — for each endpoint with an `SLAConfig`, assert `latency_ms <= max_latency_ms` (single-request check, not load testing).
- **CRUD lifecycle (`TestScenario`)** — a first-class ordered lifecycle per resource: `create → read → list → update → (read-back) → delete → (read-404)`. Tests real **state transitions** (create is readable, update persists, delete removes) a single-call suite can't reach.
  - **Chaining (hybrid):** named `capture` (from response **body** JSONPath, **header** `Location`, or **status**) threads values into later `path`/`query`/`body`; assertions may also address a prior step directly (`s1.response.total`).
  - **Body synthesis:** schema + knowledge invariants + spec examples, LLM-assisted, with **run-unique injection** for unique/format-constrained fields (email, `external_id`) so re-runs don't 409.
  - **Auth on writes:** `create_via` auto-seeds a resource per identity (so BOLA/BFLA extend to update/delete), replacing the single-call fallback of a client-supplied existing ID.
  - `schema` and `negative` apply to **every** CRUD operation: schema rides each step's response (no extra calls); negative body-fuzzing is standalone per mutating op.
  - **Seed values** (`needs_seed_value` on a `TestCase`) remain only for single-call reads on resources without a declared lifecycle — a scenario's `create` step otherwise supplies the ID.
  - **On an inferred resource** (§5.2) the lifecycle is generated only for operations that cleared the two-signal rule. No inferred `create` ⇒ no `create_via` ⇒ no `delete` step, so an auto-applied destructive operation can only ever remove a row the scenario itself created.

## 7. Maintenance Engine

The differentiator. TaloTrace watches **both inputs that a suite is generated from — the client's spec and their business knowledge** — and regenerates affected tests itself; the client does nothing. The inferred resource model (§5.2) is derived from those same two inputs, so it re-derives on the same triggers and diffs as a third `ChangeSet` input rather than needing a watcher of its own.

```
Change detected on EITHER input:
  • SpecWatch: spec change (GitHub webhook, or poll a registered spec URL)
  • Knowledge re-ingestion: platform refresh of a KnowledgeSource, or an updated upload
   → re-ingest → new SpecArtifact vN / KnowledgeArtifact vN → diff vs vN-1 → ChangeSet (classified)
   → re-run resource inference (§5.2) → new ResourceInference vK → diff vs vK-1 → ChangeSet
   → map each change to affected tests (via structured source refs — see Traceability below)
   → per change: regenerate / retire / add-assertion → proposed TestChange (+confidence)
   → GATE:  confident + capable → auto-apply
            low confidence OR capability gap → ReviewRequest (TaloTrace reviewer)
   → apply auto + approved changes → new TestSuite version → compute CoverageSnapshot
   → client consumes updated suite + report (did nothing)
```

**Trigger — fully automatic, no manual client action.** Spec: GitHub-connected clients get instant webhook regeneration, everyone else a polled spec URL. Knowledge: re-ingestion already happens via the platform's refresh infra (and on any re-upload); its output feeds the same loop. One-time setup, then hands-off.

**Traceability — how a change maps to affected tests.** The linkage is first-class, not free text:
- Each generated assertion carries a structured **`source`** ref: for schema/functional, the spec element (`endpoint_ref` + field path); for `business_logic`/value assertions, the `invariant_id` + `knowledge_version` it came from.
- Each `KnowledgeArtifact` invariant has a **stable `id`** (carried across versions by predicate/semantic match), and its `predicate` names the field(s) it constrains.
- Each invariant's `source_ref` resolves to a persisted **chunk** (`chunk_id` + quote span, §5.1), so the chain runs one hop further — down to the client's verbatim prose.
- So the reverse indexes fall out: *field → invariants* (predicate fields), *invariant → assertions → test cases* (source refs), *spec element → assertions* (source refs), *chunk → invariants* (source refs). A change to `total`, or to invariant `inv_completed_has_payment`, resolves directly to the exact assertions to regenerate — true "affected only", minimizing churn (which protects the false-positive trust budget). The chunk index runs the same way in reverse: an edited doc section resolves straight to the invariants it justified.

**Change → action — spec:**

| Spec change | Action |
|---|---|
| Endpoint added | Generate new tests (full generation for it) |
| Endpoint removed | Retire affected tests + **quarantine note** (never silent-delete) |
| Endpoint signature change (path/method) | Regenerate |
| Response field added | Add schema assertion; knowledge-match for a value/business-logic assertion |
| Response field removed | Retire assertions referencing it + quarantine note |
| Field type/constraint changed | Regenerate affected schema + negative assertions |
| Field deprecated | Flag / retire |

**Change → action — business knowledge** (invariants diffed by stable `id`):

| Knowledge change | Action |
|---|---|
| Invariant added | Generate `business_logic`/value assertions for it on endpoints whose response carries the predicate's field(s) |
| Invariant removed | Retire assertions sourced from it + **quarantine note** |
| Invariant changed (predicate/threshold/enum), source chunk changed | Regenerate the assertions sourced from it |
| Invariant changed, source chunk **hash unchanged** | **Suppress — extraction jitter, not a rule change** (fuzzy-diff guard below) |
| Chunk re-worded, predicate unchanged | Re-point `source_ref` to the new chunk; assertions untouched |
| Two chunks yield conflicting predicates on one field | Mark both `conflicted`, generate nothing, raise a `ReviewRequest` citing both chunk texts (§5.1) |

**Change → action — resource model** (`ResourceInference` re-runs on any spec *or* knowledge change; §5.2):

| Inference change | Action |
|---|---|
| Resource / operation gained | Generate its tests, subject to the confidence gate + two-signal rule (§5.2) |
| Resource / operation lost | **Quarantine, never delete** — a disappearance is as likely an extraction failure as a real change |
| Endpoint behind an inferred operation removed from spec | Retire that operation's tests + quarantine note (spec is authoritative on existence) |
| Structural claim changed, source chunk **hash unchanged** | **Suppress** — extraction jitter, same guard as invariants |
| Claim confidence drops below threshold | Field reverts to unset + `ReviewRequest`; already-generated tests quarantine rather than vanish |
| Client declares a field inference had filled | Client value wins immediately; the inferred claim is marked superseded and kept for audit |

The asymmetry is deliberate and matches the removals rule below: **gains auto-apply, losses quarantine.**

**Fuzzy-diff guard — arbitrated by chunk hash.** Spec diffs are structural and deterministic; knowledge diffs are LLM-extracted and can jitter (a re-worded doc yielding a slightly different predicate is *not* a rule change). Persisted chunks make that distinction **computable instead of guessed** — compare the `text_hash` of the chunk each invariant was extracted from:

| Chunk `text_hash` | Predicate | Reading | Action |
|---|---|---|---|
| unchanged | unchanged | nothing happened | no-op |
| **unchanged** | **changed** | **extraction jitter — source prose is byte-identical, so no rule changed** | **suppress; log, never churn the suite** |
| changed | changed | real doc edit | diff normally → auto-apply if high-confidence, else `ReviewRequest` |
| changed | unchanged | prose re-worded, rule intact | no-op on assertions; re-point `source_ref` to the new chunk |

The second row is the one that pays for chunk persistence: without a hash of the source text, an LLM re-run that phrases the same rule slightly differently is indistinguishable from the client actually changing the rule, and the suite churns on noise.

Beyond that arbitration, knowledge-change auto-apply stays deliberately conservative: only a stable-identity, high-confidence predicate change over a **changed** chunk auto-applies; anything ambiguous — a semantic shift, a new invariant with a low-confidence field match, a conflicted pair (§5.1) — becomes a `ReviewRequest` rather than churning the suite. Every such `ReviewRequest` carries the chunk's verbatim `text` for both versions, so the reviewer compares source prose, not two LLM paraphrases.

**Review gate.** Each proposed `TestChange` is scored: **auto-apply** when regeneration is high-confidence and within capability; create a **`ReviewRequest`** (routed to TaloTrace's existing HITL/review-console pipeline — never the client) when confidence is low OR the system hits a capability gap (new auth scheme, ambiguous semantic change, structural shift beyond current generators, or a fuzzy knowledge diff per above). Removals auto-retire but quarantine-and-surface rather than silently drop; "was this removal a bug?" also shows up as a normal test failure (old test → 404).

**Coverage over time.** Each `TestSuite` version computes an immutable `CoverageSnapshot` (covered / uncovered / skipped-with-reason; `generation_notes` graduates into it). Trend = diff of consecutive snapshots.

**Delivery: managed.** Tests live in and are run by TaloTrace; the client consumes reports. (Repo-sync / bot-commit into the client's own CI is a deferred add — §12.)

## 8. Data Model

All entities, final form. `id` / timestamps omitted for brevity where obvious.

**Registration & inputs**

| Entity | Key fields |
|---|---|
| **Target** | `app_id`, `project_id`, `name`, `spec_source_type` |
| **Environment** | `target_id`, `name`, `base_url`, `credential_ref` → `Credential` |
| **Credential** | `target_id`, `type` (api_key\|bearer\|oauth2_cc\|login), `label`, `secret` *(unencrypted for now — §10)* |
| **KnowledgeSource** | `target_id`, `type` (upload\|confluence\|linear), `config` *(connectors reference the platform-owned connection)* |
| **SpecWatch** | `target_id`, `mode` (github_webhook\|poll_url), `location`, `poll_interval`, `last_hash` |
| **ResourceMap** | `target_id`, `resources[]` (name, owner_field, tenant_field, `operations{crud → method, path, allowed_roles, required_scope}`). Every field additionally carries `provenance` (client_declared\|inferred_knowledge\|inferred_spec), `confidence?`, `claim_ref?` — the merged live view of declared + inferred (§5.2) |
| **AuthProfile** | `target_id`, `roles[]` (name, scopes), `credentials{role → [{id, credential_ref}]}`, `identity_slots[]` (role, needed, reason — inferred roles still awaiting client secrets). Same per-field `provenance` as `ResourceMap` |
| **SLAConfig** | `target_id`, `endpoint_ref`, `max_latency_ms`, `set_by`, `updated_at` |
| **PaginationOverride** | `target_id`, `endpoint_ref`, `style`, `traversal_params`, `metadata_fields`, `set_by`, `updated_at` |

**Artifacts (versioned, immutable)**

| Entity | Key fields |
|---|---|
| **SpecArtifact** | `target_id`, `version`, `canonical_endpoints[]`, `source_format`, `diff_from_previous`, `created_at` |
| **KnowledgeArtifact** | `target_id`, `version`, `sources[]`, `chunks[]`, `extracted_invariants[]`, `ingestion_notes[]` (chunk/source dropped + why — the graceful-degradation record) |
| **KnowledgeChunk** *(embedded)* | `id`, `source_id`, `heading_path[]`, `char_start`, `char_end`, `text` *(verbatim client prose)*, `text_hash` *(fuzzy-diff discriminator, §7)* |
| **Invariant** *(embedded)* | **stable `id`** (carried across versions by predicate/semantic match), `text` *(LLM paraphrase)*, `source_ref` (`chunk_id` + `quote_span`), `confidence`, `predicate?` (single-field or `when → then`), `status` (active\|conflicted), `conflicts_with[]` |
| **ResourceInference** | `target_id`, `version`, `spec_artifact_version`, `knowledge_artifact_version`, `claims[]`, `proposed_resources[]`, `proposed_roles[]`, `identity_slots[]`, `inference_notes[]` — the immutable proposal + evidence that the live `ResourceMap`/`AuthProfile` is merged from (§5.2) |
| **StructuralClaim** *(embedded in `ResourceInference`)* | `id`, `kind` (resource\|operation\|ownership\|role\|scope\|authz_rule), `value`, `source_ref` (`chunk_id` + `quote_span`), `confidence`, `status` (applied\|below_threshold\|conflicted\|dropped_no_spec_backing\|superseded_by_client) |
| **TestSuite** | `target_id`, `name`, `version`, `spec_artifact_version`, `knowledge_artifact_version`, `resource_inference_version?`, `test_cases[]`, `scenarios[]`, `generation_notes[]` |
| **CoverageSnapshot** | `suite_version`, `covered[]`, `uncovered[]`, `skipped[]` (ref, reason), `summary_pct` |

**Tests**

| Entity | Key fields |
|---|---|
| **TestCase** | `suite_id`, `name`, `category` (functional\|business_logic\|schema\|negative\|auth\|pagination\|sla), `endpoint_ref`, `request_template` (may carry `identity_ref`), `assertions[]` (type, field w/ array-wildcard paths, expected, rationale, **`source`** — spec element or `invariant_id`+`knowledge_version`, the machine link for change-mapping; `conditional` assertions carry `when`/`then` predicates over 2+ fields), `needs_seed_value` |
| **TestScenario** | `suite_id`, `resource`, `name`, `identity_ref`, `steps[]` |
| **ScenarioStep** *(embedded)* | `id`, `op`, `depends_on`, `request` (method/path/body), `capture{var → body/header/status source}`, `assertions[]` |
| **SeedValue** | `test_case_id`, `param_name`, `value`, `provided_by`, `provided_at` |

**Execution & findings**

| Entity | Key fields |
|---|---|
| **SuiteRun** | `suite_id`, `environment_id`, `triggered_by`, `started_at`, `finished_at`, `status` |
| **RunResult** | `suite_run_id`, `test_case_id`, `http_status`, `latency_ms`, `assertion_results[]`, `evidence`, `outcome` (pass\|fail\|error) |
| **ScenarioRun** | `suite_run_id`, `scenario_id`, `environment_id`, `outcome`, `step_results[]` |
| **StepResult** *(embedded)* | `step_id`, `op`, `http_status`, `latency_ms`, `captured{}`, `assertion_results[]`, `outcome` (pass\|fail\|error\|skipped) |
| **Finding** | `suite_run_id`, `test_ref` (case or scenario), `category`, `severity`, `summary`, `evidence_ref`, `rationale` |

**Maintenance**

| Entity | Key fields |
|---|---|
| **ChangeSet** | `target_id`, `input` (spec\|knowledge\|resource_inference), `from_version`, `to_version`, `changes[]` |
| **Change** *(embedded)* | `type` (spec: endpoint_added\|endpoint_removed\|signature_change\|field_added\|field_removed\|field_changed\|deprecated; knowledge: invariant_added\|invariant_removed\|invariant_changed; resource_inference: resource_added\|resource_lost\|operation_added\|operation_lost\|authz_changed\|confidence_dropped), `target_ref` (endpoint/field, `invariant_id`, or resource/operation ref), `detail` |
| **MaintenanceRun** | `target_id`, `change_set_id`, `produced_suite_version`, `test_changes[]`, `status` |
| **TestChange** *(embedded)* | `action` (create\|regenerate\|retire\|add_assertion), `test_ref`, `confidence`, `disposition` (auto_applied\|pending_review\|quarantined), `reason` |
| **ReviewRequest** | `maintenance_run_id`, `test_change_ref`, `trigger` (low_confidence\|capability_gap\|conflicting_claims), `subject_ref?` (invariant, or `StructuralClaim` under review), `status` |

## 9. Worked Example — "Modun Orders API"

```json
// Target + Environments + Credentials
{ "id": "tgt_01h9x", "app_id": "app_modun", "project_id": "proj_bookings",
  "name": "Modun Orders API", "spec_source_type": "openapi3" }
[ { "id": "env_01a1", "target_id": "tgt_01h9x", "name": "staging",
    "base_url": "https://staging.modun.example.com/api", "credential_ref": "cred_01m1" },
  { "id": "env_01a2", "target_id": "tgt_01h9x", "name": "production",
    "base_url": "https://api.modun.example.com", "credential_ref": "cred_01m2" } ]
[ { "id": "cred_01m1", "type": "bearer", "label": "staging token", "secret": "eyJ..." },
  { "id": "cred_01m3", "type": "bearer", "label": "staging userA", "secret": "eyJ...A" },
  { "id": "cred_01m4", "type": "bearer", "label": "staging userB", "secret": "eyJ...B" } ]

// KnowledgeArtifact (from an uploaded PDF + Confluence space)
// chunks are persisted evidence — invariants cite them by id, not by a "#3.2" string
{ "id": "know_01d1", "target_id": "tgt_01h9x", "version": 1,
  "chunks": [   // excerpt — chk_01/chk_02 shown; chk_03…chk_09 elided, cited later by ResourceInference
    { "id": "chk_01", "source_id": "ks_pdf1", "heading_path": ["Orders", "3.2 Totals"],
      "char_start": 4120, "char_end": 4390, "text_hash": "sha256:9f3a…",
      "text": "Every order carries a total in minor units. An order's total must always be positive; a zero or negative total indicates a pricing fault and must never be persisted." },
    { "id": "chk_02", "source_id": "ks_pdf1", "heading_path": ["Orders", "4.1 Payment"],
      "char_start": 5880, "char_end": 6040, "text_hash": "sha256:c17b…",
      "text": "A completed order must have a payment record. Orders may not transition to completed until payment settles." } ],
  "extracted_invariants": [
    { "id": "inv_total_positive", "text": "An order's total must always be positive.",
      "source_ref": { "chunk_id": "chk_01", "quote_span": [72, 116] }, "confidence": 0.91,
      "status": "active",
      "predicate": { "field": "total", "op": "greater_than", "value": 0 } },
    { "id": "inv_completed_has_payment", "text": "A completed order must have a payment record.",
      "source_ref": { "chunk_id": "chk_02", "quote_span": [0, 45] }, "confidence": 0.88,
      "status": "active",
      "predicate": { "when": { "field": "status", "op": "equals", "value": "completed" },
                     "then": { "field": "payment_id", "op": "not_null" } } } ],
  "ingestion_notes": [ { "source_id": "ks_conf1", "chunk_id": "chk_09",
                         "reason": "LLM extraction failed after retries — chunk dropped, artifact retained" } ] }

// SpecArtifact (excerpt — nested response schema)
{ "id": "spec_01c1", "target_id": "tgt_01h9x", "version": 1, "source_format": "openapi3",
  "canonical_endpoints": [
    { "method": "GET", "path": "/orders/{id}", "response_schema": { "200": {
        "id": "string", "status": "string", "total": "number",
        "line_items": { "type": "array", "items": {
          "id": "string", "quantity": "number", "weight": "number" } } } } } ],
  "diff_from_previous": null }

// ResourceInference — client declared NO ResourceMap; inferred from the chunks above + spec (§5.2)
{ "id": "rinf_01g1", "target_id": "tgt_01h9x", "version": 1,
  "spec_artifact_version": 1, "knowledge_artifact_version": 1,
  "claims": [
    { "id": "clm_01", "kind": "resource", "value": "order",
      "source_ref": { "chunk_id": "chk_01", "quote_span": [0, 45] }, "confidence": 0.94, "status": "applied" },
    { "id": "clm_02", "kind": "ownership", "value": { "resource": "order", "owner_field": "userId" },
      "source_ref": { "chunk_id": "chk_03", "quote_span": [12, 58] }, "confidence": 0.87, "status": "applied" },
    { "id": "clm_03", "kind": "authz_rule", "value": { "operation": "delete", "allowed_roles": ["admin"] },
      "source_ref": { "chunk_id": "chk_07", "quote_span": [0, 39] },   // "Only admins may cancel an order."
      "confidence": 0.91, "status": "applied" },
    { "id": "clm_04", "kind": "operation", "value": { "resource": "order", "op": "update" },
      "source_ref": { "chunk_id": "chk_05", "quote_span": [0, 61] }, "confidence": 0.42,
      "status": "below_threshold" } ],                                 // → ReviewRequest, update left unset
  "identity_slots": [ { "role": "admin", "needed": 1, "reason": "delete is admin-only" },
                      { "role": "user", "needed": 2, "reason": "2nd same-role identity enables BOLA replay" } ],
  "inference_notes": [
    { "reason": "operation dropped — PATCH /orders/{id} named in prose, absent from spec v1",
      "claim_ref": "clm_09", "status": "dropped_no_spec_backing" } ] }

// ResourceMap — live merged view. No client declaration here, so every field is inferred.
// `update` is absent: clm_04 fell below threshold. Read/list/delete still generate normally.
{ "target_id": "tgt_01h9x",
  "resources": [ { "name": "order",
    "owner_field": { "value": "userId", "provenance": "inferred_knowledge", "confidence": 0.87, "claim_ref": "clm_02" },
    "operations": {
      "read":   { "method": "GET",    "path": "/orders/{id}", "allowed_roles": ["user","admin"],
                  "provenance": "inferred_knowledge", "confidence": 0.9 },
      "list":   { "method": "GET",    "path": "/orders",      "allowed_roles": ["user","admin"],
                  "provenance": "inferred_knowledge", "confidence": 0.9 },
      "delete": { "method": "DELETE", "path": "/orders/{id}", "allowed_roles": ["admin"],
                  "provenance": "inferred_knowledge", "confidence": 0.91, "claim_ref": "clm_03" } } } ] }
// two-signal check on `delete`: spec has DELETE /orders/{id} (signal 1) AND chk_07 describes cancelling (signal 2) → auto-applied
// `create` was never claimed in prose → no create → no create_via → NO delete test generated yet (§5.2 self-exclusion)

// AuthProfile (roles + credentials; resource rules live in ResourceMap §4.1)
{ "id": "authp_01k0", "target_id": "tgt_01h9x",
  "roles": [ { "name": "user", "scopes": ["read:own","write:own"] },
             { "name": "guest", "scopes": ["read:public"] } ],
  "credentials": { "user": [ { "id": "userA", "credential_ref": "cred_01m3" },
                             { "id": "userB", "credential_ref": "cred_01m4" } ] } }

// TestCase — schema (nested wildcard assertions)
{ "id": "tc_02a1", "suite_id": "suite_01e1", "category": "schema",
  "name": "GET /orders/{id} — response matches schema incl. line_items",
  "endpoint_ref": "GET /orders/{id}",
  "assertions": [
    { "type": "field_type", "field": "line_items[].quantity", "expected": "number",
      "source": { "kind": "spec", "endpoint_ref": "GET /orders/{id}", "field": "line_items[].quantity" }, "rationale": "spec" },
    { "type": "value_positive", "field": "total",
      "source": { "kind": "knowledge", "invariant_id": "inv_total_positive", "knowledge_version": 1 },
      "rationale": "total must be positive" } ] }

// TestCase — business_logic (cross-field conditional)
{ "id": "tc_02b1", "suite_id": "suite_01e1", "category": "business_logic",
  "name": "GET /orders/{id} — completed order has a payment_id",
  "endpoint_ref": "GET /orders/{id}",
  "assertions": [
    { "type": "conditional",
      "when": { "field": "status", "op": "equals", "value": "completed" },
      "then": { "field": "payment_id", "op": "not_null" },
      "source": { "kind": "knowledge", "invariant_id": "inv_completed_has_payment", "knowledge_version": 1 },
      "rationale": "a completed order must have a payment record" } ] }
// evaluation: status != completed → not-applicable (N/A, not a fail); status == completed → payment_id must be non-null
// change-mapping: this assertion's source.invariant_id links it to the invariant — edit/remove the rule and only this assertion is affected

// TestCase — auth (BOLA)
{ "id": "tc_02e1", "suite_id": "suite_01e1", "category": "auth",
  "name": "GET /orders/{id} — userB cannot read userA's order", "endpoint_ref": "GET /orders/{id}",
  "request_template": { "path_params": { "id": "ord_8842" }, "identity_ref": "userB" },
  "assertions": [ { "type": "status_code", "expected": "403|404",
    "rationale": "BOLA: order owned by userA, requested as same-role userB" } ] }

// TestScenario — CRUD lifecycle (create threads order_id; DAG via depends_on)
{ "id": "scn_03a1", "suite_id": "suite_01e1", "resource": "order", "identity_ref": "userA",
  "name": "Order lifecycle — create → read → list → update → delete",
  "steps": [
    { "id": "s1", "op": "create", "request": { "method": "POST", "path": "/orders", "body": "<schema+knowledge synth, run-unique external_id>" },
      "capture": { "order_id": "$.id" }, "assertions": [ { "status_code": "201" } ] },
    { "id": "s2", "op": "read", "depends_on": "s1", "request": { "method": "GET", "path": "/orders/{order_id}" },
      "assertions": [ { "field_equals": "total", "from": "s1.response.total", "rationale": "create is readable" } ] },
    { "id": "s4", "op": "update", "depends_on": "s1", "request": { "method": "PUT", "path": "/orders/{order_id}", "body": { "status": "paid" } },
      "assertions": [ { "status_code": "200" } ] },
    { "id": "s5", "op": "read", "depends_on": "s4", "request": { "method": "GET", "path": "/orders/{order_id}" },
      "assertions": [ { "field_equals": "status", "expected": "paid", "rationale": "update persisted" } ] },
    { "id": "s6", "op": "delete", "depends_on": "s1", "request": { "method": "DELETE", "path": "/orders/{order_id}" },
      "assertions": [ { "status_code": "204|200" } ] },
    { "id": "s7", "op": "read", "depends_on": "s6", "request": { "method": "GET", "path": "/orders/{order_id}" },
      "assertions": [ { "status_code": "404", "rationale": "deleted resource is gone" } ] } ] }

// ScenarioRun — s4 update failed; only its dependent s5 skipped, rest ran
{ "id": "srun_03b1", "scenario_id": "scn_03a1", "outcome": "fail",
  "step_results": [
    { "step_id": "s1", "http_status": 201, "captured": { "order_id": "ord_9001" }, "outcome": "pass" },
    { "step_id": "s2", "http_status": 200, "outcome": "pass" },
    { "step_id": "s4", "http_status": 500, "outcome": "error" },
    { "step_id": "s5", "outcome": "skipped", "reason": "depends_on s4 (error)" },
    { "step_id": "s6", "http_status": 204, "outcome": "pass" },
    { "step_id": "s7", "http_status": 404, "outcome": "pass" } ] }

// Finding — rationale resolves the full audit chain (§5.1), ending in the client's own words
{ "id": "find_01j1", "suite_run_id": "run_01h1", "test_ref": "tc_02a1", "category": "schema", "severity": "high",
  "summary": "GET /orders/ord_8842 returned total=-5, expected positive.",
  "rationale": "invariant inv_total_positive (knowledge v1)",
  "evidence_ref": { "chunk_id": "chk_01", "cited_text": "An order's total must always be positive",
                    "location": "modun-rules.pdf § Orders › 3.2 Totals" } }

// Maintenance A — SPEC change: client ships spec v2 dropping field `weight`; auto-detected, regenerated
{ "id": "cs_04a1", "target_id": "tgt_01h9x", "input": "spec", "from_version": 1, "to_version": 2,
  "changes": [ { "type": "field_removed", "target_ref": "GET /orders/{id}.line_items[].weight" } ] }
{ "id": "tchg_04b1", "action": "retire", "test_ref": "tc_02a1.assertion[line_items[].weight]",
  "confidence": 0.95, "disposition": "auto_applied", "reason": "field removed from spec v2" }

// Maintenance B — KNOWLEDGE change: Confluence edited — refunded orders must also carry payment_id.
// Invariant inv_completed_has_payment (stable id) matched across versions; its predicate widened.
// Chunk hash CHANGED (c17b… → 4e02…) → real doc edit, not extraction jitter → diff is trusted.
{ "id": "cs_04c1", "target_id": "tgt_01h9x", "input": "knowledge", "from_version": 1, "to_version": 2,
  "changes": [ { "type": "invariant_changed", "target_ref": "inv_completed_has_payment",
                 "chunk_ref": { "chunk_id": "chk_02", "hash_from": "sha256:c17b…", "hash_to": "sha256:4e02…" },
                 "detail": "when-clause now status in [completed, refunded]" } ] }
// affected assertion found via source.invariant_id (not a text search); confidence high, predicate structured → auto-apply
{ "id": "tchg_04d1", "action": "regenerate", "test_ref": "tc_02b1", "confidence": 0.9,
  "disposition": "auto_applied", "reason": "invariant inv_completed_has_payment predicate widened (knowledge v2)" }

// Maintenance C — SUPPRESSED jitter: re-ingest produced a differently-worded predicate for inv_total_positive
// ("must be greater than zero" vs "must be positive"), but chunk chk_01 hash is BYTE-IDENTICAL.
// Source prose unchanged ⇒ no rule changed ⇒ no ChangeSet entry, no TestChange, suite untouched.
{ "suppressed": { "invariant_id": "inv_total_positive", "chunk_id": "chk_01",
                  "text_hash": "sha256:9f3a…", "note": "extraction jitter — chunk unchanged, predicate diff ignored" } }
// without the persisted chunk hash this is indistinguishable from a real rule edit, and tc_02a1 churns for nothing
```

## 10. Security Note — Credential Storage

The `Credential` entity stores client API secrets **unencrypted in the app DB** — a known liability, accepted as a time-boxed interim because no secret vault exists yet.

- **Why stored:** executing a test requires presenting valid auth to the client's API, and unattended maintenance runs have no human to inject a secret. Storage is unavoidable.
- **Distinct from user-auth:** this is *client API* secret storage; the existing user-auth-to-our-system mechanism does not cover it.
- **Required hardening before real-client/production use:** at minimum encryption-at-rest with an app-managed key (env/KMS); ideally a proper secret store when the platform provides one. Until then, treat secrets as sensitive — restrict access, keep them out of logs.

## 11. Design Decisions

**Architecture & data**

| Decision | Choice | Rationale |
|---|---|---|
| Structure | Single service, staged pipeline, versioned immutable artifacts | "Fast and minimal" execution infra; versioning is the maintenance backbone, built in from day one |
| Multi-tenancy | `app_id`/`project_id` opaque FKs on `Target` | Matches platform partitioning; tenancy model itself out of scope |
| LLM retry policy | Defer to codebase's shared LLM client | Not a decision to invent in isolation |
| Engine testing | Fixtures + mock HTTP server for deterministic parts; no generation-quality eval harness (yet) | Parsing/execution is conventionally testable; a golden-reference LLM eval is premature |

**Inputs & registration**

| Decision | Choice | Rationale |
|---|---|---|
| Spec formats | OpenAPI 3.x, Swagger 2.0, Postman, Insomnia | Covers clients with only an internal collection |
| Auth support | API key, Bearer, OAuth2 client-credentials | Realistic majority of client APIs |
| Environments | Multiple per target from the start | One suite spans dev/staging/prod; cheap vs. retrofitting |
| Knowledge input | PDF/DOCX + Confluence/Linear; auth/fetch is platform infra | How clients hold product knowledge; needs extraction, not structured entry |
| Client secret storage | `Credential` entity, unencrypted interim; encryption deferred (§10) | No vault exists; storage required for unattended runs. Deliberate, time-boxed |
| Resource model | Shared `ResourceMap`; `AuthProfile` = roles + credentials only | One client declaration serves both auth and CRUD generation |
| Resource-model source | **Inferred from the knowledge base**, spec used as validator (§5.2) | The spec cannot express authz or ownership at all; the client already wrote both in prose. Transcribing a resource model by hand is the largest onboarding cost the product can delete |
| Inference gate | Auto-apply above a confidence threshold; below → field unset + `ReviewRequest` to a TaloTrace reviewer | Keeps "client closes their eyes"; an unset field degrades one operation, not the resource |
| Mutating-operation safety | Two signals required: spec-backed endpoint **and** prose that explicitly describes the operation | Path symmetry alone is not evidence of intent; mirrors the pagination two-signal rule. Backed by `create_via` — an auto-applied delete only removes rows the suite created |
| Declared vs inferred | One live entity with per-field `provenance`; precedence `client_declared` > `inferred_knowledge` > `inferred_spec` | Generation reads one view with no merge logic; client input is never overwritten, only supersedes |
| Inference persistence | Immutable versioned `ResourceInference` beside the live merged map | The proposal must be diffable and reviewable; a live-only map would let a re-run silently rewrite the resource model |
| AuthProfile inference | Roles + scopes inferred; credentials never. Gaps surface as `identity_slots[]` | An inferred `allowed_roles` is meaningless unless the role exists; secrets are the one thing only the client holds |

**Generation**

| Decision                   | Choice                                                                                                                                                  | Rationale                                                                                                                                                                                                                               |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Grounding                  | Knowledge-informed assertions + LLM-assisted body synthesis                                                                                             | USP Pillar 1 — the reason the product beats a free fuzzer                                                                                                                                                                               |
| Cross-field business logic | `conditional` (`when → then`) assertions over 2+ fields; structured predicate extracted by LLM, evaluated by deterministic code; antecedent-false → N/A | The "business-logic, not just schema shape" differentiator; deterministic eval keeps runs fast/reproducible; N/A avoids false fails when the rule doesn't apply                                                                         |
| Per-capability tagging     | Separate `category`-tagged cases                                                                                                                        | Enables per-capability reporting + coverage attribution                                                                                                                                                                                 |
| Schema depth               | Recursive walk, array-wildcard per-item assertions                                                                                                      | Nested structures need per-item, per-field checks                                                                                                                                                                                       |
| Fuzzing                    | Property-based, schema-driven; no per-parameter cap yet                                                                                                 | Scales with the spec; cap only on evidence of blow-up                                                                                                                                                                                   |
| Auth model                 | `AuthProfile` × `ResourceMap` → authz truth table                                                                                                       | Computes allow/deny per (identity, op); makes wrong-scope + BFLA first-class                                                                                                                                                            |
| BOLA                       | Same-role replay via `owner_field` (`create_via` auto-seed once mutating is live)                                                                       | Strongest object-level signal                                                                                                                                                                                                           |
| Pagination detection       | LLM-assisted, two-signal rule; ambiguous → skip + `PaginationOverride`                                                                                  | Naming varies too widely for a fixed list; never guess                                                                                                                                                                                  |
| SLA                        | Single-request latency vs. client threshold; no default                                                                                                 | "Contained scope, real value fast"; load testing is a separate capability                                                                                                                                                               |
| Undeclared resources       | Knowledge-inferred and auto-applied under the §5.2 gate, not client-confirmed                                                                           | Client confirmation would re-introduce the manual step the product exists to remove; the two-signal rule + `create_via` carry the safety instead                                                                                        |
| Inference failure          | Graceful degradation — failed chunk drops its claims, unbacked operation drops itself; `inference_notes[]`                                              | Same class as knowledge ingestion: a partial resource model is useful, a blocked one is not                                                                                                                                             |
| Generation failure         | All-or-nothing per suite                                                                                                                                | A coverage contract must not be silently incomplete                                                                                                                                                                                     |
| Spec parse failure         | All-or-nothing per spec file                                                                                                                            | Same principle at the spec layer                                                                                                                                                                                                        |
| Knowledge chunking         | Section/heading boundary, `heading_path[]` preserved; not fixed-token, not sliding-window                                                               | A rule and its qualifiers live in one section — a token window splits a `when → then` across two LLM calls and loses half the predicate                                                                                                 |
| Chunk persistence          | Chunks stored on the `KnowledgeArtifact` with verbatim `text`, char offsets, and `text_hash`                                                            | Chunks are the *evidence* a rule was written down: enables source-quoted findings, reviewer comparison against real prose, and hash-based jitter suppression (§7). A `"file.pdf#3.2"` string does none of these and rots on renumbering |
| Invariant citation         | Structured `source_ref` = `chunk_id` + `quote_span`, not a heading string                                                                               | Machine-resolvable to the exact sentence, version-pinned and reproducible                                                                                                                                                               |
| Conflicting invariants     | Both marked `conflicted`, neither generates an assertion, `ReviewRequest` cites both chunks                                                             | Contradicting client docs are a finding about the docs; silently picking one manufactures a false assertion                                                                                                                             |
| Knowledge failure          | Graceful degradation (source + chunk), each drop recorded in `ingestion_notes[]`                                                                        | Best-effort enrichment; partial is useful — but a thinner artifact must be visible, not silent                                                                                                                                          |

**Execution**

| Decision | Choice | Rationale |
|---|---|---|
| Chain model | First-class `TestScenario`, DAG steps | CRUD needs unit-level semantics single-call cases can't give |
| Chaining | Hybrid: named `capture` into requests; step-response addressing in assertions | Explicit vars for real data flow; ad-hoc for one-off checks |
| Scenario failure | Skip transitive dependents, run other branches | More signal per run than stop-on-first |
| Single-call failure | Continue, mark `error`; one network retry | Independent cases; partial report beats aborting |
| Request dedup | Dedupe identical functional/schema/sla requests | Avoids redundant load; transparent to reporting |
| Body re-run safety | Run-unique injection for unique/format fields | Prevents re-run 409s |
| **Mutating env safety** | **Client-owned — no guard** | Client's explicit choice. **Risk:** a misconfigured run can create/delete real prod data |
| **Teardown** | **Deferred — self-clean via `delete` step only** | Not a spec requirement. **Risk:** partial/failed runs leak real data |
| Report/ticket | Emit a `Finding` contract; pipeline is an external boundary | Buildable without pipeline internals |
| `Finding` severity | Not scoped yet | No requirement; calibrate on real findings |

**Maintenance**

| Decision | Choice | Rationale |
|---|---|---|
| Trigger | Spec-diff **and knowledge-diff**, auto-fetched (GitHub webhook / poll URL; platform knowledge refresh) | Both inputs a suite is generated from can change; watching only the spec would leave business-logic tests to rot |
| Assertion traceability | Structured `source` ref on every assertion (spec element or `invariant_id`+version) + stable invariant IDs | "Affected-only" mapping needs a real link, not a text search; stable IDs let a rule be tracked release to release |
| Knowledge-diff conservatism | Fuzzy diffs bias to `ReviewRequest`; only stable-identity, high-confidence predicate changes over a **changed chunk** auto-apply | Spec diffs are deterministic, knowledge is LLM-extracted and jitters — extraction noise must not churn the client's suite |
| Jitter vs. real edit | Arbitrated by chunk `text_hash`: unchanged hash + changed predicate ⇒ suppress | Makes the distinction computable instead of guessed — the single highest-leverage payoff of persisting chunks |
| Inferred-model drift | Re-infer on either input change; gains auto-apply, losses quarantine; same hash-jitter suppression | A vanished resource is as likely an extraction failure as a real removal — matching the removals rule below |
| Review gate | TaloTrace-internal; auto-apply when confident+capable, else `ReviewRequest` | Client hands-off *and* an intended-vs-bug check; human only when unsure/stuck |
| Affected-test mapping | Hybrid — field-level for field edits, endpoint-level for structural | True "affected only"; minimizes churn, protecting the trust budget |
| Removals | Auto-retire + quarantine, never silent-delete | A removal may be an unintended break |
| Coverage-over-time | Immutable `CoverageSnapshot` per version; trend by diff | Fits the versioned-artifact pattern |
| Delivery | Managed — TaloTrace runs + maintains | The promise is "we run it, you consume results" |

## 12. Deferred & Open Items

| Item                                 | Status                            | Note                                                                                                                                                                                                                                                                         |
| ------------------------------------ | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Credential encryption at rest        | Before real-client/production use | Unencrypted interim; §10                                                                                                                                                                                                                                                     |
| Teardown / created-resource cleanup  | Deferred                          | Self-clean via `delete`; partial runs leak. Candidate: created-resource ledger + best-effort sweep + `orphaned_resources` report                                                                                                                                             |
| Mutating-run environment guard       | Deferred                          | No `allow_mutating` flag; client points at the right env                                                                                                                                                                                                                     |
| Expired-credentials check            | Deferred (non-vital)              | Candidates: token-gen endpoint w/ TTL param, or static expired token                                                                                                                                                                                                         |
| Per-parameter fuzzing cap            | Deferred (no evidence)            | Generate all; cap only if volume proves a problem                                                                                                                                                                                                                            |
| Cross-category `Finding` severity    | Deferred (no requirement)         | Calibrate on real findings                                                                                                                                                                                                                                                   |
| Oversized / heading-less chunks      | Deferred                          | Section-chunking assumes headed prose. A 40-page unheaded doc, or one section longer than the LLM window, has no fallback yet. Candidates: recursive split on paragraph boundaries with overlap, or a heading-inference pass. Record as an `ingestion_notes` skip until then |
| Inference confidence threshold value | Open — calibrate on real docs     | §5.2's gate is designed; the numeric threshold is not set. Pick it from the first real knowledge bases, not from a guess. Interim: start strict, loosen on measured false-proposal rate                                                                                      |
| Spec-only inference fallback         | Deferred                          | A client with a spec but no knowledge base gets no inferred `ResourceMap` today (`inferred_spec` provenance exists but nothing emits it). Candidate: path-symmetry proposal for the read slice only, never mutating                                                          |
| Cross-source invariant dedup         | Deferred                          | Same rule stated in both a PDF and Confluence yields two invariant ids over one predicate. Conflict detection (§5.1) catches contradictions, not duplicates — duplicates only cost a redundant assertion                                                                     |
| Live-API re-validation               | eDeferred                         | Automatic drift safety net for spec-less clients / "spec lies"; complements spec-diff                                                                                                                                                                                        |
| Repo-sync / bot-commit delivery      | Deferred                          | Dependabot-style sync into the client's own CI; managed is the default                                                                                                                                                                                                       |
| Continuous monitoring vs. reference  | Deferred                          | Most thorough/expensive detection; only if spec-diff + re-validation prove insufficient                                                                                                                                                                                      |

## Appendix A — Build Sequencing (Tier Map)

Capabilities map to the spec's tiers for build order. The design above is one system; this is only *when* each piece ships.

| Tier | Capabilities |
|---|---|
| **Tier 0 — Foundation** | Registration (Target/Environment/Credential/KnowledgeSource), spec normalization, knowledge ingestion **incl. chunk persistence + structured `source_ref`** (§5.1 — cheap now, unbuildable retroactively for artifacts already ingested), single-call `functional` generation, manual execution, reporting boundary. GET/HEAD only. |
| **Tier 1 — v1, client-usable** | `schema`, `negative`, `auth` matrix (`AuthProfile` + BOLA/BFLA read-slice), `pagination`, `sla`, per-category tagging, request dedup. **Resource inference, read slice** (§5.2 passes 1–3, `AuthProfile` roles/scopes + `identity_slots`) — the read operations are all Tier 1 can use, and it removes the hand-written `AuthProfile` from onboarding. Still GET/HEAD. |
| **Tier 2a — Full CRUD lifecycle** | `ResourceMap`, `TestScenario`, mutating methods, chaining/DAG, body synthesis, `schema`+`negative` across all CRUD ops, `create_via` + BOLA/BFLA on writes. **Resource inference, mutating slice** — two-signal rule + `create_via` safety net, which only mean anything once mutating ops exist. |
| **Tier 2b — Maintenance engine (USP Pillar 2)** | `SpecWatch` auto-detection, `ChangeSet` diff, **chunk-hash jitter arbitration** (§7), **inferred-model re-inference + drift handling**, affected-only regeneration, review gate, `CoverageSnapshot` trend, managed delivery. |
| **Later tiers (not designed here)** | Tier 3: OWASP API Top 10, idempotency, rate-limiting, CDC. Tier 4: GraphQL, gRPC. Tier 5: trust/enterprise (published FP-rate, HITL metrics, data-masking). |
