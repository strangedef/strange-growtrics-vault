### Fuzzing
Negative/edge-case fuzzing just means: **instead of testing that the API works correctly, you test that it fails safely.**

Think of it like this — normal testing checks "if I order 1 ticket, do I get 1 ticket confirmed?" That's the happy path.

Fuzzing does the opposite. It deliberately sends the API bad, weird, or extreme input to see what happens:

- **Malformed** — send garbage data where a proper value is expected (e.g., sending `"banana"` where a date is expected, or a broken/incomplete JSON body)
- **Boundary** — push values to their limits (e.g., order **0** tickets, order **-1** tickets, order **999999999** tickets, send a name that's 10,000 characters long)
- **Wrong-type** — send the wrong kind of data entirely (e.g., sending text `"abc"` into a field that expects a number, or sending `true` where a price is expected)

**Why it matters:** a well-built API should reject this stuff gracefully — return a clean error message like "invalid input" — instead of crashing, freezing, or (worse) silently accepting the bad data and corrupting something in the database. If it _does_ crash or misbehave, that's a real bug you just found before a real user (or attacker) stumbled into it accidentally.

That's also why the spec calls this a "high-value" capability — real-world bugs cluster heavily around this kind of unexpected input, way more than around the happy path everyone already tests.
### BOLA/BFLA
This is testing two different things that people often mix up: **"are you logged in?"** vs. **"are you allowed to do this specific thing?"**

**1. The credential checks (missing/invalid/expired/wrong-scope)**

This is the basic "who are you" layer — authentication. You systematically hit every endpoint with:

- **Missing** credentials — no token at all
- **Invalid** credentials — a fake or malformed token
- **Expired** credentials — a token that used to work but shouldn't anymore
- **Wrong-scope** credentials — a token that's valid, but only for limited permissions (e.g., a "read-only" token trying to do a "delete" action)

The API should reject all of these consistently. Sounds obvious, but it's easy for one endpoint to get missed when a team is manually building these checks by hand.

**2. BOLA/BFLA — the more dangerous one**

This is authoriz_ation_, not authentic_ation_ — meaning: you _are_ logged in validly, but should you be allowed to see or touch _this particular thing_?

- **BOLA** (Broken Object Level Authorization) — can User A access User B's _data_? Example: you're logged into your own account, but you change `/orders/1234` to `/orders/1235` in the URL and suddenly you're looking at a stranger's order.
- **BFLA** (Broken Function Level Authorization) — can a regular user access an _admin-only action_? Example: a normal user calls the "delete user" endpoint that should only work for admins.

**"Multi-tenant object-level authorization replay"** just means the actual test technique: log in as Account B, try to request Account A's object, and confirm the API says **403 Forbidden** — not confirm it just returns the data.

**Why this is called out specially:** the spec notes this is the single most common and most severe real-world API bug class — roughly 40% of real API attacks exploit BOLA specifically — and it's the one category that automated scanners tend to miss, because it requires understanding _whose data is whose_ (business context), not just checking if a token is valid. That's exactly why it fits TaloTrace's "business-knowledge-grounded testing" pitch rather than being commodity work.

# Software Authorization Types

**Authorization** determines _what a user is allowed to do_ (different from authentication, which verifies _who they are_).

## Main Models

| Type                                          | Description                                                       | Example                                                |
| --------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------ |
| **DAC** (Discretionary Access Control)        | Resource owner decides who can access it                          | File sharing permissions (Google Docs "share with...") |
| **MAC** (Mandatory Access Control)            | System enforces fixed security labels/rules, users can't override | Military/government classified systems                 |
| **RBAC** (Role-Based Access Control)          | Permissions assigned to roles, users assigned to roles            | Admin, Editor, Viewer roles                            |
| **ABAC** (Attribute-Based Access Control)     | Access based on attributes (user, resource, environment)          | "Managers can approve if amount < $10k"                |
| **PBAC** (Policy-Based Access Control)        | Centralized policies define access rules                          | OPA (Open Policy Agent) policies                       |
| **ReBAC** (Relationship-Based Access Control) | Access based on relationships between entities                    | "Only friends can see this post" (social networks)     |
| **ACL** (Access Control List)                 | List attached to a resource specifying who can do what            | File system permissions (rwx)                          |
| **Capability-based**                          | Possessing a "token/capability" grants access                     | OAuth tokens, API keys                                 |

## Most Popular in Practice

1. **RBAC** — most widely used in enterprise apps, easy to understand and manage
2. **ABAC** — used when RBAC becomes too rigid, needed for fine-grained/dynamic rules
3. **ReBAC** — increasingly popular for modern apps (Google Zanzibar model — used by Slack, GitHub, Canva)
4. **OAuth 2.0 / OIDC scopes** — the de facto standard for API/third-party authorization

## Quick Recommendation

- **Simple apps** → RBAC
- **Complex, dynamic rules** → ABAC or PBAC
- **Social/collaborative apps with nested permissions** → ReBAC
- **APIs / third-party access** → OAuth 2.0 scopes

# Solving Multi-Tenancy: Scope + RBAC Together

The key idea: **don't put per-org roles in the token's scope.** Instead, keep scope generic, and resolve the role **dynamically at request time** based on which tenant the request is targeting.

## Approach 1: Generic Scope + Tenant Context Header (Most Common)

**Token scope stays simple and tenant-agnostic:**

```
scopes: ["tasks:read", "tasks:write"]
```

This just says: "this token is _capable of_ read/write actions in general" — not tied to any specific org.

**Every request includes a tenant/org context:**

```
GET /orgs/{org_id}/projects/42/tasks
Authorization: Bearer <token>
```

**At request time, the API does a lookup:**

```
1. Check scope → does token have "tasks:write"? ✅
2. Check RBAC → SELECT role FROM memberships 
                 WHERE user_id = X AND org_id = {org_id}
   → Returns: "Viewer" for Org B
3. Decision: Viewer + write attempt → ❌ Deny
```

The role is **never stored in the token** — it's fetched fresh from the database every time, scoped to whichever org the URL/request targets. This solves stale-token and per-org variation in one shot.

---

## Approach 2: Tenant-Scoped Tokens (Used by Slack, Google Workspace-style APIs)

Instead of one token for "the user everywhere," issue a **separate token per org**, obtained when the user selects/switches organization:

```
User logs in → selects "Org A" → 
Token issued: { sub: "alice", org_id: "A", scope: "tasks:write", role: "editor" }
```

If Alice switches to Org B, she gets a **new token**:

```
Token issued: { sub: "alice", org_id: "B", scope: "tasks:read", role: "viewer" }
```

✅ Solves the bloat problem — each token only carries the role for **one** org ✅ Common in apps with an explicit "workspace switcher" UI (Slack, Notion, Linear) ❌ Requires re-authentication/token exchange on every org switch

---

## Approach 3: JWT with Compact Org-Role Map (For Users in Few Orgs)

If a user only belongs to a handful of orgs (not thousands), embed a **compact map** in the JWT claims instead of full scope explosion:

```json
{
  "sub": "alice",
  "scope": "tasks:read tasks:write",
  "orgs": {
    "org_A": "admin",
    "org_B": "viewer"
  }
}
```

The API then does:

```
1. Scope check: "tasks:write" in token? ✅
2. RBAC check: token.orgs[request.org_id] == "admin" or "editor"? 
   → org_B = "viewer" → ❌ Deny
```

✅ No DB lookup needed (fast, stateless) ❌ Doesn't scale if a user is in hundreds/thousands of orgs (token size limit) ❌ Still stale until token refresh — role changes mid-session won't apply until re-issued

---

## Comparison

|Approach|Role Source|Freshness|Best For|
|---|---|---|---|
|**Generic scope + DB lookup**|Database (live)|Always fresh|Most SaaS apps, any # of orgs|
|**Tenant-scoped token**|Token (per-org)|Fresh on switch|Apps with explicit workspace switching|
|**Compact org-role map in JWT**|Token (embedded)|Stale until refresh|Users in few orgs, need low-latency/stateless checks|

---

## Real-World Example: Slack's Model

Slack uses **Approach 2**: your OAuth token is scoped to **one workspace at a time**. If your app needs access to multiple Slack workspaces, it must obtain **separate tokens per workspace** — each with its own scope and the user's role in that specific workspace baked in.

## Real-World Example: GitHub's Model

GitHub uses a hybrid: **Approach 1** for org-level checks (role fetched live via `org_id` + `user_id`), combined with **repo-level RBAC** — since a user's role can differ per-repository even within the same org (e.g., Write on Repo X, Admin on Repo Y).

---

**Bottom line:** the pattern is always the same — **keep scope generic (what kind of action), keep tenant/role resolution dynamic and server-side (who can do it, where)**. Never try to encode "user X's role in every possible org" into a single static token.

