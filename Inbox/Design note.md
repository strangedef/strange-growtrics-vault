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