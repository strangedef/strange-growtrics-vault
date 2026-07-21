Here's the consolidated requirement scope across Tier 0 through Tier 2, pulled from the capability spec:

## Tier 0 — Foundation

**Timing:** Spike week, Jul 14–16

**Scope:**

- **Target registration** — base URL + auth + environment setup (reuses the existing credentials-vault)
- **Generation v0** — produce test cases from OpenAPI spec **+ connected product knowledge** (not mechanical schema-only fuzzing)
- **Basic functional testing** — happy-path checks (right status codes, right fields)
- **Reporting plumbing** — results flow into the existing report/ticket pipeline (no new reporting system)

**Gate to pass:** a human reviewer must judge the generated output as genuinely smarter than a bare Postman/Schemathesis run — if not, iterate before moving to Tier 1.

**Open item to resolve during this tier:** decide the release-over-release change-detection mechanism (scheduled re-validation vs. spec-diff vs. continuous monitoring) — this decision unlocks Tier 2's confidence/effort estimates.

---

## Tier 1 — v1, Client-Usable

**Timing:** Prototype Jul 20–Aug 1, productize Aug 3–14 (mid-Aug commitment)

**Scope:**

- **Schema/contract validation** — responses validated against OpenAPI/GraphQL schema
- **Negative/edge-case fuzzing** — malformed, boundary, wrong-type inputs to confirm safe failure
- **Auth/authz matrix incl. BOLA/BFLA** — missing/invalid/expired/wrong-scope checks, plus multi-tenant object-level authorization replay
- **Pagination checks** — no duplicates/gaps, correct next/previous metadata
- **Per-endpoint SLA (client-configurable)** — client sets latency/error-rate thresholds per endpoint in the interface; report names which endpoint breached its SLA and by how much
- **Basic reporting** — pass/fail per endpoint

**Also folded in during this window (per the week-by-week roadmap):**

- Codebase-aware generation, when a client has connected GitHub (opt-in enhanced tier)
- Productized target registration UI, suite composition, credit pricing/docs

**Explicitly excluded from Tier 1:** the maintenance/auto-regeneration engine — deferred to Tier 2.

---

## Tier 2 — Maintenance Engine (USP Pillar 2)

**Timing:** Aug 18–Sep 5 — pulled forward, starts immediately after Tier 1 ships (not "proposed Sep")

**Scope:**

- **Release-over-release change detection** — compare client's declared spec (or live API behavior, if no spec exists) between releases; catch new/removed fields, changed types, new/deprecated endpoints
- **Automatic test regeneration/update for affected tests only** — not a full rebuild, not silence
- **Regression-safe evolution** — retire/flag tests for endpoints that no longer exist, protecting the false-positive-rate trust threshold
- **Human-review gate on auto-updates** — auto-generated test changes route through the same HITL reviewer pipeline (epic HGSQA-778) before being trusted, since an "API change" might be an unintentional bug
- **Coverage attribution over time** — report which parts of the API are covered vs. not, and how that's changed release over release

**Dependency:** HGSQA-968 (multi-target projects — treats an API as just another target type)

**Why this tier isn't optional/deferrable:** this is the second half of the actual USP (§0) — without it, TaloTrace's API offering is "just another Postman." Its RICE scores look mid-table only due to low confidence/high build-uncertainty, not because it matters less; Impact is rated at maximum on both key items.

---

### Quick summary table

| Tier | Core deliverable                              | Answers                                          |
| ---- | --------------------------------------------- | ------------------------------------------------ |
| 0    | Prove generation quality beats a generic tool | Can we generate smarter tests than free tools?   |
| 1    | Ship a sellable, execution-complete v1        | Does the API actually work correctly and safely? |
| 2    | Make the test suite self-maintaining          | Does the client ever have to touch this again?   |
#### Note Tier 0
At the end of Tier 0, someone has to look at the generated tests and honestly judge: **is this actually better than what a free tool like Postman or Schemathesis would produce?**

If yes → move on to Tier 1. If no → stop and improve the generation quality first, don't proceed yet.

It's a checkpoint to make sure the core differentiator (smarter test generation) is real before building the rest of the product on top of it.


### Does testing BOLA/BFLA per enpoint cover wrong-scope check

Yes — for what wrong-scope was actually trying to protect against, comprehensive BOLA/BFLA per-endpoint testing covers it, **with one condition**: the client's role/permission model has to be captured and fed into the system first (the onboarding gap flagged earlier).

To be precise about _why_ this holds:

**What wrong-scope was checking:** "is this credential being used for something it wasn't issued permission to do?" — mechanically verified by reading the token's declared scope.

**What BOLA/BFLA checks instead:** the same underlying question — "should this credential be allowed to do this?" — just verified by replaying the request as different roles/accounts and checking the response, rather than by reading a scope field off the token.

- **BFLA** (vertical + horizontal) → catches "wrong function" cases, which is what wrong-scope was mostly protecting against for OAuth2 clients (a read-only-scoped token trying to write, a sales-scoped token trying to hit an admin endpoint)
- **BOLA** → catches "wrong data owner" cases, which formal scope systems don't even really cover well in the first place (scope says _what actions_ a token can do, not _whose data_ it can touch)

### OAuth2 access token looks
the format is standardized (OAuth2 spec), but the mapping of scope→endpoint is still client business logic that has to come from onboarding