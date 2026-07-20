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

# Tier 0 questions

1. **Change-detection mechanism** — scheduled re-validation, spec-diff, or continuous monitoring? (owner: Khoa Hung, due Jul 16)
2. **Can API testing reuse the existing app-graph** (`app_graph_reader.py`), or does it need its own representation?
3. **Does `services/knowledge_base/` already hold product knowledge**, or is it not built yet?
4. **What's the real quality gap between baseline (spec + product knowledge) vs. enhanced (+ codebase) generation** — worth pushing clients to connect GitHub?
5. **Does generation pass the gate** — is it genuinely smarter than free Postman/Schemathesis?

**Not yet asked, worth adding:** 6. How do we trace a field-level spec change back to the right endpoint-level tests? (needed for Tier 2's "affected tests only")