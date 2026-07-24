-  is item_path and collection_path redundant as in each operation, we already have path
- does the requirement need teardown
- why don't group authz into operations like what we did in resouces[]?

```yaml
resources:
  - name: order
    owner_field: userId
    tenant_field: tenantId
    operations:
      create: { method: POST,   path: /orders,      allowed_roles: [user, admin], required_scope: "write:own" }
      read:   { method: GET,    path: /orders/{id}, allowed_roles: [user, admin], required_scope: "read:own" }
      list:   { method: GET,    path: /orders,      allowed_roles: [user, admin], required_scope: "read:own" }
      update: { method: PUT,    path: /orders/{id}, allowed_roles: [user, admin], required_scope: "write:own" }
      delete: { method: DELETE, path: /orders/{id}, allowed_roles: [admin],       required_scope: "write:own" }
```
