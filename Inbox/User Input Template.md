
# Full RBAC/Scope BOLA/BFLA config
```yaml
# ==========================================
# 1. ROLES — defines scope sets per role
# ==========================================
roles:
  - name: admin
    scopes: ["*"]
  - name: user
    scopes: ["read:own", "write:own"]
  - name: guest
    scopes: ["read:public"]

# ==========================================
# 2. CREDENTIALS — needs 2+ identities per role
#    (required for BOLA/BFLA cross-user replay)
# ==========================================
credentials:
  admin:
    - id: adminA
      token: <token/login ref>

  user:
    - id: userA
      token: <token/login ref>
    - id: userB
      token: <token/login ref>

  guest:
    - id: guestA
      token: <token/login ref>

# ==========================================
# 3. RESOURCES — endpoint-level authz rules
#    No body_template — system auto-generates
#    payloads from the OpenAPI schema
# ==========================================
resources:
  - path: /orders/{id}
    owner_field: "userId"          # used to detect ownership for BOLA
    tenant_field: "tenantId"       # omit if single-tenant app

    create_via:                    # used to auto-seed resources per identity
      path: /orders
      method: POST

    actions:
      GET:
        allowed_roles: [admin, user]
        required_scope: "read:own"

      PUT:
        allowed_roles: [admin, user]
        required_scope: "write:own"

      DELETE:
        allowed_roles: [admin]
        required_scope: "write:own"
```

Here are standalone, runnable templates for each case.

## 1. RBAC-only API

```yaml
roles:
  - name: admin
  - name: user

credentials:
  admin:
    - id: adminA
      token: <token/login ref>
  user:
    - id: userA
      token: <token/login ref>

resources:
  - path: /reports/{id}
    actions:
      GET:
        allowed_roles: [admin, user]
      PUT:
        allowed_roles: [admin]
      DELETE:
        allowed_roles: [admin]
```

**Runs:** no-creds, invalid-token, vertical privesc, BFLA. **Skips:** scope checks, BOLA checks.

---

## 2. Scope-only API (pure OAuth2, no roles)

```yaml
credentials:
  clientA:
    token: <token ref>
    scopes: ["invoices:read"]
  clientB:
    token: <token ref>
    scopes: ["invoices:read", "invoices:write"]

resources:
  - path: /invoices/{id}
    actions:
      GET:
        required_scope: "invoices:read"
      DELETE:
        required_scope: "invoices:write"
```

**Runs:** no-creds, invalid-token, wrong-scope. **Skips:** role checks, BOLA checks.

---

## 3. BOLA-only API (single role, no scopes — just ownership matters)

```yaml
credentials:
  user:
    - id: userA
      token: <token/login ref>
    - id: userB
      token: <token/login ref>

resources:
  - path: /orders/{id}
    owner_field: "userId"
    create_via:
      path: /orders
      method: POST
    actions:
      GET: {}
      PUT: {}
      DELETE: {}
```

**Runs:** no-creds, invalid-token, BOLA (userA creates → userB tries to access → expect 403/404). **Skips:** role checks, scope checks (no `allowed_roles`/`required_scope` given).

---

## 4. Multi-tenant BOLA API

```yaml
credentials:
  user:
    - id: userA_tenant1
      token: <token/login ref>
    - id: userB_tenant2
      token: <token/login ref>

resources:
  - path: /projects/{id}
    owner_field: "userId"
    tenant_field: "tenantId"
    create_via:
      path: /projects
      method: POST
    actions:
      GET: {}
      DELETE: {}
```

**Runs:** no-creds, invalid-token, BOLA, tenant-isolation (userA_tenant1's project must be unreachable by userB_tenant2 even if role/user check alone would pass). **Skips:** role checks, scope checks.

---

## 5. Full model — RBAC + scope + multi-tenant BOLA combined

```yaml
roles:
  - name: admin
    scopes: ["*"]
  - name: user
    scopes: ["read:own", "write:own"]

credentials:
  admin:
    - id: adminA
      token: <token/login ref>
  user:
    - id: userA
      token: <token/login ref>
    - id: userB
      token: <token/login ref>

resources:
  - path: /invoices/{id}
    owner_field: "customerId"
    tenant_field: "tenantId"
    create_via:
      path: /invoices
      method: POST
    actions:
      GET:
        allowed_roles: [admin, user]
        required_scope: "read:own"
      DELETE:
        allowed_roles: [admin]
        required_scope: "write:own"
```

**Runs:** all five test types — no-creds, invalid-token, vertical privesc/BFLA, wrong-scope, BOLA + tenant-isolation.

---

## 6. Minimal — no auth model declared (baseline only)

```yaml
credentials:
  anyUser:
    token: <token/login ref>

resources:
  - path: /health
    actions:
      GET: {}
```

**Runs:** no-creds, invalid-token only. **Skips:** everything else — this is the floor; every endpoint gets at least this much for free just by having a token and a path.