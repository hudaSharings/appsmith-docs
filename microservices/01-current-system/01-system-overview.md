# Current System — Overview

[← Index](../README.md) · [← Java Primer](../00-orientation/02-java-to-dotnet-primer.md)

---

## 1. What the product does

Appsmith is a **low-code internal-tools builder**. The mental model:

```mermaid
flowchart TD
    O[Organization / Instance] --> W[Workspace<br/><i>tenancy + permission boundary</i>]
    W --> A[Application]
    W --> D[Datasource<br/><i>Postgres, REST, S3, Sheets…</i>]
    A --> P[Page]
    P --> L[Layout / DSL<br/><i>one JSON tree = the canvas</i>]
    A --> AC[Action / Query<br/><i>references a Datasource</i>]
    A --> JS[ActionCollection<br/><i>JS Object = bundle of JS functions</i>]
    A --> T[Theme]
    A --> JL[CustomJSLib]
    L -.->|{{ binding }}| AC
    AC -->|executes against| D
    A -->|Publish| V[Published copy<br/><i>what end users see</i>]
    A -.->|optional| G[Git remote]
```

Two modes exist for every application, and this duality is stamped through the entire data model:

- **Edit mode** — the draft. Fields named `unpublishedX`.
- **View mode** — the published snapshot. Fields named `publishedX`.

Publishing copies draft → published. That is literally what it does; see [Golden Paths §7](05-golden-paths.md#7-publish--deploy).

## 2. Size and shape

| Metric | Value |
|---|---|
| Java backend | 2,143 files · ~163,000 LOC |
| React client | ~5,200 TS/TSX files · ~708,000 LOC |
| Widget types | 82 |
| Connectors (plugins) | 25 |
| REST controllers | 26 |
| MongoDB collections | ~24 |
| Mongock migration classes | 83 |
| Maven modules | 7 (incl. aggregate reports) |

**The client is 4× the backend.** Any plan that treats the UI as a trailing task is wrong. See [Angular Frontend](../02-target-architecture/08-angular-frontend.md).

## 3. Module map

```
app/
├── server/                          ← Java, Maven multi-module
│   ├── appsmith-interfaces/         Shared contracts + the PluginExecutor SPI
│   ├── appsmith-plugins/            25 connector modules
│   ├── appsmith-server/             THE MONOLITH (Spring WebFlux)
│   ├── appsmith-git/                JGit wrapper — zero deps back into the server
│   └── reactive-caching/            Redis-backed cache/lock abstraction
└── client/
    ├── src/                         React 18 + Redux + Redux-Saga SPA
    └── packages/
        ├── rts/                     Node/Express/Socket.IO — a real second service
        ├── ast/                     Shared JS AST parsing (used by both client and RTS)
        ├── dsl/                     DSL schema + migrations
        └── design-system/, wds/…    Component libraries
```

`appsmith-git` and `rts` are the two places where the codebase **already** has clean service boundaries. Both are direct precedents for the target design.

## 4. Bounded contexts as they exist today

The Java package layout is mid-refactor: older code sits in flat packages (`services/`, `solutions/`), newer code in per-entity vertical slices (`applications/`, `datasources/`, `newactions/`, `newpages/`). Read the table by *capability*, not by package.

| Context | Controllers | Base path | Primary entities |
|---|---|---|---|
| **Application / Page** | `ApplicationController`, `PageController`, `LayoutController`, `ApplicationTemplateController` | `/api/v1/applications`, `/pages`, `/layouts`, `/app-templates` | `Application`, `NewPage` (embeds `Layout`) |
| **Action / Query** | `ActionController`, `ActionCollectionController`, `SaasController` | `/api/v1/actions`, `/collections/actions`, `/saas` | `NewAction`, `ActionCollection` |
| **Datasource / Plugin** | `DatasourceController`, `PluginController` | `/api/v1/datasources`, `/plugins` | `Datasource`, `DatasourceStorage`, `Plugin` |
| **Workspace / Org / Tenant** | `WorkspaceController`, `OrganizationController`, `TenantController`, `AIConfigController` | `/api/v1/workspaces`, `/tenants` | `Workspace`, `Organization`, `Tenant` |
| **User / Auth** | `UserController`; login/logout handled by the Spring Security filter chain (no controller) | `/api/v1/users`, `/login`, `/logout` | `User`, `UserData` |
| **Permission / ACL** | none — reached through `solutions/UserAndAccessManagementService` | — | `PermissionGroup`, embedded `Policy` |
| **Git** | `GitController` | `/api/v1/git` | `GitArtifactMetadata` (embedded), `GitDeployKeys` |
| **Theme** | `ThemeController` | `/api/v1/themes` | `Theme` |
| **Import / Export / Fork** | `RestApiImportController` + `imports/`, `exports/`, `fork/` packages | `/api/v1/import` | `ApplicationSnapshot` |
| **Custom JS libs** | `CustomJSLibController` | `/api/v1/libraries` | `CustomJSLib` |
| **Usage / telemetry** | `UsagePulseController`, `ProductFeatureAlertController` | `/api/v1/usage-pulse`, `/product-alert` | `UsagePulse` |
| **Infra / BFF** | `HealthCheckController`, `ConfigController`, `AssetController`, `SearchEntityController`, `InstanceAdminController`, **`ConsolidatedAPIController`** | `/api/v1/health`, `/configs`, `/assets`, `/admin`, **`/consolidated-api`** | `Asset`, `Config` |

### The one you should look at first: `ConsolidatedAPIController`

`/api/v1/consolidated-api` is a **hand-built BFF endpoint already in production**. One call for "open this page in the editor" fans out to ~10 collections in parallel (`services/ce/ConsolidatedAPIServiceCEImpl.java`, `getAllFetchableMonos`) and returns a single zipped payload — application, pages, theme, JS libs, actions, action collections, current user, feature flags, org config, product alerts.

This is direct precedent for the target's API Gateway/BFF composition layer. **Don't design it from scratch — generalise this.**

## 5. Cross-context orchestration: the `solutions/` package

46 files that deliberately reach *across* the contexts above. These are the seams that become sagas or gateway composition in the target — read them before designing service contracts.

| Class | What it spans | Becomes |
|---|---|---|
| `ActionExecutionSolution` | NewAction + Datasource + DatasourceStorage + Plugin + connection pool | Application Service → **Query Execution Service** (sync gRPC) |
| `ApplicationForkingService` | Application + Page + Action + ActionCollection + Theme + Datasource + CustomJSLib, into a target workspace | An **orchestrated saga** inside Application Service, with a compensating call to Datasource Service |
| `ImportService` / `ExportService` | Serialise/deserialise an entire application tree as one JSON | Same shape as Fork — internal orchestration, not a service |
| `PolicySolution` | Recomputes ACLs across Workspace → Application → Page → Action whenever a permission changes | **Event-driven authorization projections** ([Security & AuthZ](../02-target-architecture/06-security-and-authz.md)) |
| `UserAndAccessManagementService` | User + PermissionGroup + Workspace | Folds into **Identity & Access Service** |
| `CreateDBTablePageSolution` | Introspects a datasource schema and generates a whole CRUD page | Stays in Application Service; calls Query Execution for introspection |

## 6. Where each capability lives today

| Capability | Implementation | Notes |
|---|---|---|
| Authentication | Spring Security filter chain, `configurations/SecurityConfig.java` | Cookie session in Redis + CSRF token. Login is a **form POST**, not JSON |
| Authorization | Embedded `policyMap` on every document + `PermissionGroup` reverse index | Zero-hop checks. See [Permissions & ACL](04-permissions-and-acl.md) |
| Query execution | `solutions/ce/ActionExecutionSolutionCEImpl.java` (1,341 lines) | The hottest and riskiest path in the system |
| Connection pooling | `services/ce/DatasourceContextServiceCEImpl.java` | In-process map keyed by `(datasourceId, environmentId)` |
| Binding/dependency analysis | `services/ce/AstServiceCEImpl.java` → HTTP call to **RTS** `/rts-api/v1/ast/multiple-script-data` | The Java server cannot parse JS; it delegates to Node |
| On-load action ordering | `onload/internal/OnLoadExecutablesUtilCEImpl.java` | Builds a dependency DAG (JGraphT) from bindings, decides execution order |
| Git | `appsmith-git` (JGit) + `git/` package, Redis file-lock per artifact | Synchronous, in the request thread |
| Email | `notifications/EmailSender.java` | Fire-and-forget SMTP, **errors swallowed**, no retry |
| Analytics | `services/ce/AnalyticsServiceCEImpl.java` → Segment SDK | Third-party async queue |
| Scheduled jobs | Spring `@Scheduled` in-process, Redis `@DistributedLock` | No Quartz, no external scheduler, no queue |
| File/image storage | `Asset` domain — raw `byte[]` **stored in MongoDB** | Not GridFS, not S3, not disk |

## 7. What is notably absent

Naming these prevents wrong assumptions:

- **No message broker.** Zero Kafka/RabbitMQ/AMQP/SQS/Pulsar in any pom or source file. Everything is synchronous request/response or in-process reactive chaining. A broker is *net-new infrastructure* for this team.
- **No database transactions, essentially.** Exactly one unrelated `@Transactional` usage exists in the whole server module. Publish, fork, and cascade-delete are all multi-write sequences with **no atomicity guarantee** today.
- **No cross-collection joins.** An exhaustive search for `$lookup` / `LookupOperation` found two uses, both single-collection. All "joins" are parallel fetch-by-id + stitch in application code.
- **No plugin sandboxing.** 25 connectors run in the monolith's own JVM.
- **No comments/threads feature.** `COMMENT_ON_APPLICATIONS` permission exists but is `@Deprecated` and no `Comment` collection remains. Don't design for it.
- **No general domain-event bus.** Spring's `ApplicationEventPublisher` is used only for git autocommit housekeeping.

## 8. Reading list, in order

1. [Runtime Topology](02-runtime-topology.md) — what actually runs
2. [Domain Model & Database](03-domain-model-and-db.md) — the data
3. [Permissions & ACL](04-permissions-and-acl.md) — the hardest constraint
4. [Golden Paths](05-golden-paths.md) — how it all moves
5. [Plugin & Execution Engine](06-plugin-execution-engine.md) — the riskiest subsystem

---

[← Java Primer](../00-orientation/02-java-to-dotnet-primer.md) · [Next: Runtime Topology →](02-runtime-topology.md)
