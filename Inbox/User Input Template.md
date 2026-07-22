
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
