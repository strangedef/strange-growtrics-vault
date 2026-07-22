So conceptually: tenant = organization, and tenant_id is how the backend scopes every read/write to "only this org's data."

### App

```python
class App(_Row):
    """An app under test (intake)."""
    doc_id: str = Field(default_factory=lambda: new_id("app"), alias="_id")
    project_id: str
    name: str = "app"
    platform: str = "android_emulator"   # or "browser"/"web"
    url: str | None = None               # WEB target's entry URL
    web_allowed_origins: tuple[str, ...] = ()
    web_authority_error: str | None = None
    icon_object_key: str | None = None
    preview_image_url: str | None = None
    preview_title: str | None = None
    preview_description: str | None = None
    favicon_url: str | None = None
```

What it represents: one app-under-test registered inside a project — either a mobile target (Android APK, platform="android_emulator") or a web target (platform="browser"/"web", driven by url instead of a package). It's the unit that scenarios, builds, and runs all hang off.

Relationships:
- project_id — every app belongs to exactly one project (same tenant/project hierarchy as scenarios: tenant → project → app → scenario).
- runtime_target — derived from platform, tells the run/scenario-gen planes which device backend + MCP plane to use.
- Related sibling entity Build (entities.py:96) — one registered build/version of an app (app_id FK), with verification status.

How you get an app_id (mirrors the project_id flow) — services/run_execution/interface/intake_routers.py:45, router prefixed /v1/projects/{project_id}:
- POST /v1/projects/{project_id}/apps — register a new app, returns its id (CreateAppResponse).
- GET /v1/projects/{project_id}/apps — list apps already registered under that project (ListAppsResponse).

So the full chain to call GET /v1/projects/{project_id}/apps/{app_id}/scenarios is: GET /v1/projects → pick project_id → GET /v1/projects/{project_id}/apps → pick app_id.