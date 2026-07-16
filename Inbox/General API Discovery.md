#### 2. Schema Extraction Layer (get the API's shape)

For each discovered vendor, try in order:

1. **Published spec** — check `/openapi.json`, `/swagger.json`, GraphQL introspection, known docs URLs
2. **Traffic-based inference** — from captured request/response pairs (proxy), build JSON Schema per endpoint (union of observed fields, required = always-present, optional = sometimes-present)
3. **Changelog/docs scraping** — fallback when no spec exists, diff docs page text over time

→ Output: **Schema Snapshot** per endpoint you actually use (not vendor's full API — just your subset)