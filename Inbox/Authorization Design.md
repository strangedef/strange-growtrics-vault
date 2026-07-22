# StackHawp-style approach

## Core idea

Separate **authorization model declaration** (what the user provides) from **test generation** (what your system does automatically). The user should only ever answer: _"who can do what to which resources"_ — never _how_ to test it.

## 1. Input layer (what the user configures)

Keep it to 3 things, expressed declaratively (YAML/JSON or a UI form, not code):

```yaml
roles:
  - name: admin
    scopes: ["*"]
  - name: user
    scopes: ["read:own", "write:own"]
  - name: guest
    scopes: ["read:public"]

resources:
  - path: /orders/{id}
    owner_field: "userId"      # tells system how to detect ownership
    allowed_roles:
      GET: [admin, user]
      DELETE: [admin]

credentials:
  admin: <token or login flow ref>
  user: <token or login flow ref>
  guest: <token or login flow ref>
```

That's it — user never thinks about test cases, only permissions. This is the key UX shift from StackHawk (which makes users think in scan configs) to Testsigma-style (users think in business rules).

## 2. Auto-generation engine (the "brain")

This is what does the heavy lifting your current design is missing:

- **Parse OpenAPI spec** → extract every endpoint + method + path params.
- **Cross-reference with role/resource declarations** → for each endpoint, compute the full access matrix (which roles _should_ pass, which _should_ fail).
- **Auto-generate test case types** without user input:
    - **Vertical privilege escalation**: lower role attempts higher role's allowed action.
    - **Horizontal privilege escalation (BOLA/IDOR)**: same role, different `owner_field` value — auto-generate by creating two same-role users and swapping resource IDs.
    - **Scope violation**: token with insufficient scope attempts an action requiring more.
    - **Auth bypass**: no token / expired token / malformed token.
- **Resource ID discovery**: if user doesn't manually provide two sample resource owners, your system should auto-create test fixtures (call the API as User A to create a resource, then try accessing it as User B) — this removes another manual step competitors require.

## 3. Execution workflow (fully automatic, hidden from user)

```
Spec + Role Model → Test Matrix Generator → Fixture Setup (auto-create test data per role)
    → Execute all generated requests → Compare actual vs expected (from matrix)
    → Flag mismatches as authz violations
```

## 4. What this buys you over StackHawk

||StackHawk|Your target design|
|---|---|---|
|Input|Spec + raw credentials|Spec + role/scope model|
|BOLA detection|Requires you to structure credential sets correctly|System infers owner relationships and auto-creates cross-user test data|
|Coverage|Endpoint-level pass/fail|Full expected-vs-actual matrix per role×endpoint×action|
|New endpoint added|You manually add test config|Auto-covered if it matches an existing resource pattern in the spec|

## 5. Practical build note

The hardest part isn't the workflow UI — it's **owner/resource-relationship inference**. You either:

- Require the user to annotate `owner_field` in the resource declaration (still low-code, one field), or
- Try to infer it via naming heuristics (`userId`, `ownerId`, `createdBy`) + confirm via a test call, falling back to asking the user only when inference fails.

That fallback-only-when-needed pattern is what makes something feel "low configuration" — you don't ask for everything upfront, you ask only when auto-inference genuinely can't resolve it.