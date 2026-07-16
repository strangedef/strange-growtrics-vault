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