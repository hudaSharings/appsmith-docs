# Current System — API Endpoint Catalog

[← Index](../README.md) · [← Cross-Cutting Concerns](09-cross-cutting-concerns.md)

---

Every public route, with its future owning service. This is the working checklist for the gateway routing table and for tracking rewrite coverage.

**Owner legend:** `IAM` Identity & Access · `APP` Application · `DS` Datasource · `EXEC` Query Execution · `GIT` Git Versioning · `RT` Realtime · `NOTIF` Notifications & Telemetry · `GW` Gateway itself

Base paths come from `constants/ce/UrlCE.java`. All routes are prefixed `/api/v1`.

---

## Auth & users — `/users`, `/login`, `/logout`

| Method | Path | Purpose | Owner |
|---|---|---|---|
| POST | `/login` | Form-POST login (Spring Security filter chain, no controller) | IAM |
| POST | `/logout` | End session | IAM |
| POST | `/users` | Signup (form-encoded). Creates user + workspace + 3 roles | IAM |
| POST | `/users/super` | First-user / instance-admin signup | IAM |
| GET | `/users/me` | Current user profile | IAM |
| PUT | `/users` | Update profile | IAM |
| POST | `/users/forgotPassword` | Start password reset | IAM (+NOTIF for the email) |
| GET | `/users/verifyPasswordResetToken` | Validate a reset token | IAM |
| PUT | `/users/resetPassword` | Complete reset | IAM |
| POST | `/users/resendEmailVerification` | Re-send verification | IAM (+NOTIF) |
| POST | `/users/invite` | Invite users to a workspace role | IAM (+NOTIF) |
| PUT | `/users/leaveWorkspace/{workspaceId}` | Leave a workspace | IAM |
| GET/POST/DELETE | `/users/photo`, `/users/photo/{email}` | Profile photo (Asset) | IAM (blob → object storage) |
| GET | `/users/features` | Feature flags for this user | GW (flag provider) |
| PUT | `/users/setReleaseNotesViewed` | UI state | IAM |
| PUT | `/users/applications/{applicationId}/favorite` | Toggle favourite | IAM (userData) |
| GET | `/users/favoriteApplications` | Favourites list | IAM |
| POST | `/users/ai-assistant/request` | AI assistant request | APP |

## Workspaces — `/workspaces`

| Method | Path | Purpose | Owner |
|---|---|---|---|
| POST | `/workspaces` | Create workspace + default roles | IAM |
| GET | `/workspaces/home` | Workspaces for the current user | IAM |
| GET/PUT/DELETE | `/workspaces/{id}` | CRUD | IAM |
| GET | `/workspaces/{id}/permissionGroups` | Roles in the workspace | IAM |
| GET | `/workspaces/{id}/members` | Members | IAM |
| PUT | `/workspaces/{id}/permissionGroup` | Change a member's role | IAM |
| POST/DELETE | `/workspaces/{id}/logo` | Workspace logo (Asset) | IAM |

## Organization / tenant — `/tenants`

| Method | Path | Purpose | Owner |
|---|---|---|---|
| GET | `/tenants/current` | Instance configuration (branding, auth methods) | IAM |
| PUT | `/tenants` | Update instance config | IAM |
| GET/PUT | `/tenants/ai-config` | AI provider configuration | APP |
| POST | `/tenants/ai-config/test-connection`, `/fetch-models`, `/test-api-key` | AI provider probes | EXEC |

> `Tenant` and `Organization` are two near-duplicate collections mid-migration. The target collapses them into a single `Instance` row owned by IAM. **Keep the `/tenants` route shape at the gateway** so the client doesn't churn, or rename deliberately as part of the client rewrite.

## Applications — `/applications`

| Method | Path | Purpose | Owner |
|---|---|---|---|
| POST | `/applications` | Create | APP |
| GET | `/applications/home` | Application list for the home screen | GW (composition) |
| GET/PUT/DELETE | `/applications/{branchedApplicationId}` | CRUD | APP |
| POST | `/applications/publish/{id}` | Publish | APP |
| GET | `/applications/view/{id}` | Published application | APP |
| PUT | `/applications/{id}/changeAccess` | Make public/private (writes the anon group into policies) | APP + IAM event |
| POST | `/applications/clone/{id}` | Clone within a workspace | APP |
| POST | `/applications/{id}/fork/{workspaceId}` | Fork to another workspace | APP (saga → DS) |
| GET | `/applications/export/{id}` | Export as JSON | APP (+DS for configs) |
| POST | `/applications/import/{workspaceId}` | Import from JSON | APP (saga → DS) |
| POST | `/applications/import/partial/block` | Partial import (building blocks) | APP |
| GET | `/applications/import/{workspaceId}/datasources` | Datasources needed by an import | DS |
| GET/POST/DELETE | `/applications/snapshot/{id}`, `/restore` | Snapshots | APP |
| PUT | `/applications/{id}/page/{pageId}/makeDefault`, `/reorder` | Page ordering | APP |
| PATCH | `/applications/{id}/themes/{themeId}` | Apply a theme | APP |
| POST/DELETE | `/applications/{id}/logo` | App logo (Asset) | APP |
| POST/GET | `/applications/ssh-keypair/{id}` | Git SSH keys | GIT |
| GET/POST/PATCH/DELETE | `/applications/{id}/static-url*` | Custom slugs for published apps | APP |
| GET | `/applications/releaseItems` | Release notes | GW |

## Pages — `/pages`

| Method | Path | Purpose | Owner |
|---|---|---|---|
| POST | `/pages` | Create page | APP |
| GET/PUT/DELETE | `/pages/{branchedPageId}` | CRUD | APP |
| GET | `/pages/{id}/view` | Published page | APP |
| GET | `/pages/application/{applicationId}` | Pages of an app (edit) | APP |
| GET | `/pages/view/application/{applicationId}` | Pages of an app (view) | APP |
| POST | `/pages/clone/{id}` | Clone a page (with its actions) | APP |
| POST/PUT | `/pages/crud-page`, `/pages/crud-page/{id}` | Generate a CRUD page from a table | APP (+EXEC for introspection) |
| PUT | `/pages/{defaultPageId}/dependencyMap` | Client-computed dependency map | APP |
| PATCH/GET | `/pages/static-url`, `/pages/{id}/static-url/verify/{slug}` | Page slugs | APP |

## Layouts — `/layouts`

| Method | Path | Purpose | Owner |
|---|---|---|---|
| GET | `/layouts/{layoutId}/pages/{pageId}` | Get layout (edit) | APP |
| GET | `/layouts/{layoutId}/pages/{pageId}/view` | Get layout (view) | APP |
| PUT | `/layouts/{layoutId}/pages/{pageId}` | **Save DSL + recompute the on-load plan** | APP (needs binding analysis) |
| PUT | `/layouts/application/{applicationId}` | App-level layout (screen size) | APP |
| PUT | `/layouts/refactor` | Rename an entity and rewrite every binding referencing it | APP |

## Actions — `/actions`, `/collections/actions`

| Method | Path | Purpose | Owner |
|---|---|---|---|
| POST | `/actions` | Create action | APP |
| GET | `/actions` | Actions by application/page | APP |
| GET | `/actions/view` | Actions for a published app | APP |
| PUT | `/actions/{branchedActionId}` | Update action | APP |
| DELETE | `/actions/{id}` | Delete | APP |
| PUT | `/actions/move`, `/actions/refactor` | Move between pages / rename | APP |
| PUT | `/actions/executeOnLoad/{id}`, `/actions/runBehaviour/{id}` | On-load flags | APP |
| **POST** | **`/actions/execute`** | **Run a query (multipart)** | **APP → EXEC (gRPC)** |
| GET/PATCH/DELETE | `/collections/actions*` | JS objects (ActionCollection) | APP |
| PUT | `/collections/actions/{id}/body`, `/refactorAction`, `/move`, `/refactor` | JS object edits | APP |

## Datasources & plugins — `/datasources`, `/plugins`, `/saas`

| Method | Path | Purpose | Owner |
|---|---|---|---|
| POST | `/datasources` | Create | DS |
| PUT/DELETE | `/datasources/{id}` | Update / delete | DS |
| PUT | `/datasources/datasource-storages` | Update per-environment config | DS |
| POST | `/datasources/test` | Test connection | DS → EXEC |
| GET | `/datasources/{id}/structure` | Schema introspection | EXEC (cached in DS) |
| POST | `/datasources/{id}/trigger` | Dynamic dropdown data | EXEC |
| POST | `/datasources/{id}/schema-preview` | Schema preview | EXEC |
| GET | `/datasources/{id}/pages/{pageId}/code` | Generate CRUD code | APP + EXEC |
| GET | `/datasources/authorize` | OAuth callback landing | DS |
| GET/POST | `/datasources/mocks` | Mock datasource catalog | DS |
| GET | `/plugins` | Plugin catalog | EXEC (replicated to DS) |
| GET | `/plugins/{id}/form` | Connector form schema for the UI | EXEC |
| POST | `/plugins/{id}/trigger` | Plugin-level trigger | EXEC |
| GET | `/plugins/default/icons`, `/upcoming-integrations` | Catalog metadata | EXEC |
| POST | `/saas/{datasourceId}/oauth`, `/token`, GET `/saas/authorize` | OAuth2 flows for SaaS datasources | DS |

## Git — `/git`

| Method | Path | Purpose | Owner |
|---|---|---|---|
| POST/GET | `/git/profile/default`, `/git/profile/app/{id}` | Git author identity | GIT |
| GET | `/git/metadata/app/{id}` | Branch/remote metadata | GIT |
| POST | `/git/connect/app/{id}` | Connect to a remote | GIT |
| POST | `/git/commit/app/{id}` | Commit (+ optional push) | APP → GIT |
| POST | `/git/push/app/{id}` | Push | GIT |
| GET | `/git/pull/app/{id}` | Pull + re-import | GIT → APP |
| POST/GET/DELETE | `/git/create-branch/...`, `/git/branch/app/{id}` | Branch management | GIT (+APP for tree duplication) |
| GET | `/git/checkout-branch/app/{id}` | Checkout | GIT + APP |
| GET | `/git/status/app/{id}`, `/git/fetch/remote/app/{id}` | Status / fetch | GIT |
| POST | `/git/merge/app/{id}`, `/git/merge/status/app/{id}` | Merge | GIT |
| PUT | `/git/discard/app/{id}` | Discard local changes | GIT + APP |
| POST | `/git/disconnect/app/{id}` | Disconnect | GIT |
| GET/POST | `/git/branch/app/{id}/protected` | Branch protection | GIT |
| POST/GET/PATCH | `/git/auto-commit/...` | Auto-commit + progress | GIT |
| GET | `/git/import/keys`, POST `/git/import/{workspaceId}` | Import an app from a repo | GIT → APP |
| GET | `/git/protocol/key-types`, `/git/doc-urls` | Static metadata | GIT |

## Themes, JS libraries, templates

| Method | Path | Purpose | Owner |
|---|---|---|---|
| GET | `/themes/applications/{id}`, `/current` | Themes for an app | APP |
| PUT/PATCH | `/themes/applications/{id}`, `/themes/{themeId}` | Apply / customise | APP |
| PATCH | `/libraries/{applicationId}/add`, `/remove` | Custom JS libraries | APP |
| GET | `/libraries/{contextId}`, `/{contextId}/view` | List | APP |
| GET | `/app-templates`, `/{id}`, `/{id}/similar`, `/filters` | Template catalog (remote) | GW → APP |
| POST | `/app-templates/{id}/import/{workspaceId}`, `/merge/...` | Instantiate a template | APP |
| POST | `/app-templates/publish/community-template`, `/use-case` | Publish to the community | APP |

## Import, assets, search, config, admin, health

| Method | Path | Purpose | Owner |
|---|---|---|---|
| POST | `/import/...` (curl / OpenAPI / Postman) | Import an API definition into actions | APP |
| GET | `/assets/{id}` | Serve a stored image (`permitAll`) | GW → object storage |
| GET | `/search-entities` | Cross-entity search (apps, pages, queries, datasources) | GW (fan-out) |
| GET/PUT | `/configs/name/{name}` | Instance key/value config | IAM |
| GET | `/admin/env` | Read instance env config | IAM |
| POST | `/admin/restart`, `/admin/send-test-email` | Instance operations | **Drop `restart`.** Test email → NOTIF |
| GET | `/health` | Health check | Every service (`/health/live`, `/health/ready`) |
| POST | `/usage-pulse` | Anonymous activity beacon | NOTIF |
| GET | `/product-alert/alert` | Product alerts | NOTIF |
| GET | `/consolidated-api/edit`, `/consolidated-api/view` | **Editor / viewer boot payload** | **GW (BFF)** |

## Non-HTTP surfaces

| Surface | Purpose | Owner |
|---|---|---|
| Socket.IO `/rts` (app + page namespaces) | Presence, live cursors, collaborator indicators | RT (SignalR) |
| RTS `POST /rts-api/v1/ast/multiple-script-data` | Binding identifier extraction | Internal to APP |
| RTS DSL routes | DSL migration helpers | Internal to APP |

---

## Anonymous-accessible routes

These are `permitAll` in `SecurityConfig` and must remain reachable without a session at the gateway. Authorization still happens — via the anonymous permission group ([Permissions §6](04-permissions-and-acl.md#6-public-anonymous-applications)).

```
GET  /health
POST /users, /users/super, /users/forgotPassword
GET  /users/verifyPasswordResetToken, /users/me
PUT  /users/resetPassword
GET  /assets/*
GET  /actions/**, /pages/**, /applications/**, /themes/**
GET  /collections/actions/view
POST /actions/execute
GET  /tenants/current
POST /usage-pulse
GET  /libraries/*/view
GET  /product-alert/alert
GET  /consolidated-api/view
     /public/**, /oauth2/**
```

## Coverage summary

| Owner | Approx. endpoint count |
|---|---|
| APP — Application | ~75 |
| IAM — Identity & Access | ~30 |
| DS — Datasource | ~15 |
| GIT — Git Versioning | ~28 |
| EXEC — Query Execution | ~10 (but the highest traffic) |
| GW — Gateway/BFF | ~8 composition + all routing |
| NOTIF | ~3 |
| RT | WebSocket only |

**Application Service is the largest surface by far** — expected, since the editor is the product. That is a team-sizing input, not an argument for splitting it further ([Service Inventory](../02-target-architecture/01-service-inventory.md) explains why splitting it would be a mistake).

---

[← Cross-Cutting Concerns](09-cross-cutting-concerns.md) · [Next: Service Inventory →](../02-target-architecture/01-service-inventory.md)
