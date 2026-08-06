- invariants field -> predicate != req/res field
- chunking strategy -> section/heading -> docs change -> chunk changes -> tests change

~~**SpecWatch**: Where to auto-fetch the spec (`github_webhook` / `poll_url`); triggers regeneration only -> remove it~~

# SuiteRun (singgle) ScenarioRun (mul)
Correct. Both hang off one SuiteRun (execution of suite against environment):

SuiteRun
 ├─ RunResult      per TestCase   — http_status, latency_ms, assertion_results[], evidence, outcome
 └─ ScenarioRun    per TestScenario — outcome, step_results[]
      └─ StepResult (embedded) per step — step_id, op, http_status, latency_ms, captured{}, assertion_results[], outcome

Symmetry: StepResult is to ScenarioRun what RunResult is to SuiteRun — but embedded, not own row, matching step-inside-scenario containment.

Outcomes: pass | fail | error | skipped on both. Scenario-specific: step failure skips transitive depends_on dependents, their StepResults come out skipped with unmet-dependency reason; scenario outcome aggregates.

Finding references either via test_ref; evidence chain resolves into RunResult.evidence or step's captured/evidence data.
### Where does it store state of previous step
Two places — one for use during the run, one for the record afterwards:

1. In memory, during execution. While the runner walks a scenario, captured values live in a per-scenario-run variable context held by the runner itself. Each step declares what to capture via its capture field (ScenarioStep.capture{var → body JSONPath / Location header / status}) — e.g. s1 captures order_id from the create response body. Later steps reference that named variable in their request templates (GET /orders/{order_id}), and assertions can also address prior steps directly (e.g. s1.response.total). This context is scoped to the single scenario run — it isn't shared across scenarios or across runs.

2. Persisted, after each step. Each step's result is written to StepResult.captured{} (embedded in the ScenarioRun). That's the durable record — it's evidence for the audit trail ("the id we deleted in s5 is the one s1 created"), not the mechanism later steps read from at runtime.
### why we split TestScenarios vs TestCase
Because they obey different execution contracts. A test case is independent: run in any order, in parallel, dedupe identical requests, retry or retire one without touching anything else. A scenario is a stateful sequence: strict order, values captured from earlier steps feed later ones, a failed step cancels its dependents.

Splitting the entities lets the runner, the coverage report (which groups by per-case `category`), and the skip/failure logic each dispatch on type instead of guessing from step count. Merging them would save one table but smear those two contracts into one entity with conditionals everywhere.