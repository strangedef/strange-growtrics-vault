#### 1. Discovery Layer
```
Try published spec (/openapi.json, GraphQL introspection)
        │
        ├─ found → use it directly
        │
        └─ not found
              │
              ▼
        Capture traffic (mitmproxy / Postman) while exercising the API
              │
              ▼
        Convert captured traffic → schema (mitmproxy2swagger, or manual)
              │
              ▼
        Validate with edge-case calls (missing fields, errors, headers)
```

##### Setting up Proxy
The visual tool isn't cooperating right now, so here's a clear diagram in text form plus the setup steps.

## How the proxy sits in the middle

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│              │ request │              │ request │              │
│  Your Client │ ──────► │   mitmproxy  │ ──────► │ External API │
│ (app/script/ │         │ (intercepts  │         │              │
│  Postman)    │ ◄────── │  & logs)     │ ◄────── │              │
│              │ response│              │ response│              │
└──────────────┘         └──────────────┘         └──────────────┘
                                │
                                ▼
                        Saved traffic log
                        (every request +
                         response, HAR file)
```

The client thinks it's talking straight to the API. It's actually talking to mitmproxy, which forwards the call, records both sides, and passes the response back untouched.

## Setup steps (mitmproxy)

**1. Install**

```
pip install mitmproxy
```

**2. Start it**

```
mitmweb        # opens a browser UI at http://127.0.0.1:8081 to watch traffic live
```

This starts a proxy listening on `127.0.0.1:8080` by default.

**3. Point your client at the proxy**

- **curl:** `curl -x http://127.0.0.1:8080 https://api.vendor.com/v1/users`
- **Postman:** Settings → Proxy → set proxy server to `127.0.0.1:8080`
- **A mobile app or another machine:** set that device's Wi-Fi proxy settings to your machine's IP + port 8080

**4. Handle HTTPS** Since the API is HTTPS, mitmproxy needs to decrypt it — this means your client has to trust mitmproxy's certificate:

- Visit `http://mitm.it` from the proxied client/browser → download and install the cert for your OS/device
- Without this step you'll get SSL errors, since mitmproxy is technically doing a man-in-the-middle

**5. Exercise the API** Use the client normally (click through the app, run your test script, hit endpoints in Postman) — every call now shows up live in the mitmweb UI.

**6. Export the captured traffic**

```
mitmdump -r captured_flows -w output.har   # or use the "Save" option in mitmweb
```

Feed that `.har` file into `har-to-openapi` or similar to generate your schema.

One thing worth flagging: this only works for traffic _you_ control (your own devices/clients). You can't proxy an API you have no client for — you still need something (script, app, Postman) actually making calls through it.

#### 2. Schema Extraction Layer (get the API's shape)

For each discovered vendor, try in order:

1. **Published spec** — check `/openapi.json`, `/swagger.json`, GraphQL introspection, known docs URLs
2. **Traffic-based inference** — from captured request/response pairs (proxy), build JSON Schema per endpoint (union of observed fields, required = always-present, optional = sometimes-present)
3. **Changelog/docs scraping** — fallback when no spec exists, diff docs page text over time

→ Output: **Schema Snapshot** per endpoint you actually use (not vendor's full API — just your subset)

#### Approach 2: Traffic-Based Inference

The idea: capture real request/response pairs, then reverse-engineer a schema from the patterns you see across many samples.

**How it works, step by step:**

1. **Group by endpoint.** Bucket captured calls by `method + path pattern` (e.g., `GET /users/{id}` — normalize `/users/123` and `/users/456` into one bucket by detecting numeric/UUID path segments).
2. **Collect N samples per endpoint.** One response isn't enough — you need variety (different users, empty results, error cases) to know what's _always_ there vs _sometimes_ there.
3. **Infer field types.** For each field across all samples, record the observed type(s): string, number, boolean, array, object, null.
    - If a field is always a string → type: string
    - If a field is sometimes a number, sometimes null → type: number, nullable: true
    - If types conflict wildly (string in one response, object in another) → flag it, that's often a versioning issue or a polymorphic field
4. **Infer required vs optional.**
    - Field present in **100% of samples** → likely required
    - Field present in **some but not all** → optional
    - Caveat: this is probabilistic, not certain — 20 samples all having a field doesn't guarantee it's truly required, just likely
5. **Infer enums.** If a field only ever takes 3-4 distinct string values across hundreds of samples (e.g., `status: "active"/"pending"/"closed"`), treat it as a likely enum — but note it as inferred, not confirmed, since a 4th value may just not have appeared yet.
6. **Output as OpenAPI/JSON Schema.** Structure the result the same way a published spec would look, so it plugs into the same diff/catalog pipeline.

**Tools that automate most of this:**

- `mitmproxy2swagger` — converts captured mitmproxy flows directly into an OpenAPI draft
- Postman's "Generate Collection" from a captured traffic dump, then export as OpenAPI
- `optic` — watches traffic continuously and builds/updates a spec over time (better for ongoing monitoring than one-off extraction)

**Key limitation:** you can only infer what you've _observed_. If you never triggered a particular error path, or never saw an admin-only field, it won't be in your schema. This is why Step 2 (edge-case testing — empty inputs, invalid IDs, missing auth) matters: it's how you surface fields/behaviors that normal usage wouldn't show.

#### Approach 2: Changelog/Docs Scraping

Used when there's no machine-readable spec _and_ no traffic to capture yet (e.g., you want to know about a change before you've made any calls, or the API is documented only in prose).

**How it works:**

1. **Identify the source pages** — vendor's `/changelog`, `/release-notes`, or the human-readable API docs page itself.
2. **Snapshot the page content periodically** (daily/weekly): fetch the page, extract the meaningful text (strip nav/footer/ads), store it.
3. **Diff snapshots over time** — plain text diff between today's fetch and the last stored one. A changed paragraph under "v2.3 - added `phone_number` field to /users" is your signal.
4. **Parse structure if the docs are semi-structured.** Many API docs pages (Stripe, Twilio-style) render from an underlying OpenAPI/JSON they don't publicly link — check the page's network requests (via the same proxy technique) for a hidden JSON endpoint the docs UI itself calls. This sometimes gets you a real machine-readable spec even when there's no "official" public one.

**Key limitation:** this is the weakest signal of the two — it depends entirely on the vendor writing accurate, complete changelogs, and text diffing is noisy (formatting changes look like content changes). Treat it as an early-warning trigger to go re-run traffic-based inference, not as a source of truth on its own.