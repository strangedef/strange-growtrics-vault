Here's a consolidated draft of the API Testing requirement, pulled together from the capability spec and onboarding doc:

## API Testing — Requirements Draft

**Status:** DRAFT · Owner: Khoa Hung (build) · Reviewer: Praveen · Created: 2026-07-13

### Positioning (the non-negotiable framing)

Running API tests is **not** the product — Postman/Karate/ReadyAPI/k6 already do that for free. If TaloTrace's offering is just "execute requests, check status codes," it's a worse paid clone of a free tool. The USP rests on two pillars:

1. **Comprehensive test generation grounded in app + codebase + product knowledge** (not mechanical schema-fuzzing)
    - _App knowledge_ — reuse TaloTrace's existing UI-exploration understanding of real user flows/business logic
    - _Codebase knowledge_ — when a client opts into "Connect GitHub," read actual route/handler code for validation rules and edge cases a spec won't show
    - _Product knowledge_ — onboarding docs/business invariants (positive prices, future event dates, etc.) drive business-logic assertions, not just schema-shape checks
2. **Continuous maintenance** — "we generate your tests, then keep them current as your app changes — you never touch this again," mirroring the self-healing philosophy of TaloTrace's UI-testing product

**Explicitly ruled out:** capturing API tests from TaloTrace's own UI-driven traffic. Testing is driven directly through API calls.

**Still open (decide during Jul 14–16 spike):** the exact change-detection mechanism — scheduled re-validation vs. spec-diff-triggered regeneration vs. continuous monitoring.

### Core capability taxonomy (MVP → best-in-class)

Covers: functional/happy-path, schema/contract validation, consumer-driven contract testing, negative/edge-case fuzzing, auth/authz (incl. BOLA/BFLA), OWASP API Top 10 security, performance/SLA (client-configurable per endpoint), idempotency/retry-safety, rate-limiting, pagination, versioning/backward-compat, data-driven testing, async/webhook testing, mocking/virtualization, environment management, CI/CD integration, reporting. Plus protocol-specific: GraphQL and gRPC/protobuf.

Each capability is RICE-scored (Reach × Impact × Confidence / Effort) for build-order sequencing — top of the list: environment/sandbox mgmt (40), functional/happy-path (30), human-review gate on auto-updates (18), CI/CD & reporting (15). Note: the two differentiator items (release-over-release detection, auto-regeneration) score mid-table only because of low confidence/high effort — **not** because they're low priority; their Impact is rated max (3).

### Trust & enterprise-readiness layer

- False-positive rate is the make-or-break metric — target below the ~2–3% "trust cliff"
- HITL as an explicit, auditable mechanism (Human Review Ratio, Human-AI Agreement Rate, Explainability Index), reusing epic HGSQA-778
- Business-logic assertions (not just schema shape) as the real differentiator
- Sandboxed/synthetic credentials + tiered data-masking (reuses HGSQA-970)
- "What we did NOT test and why" coverage-attribution transparency report
- SOC2/ISO27001 + structured pilots = table stakes, flagged to Praveen as company-level, not build-blocking

### Differentiator feature — continuous maintenance (concrete mechanics)

1. Release-over-release change detection (spec diff or live-behavior diff)
2. Regenerate/update only affected tests
3. Regression-safe evolution — retire dead tests rather than let them rot
4. Human-review gate on auto-updates (reuses HITL pipeline)
5. Coverage attribution over time

Depends on HGSQA-968 (multi-target projects — API as a target type).

### Phased build plan

| Tier                           | Scope                                                                                                         | Timing                                      |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| 0 — Foundation                 | Target registration, generation v0 (spec + product knowledge), basic functional tests                         | Jul 14–16 spike                             |
| 1 — v1 client-usable           | Schema/contract, negative/edge-case fuzzing, auth/authz matrix, pagination, per-endpoint SLA, basic reporting | Jul 20–Aug 14 (mid-Aug commit)              |
| 2 — Maintenance engine         | Change detection, auto-regeneration, regression-safe evolution, human-review gate, coverage attribution       | Aug 18–Sep 5 (pulled forward, not optional) |
| 3 — Enterprise/security depth  | OWASP Top 10, idempotency, rate-limiting, CDC                                                                 | Sep–Oct                                     |
| 4 — Protocol expansion         | GraphQL, gRPC                                                                                                 | Only on real client demand                  |
| 5 — Trust/enterprise-readiness | FP-rate benchmark, HITL metrics, data-masking, coverage report                                                | Parallel, partly product/compliance-owned   |

### Single-FTE roadmap (Khoa Hung, starting Jul 14)

- **Wk1 (Jul 14–18):** Spike + gate — target registration, generation v0 on one client's OpenAPI spec + product knowledge
- **Wk2 (Jul 21–25):** Generation engine v0, ship per-endpoint SLA config UI + violation report
- **Wk3 (Jul 28–Aug 1):** Auth/authz matrix (BOLA/BFLA), deeper negative-case gen, pagination/versioning
- **Wk4 (Aug 4–8):** Codebase-aware generation (opt-in GitHub), productize target registration/pricing
- **Wk5 (Aug 11–15):** First real client delivery (mid-Aug commitment)
- **Wk6–8 (Aug 18–Sep 5):** Maintenance engine
- **Wk9+:** OWASP/idempotency/rate-limiting/CDC depth; GraphQL/gRPC only on demand

### Open questions

|Question|Owner|Needed by|
|---|---|---|
|Change-detection mechanism|Khoa Hung|Jul 16 (spike)|
|Re-scope mid-Aug commitment to "Tier 0+1 only"?|Praveen|Before Aug roadmap train|
|SOC2/ISO27001 pursuit|Praveen|Cross-ref security-compliance-roadmap.md|
|Where HITL metrics get built (existing HGSQA-943 vs. separate)|Wayne / Khoa Hung|Before Tier 5|

### Onboarding — this week's investigation tasks (Khoa Hung)

1. **Understand existing scenario generation** — start with `services/scenarios/infrastructure/app_graph_reader.py`; talk to Hung Ngo Luu Tan (Scenario-Gen DRI) first
2. **Design baseline vs. enhanced (GitHub opt-in) generation tiers** — check `services/integrations/` for what GitHub access already supports
3. **Sketch maintenance infrastructure** — candidate-vs-canonical versioning, bot-committed sync (Dependabot-style), CLI surface

These are investigation tasks, not a locked plan — findings get reported back and the roadmap gets revised accordingly.

---

Want me to turn this into a downloadable Word doc, or trim it down to a one-page executive version?******