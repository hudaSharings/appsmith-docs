# Reference — Glossary

[← Index](../README.md) · [← Risks & ADRs](../03-execution/04-risks-and-adrs.md)

---

Appsmith's vocabulary, with the Java class it maps to and the target-side equivalent. When a term in these docs is ambiguous, this page is authoritative.

---

## Product concepts

| Term | Means | Java class / collection | Target |
|---|---|---|---|
| **Action** | A query or API call the user authored. "Run `GetUsers`" runs an action | `NewAction` (the `New` is historical — it replaced an older `Action` collection) | `actions` table, Application Service |
| **Action Collection** | A **JS Object** — a bundle of user-written JavaScript functions usable from bindings | `ActionCollection` | `action_collections` |
| **Anvil** | The newest of three layout systems (section/zone based) | `src/layoutSystems/anvil` | Scope decision Q1 |
| **Artifact** | The abstraction over "a thing that can be git-versioned, exported and imported." Today only Application implements it | `Artifact` interface | Same concept, used by Git Service's contract |
| **Auto-layout** | The responsive flex-based layout system | `src/layoutSystems/autolayout` | Ported |
| **Auto-commit** | Background git commit after Appsmith upgrades a DSL/JSON schema version, so the user's next manual commit isn't a huge diff | `git/autocommit/` | Git Service |
| **Binding** | A `{{ expression }}` inside a widget property or event handler. Evaluated client-side; parsed server-side to compute the on-load plan | `MustacheHelper`, `AstService` | Unchanged — a frozen contract |
| **Branched id** | The per-branch id of an entity. Every API parameter named `branchedApplicationId`, `branchedPageId` uses it | `RefAwareDomain.id` | `id` column |
| **Base id** | The stable identity of "the same entity" across git branches | `RefAwareDomain.baseId` | `base_id` column |
| **Canvas** | The visual editing surface. Serialised as the DSL | `Layout.dsl` | `pages.draft_layout` (jsonb) |
| **Consolidated API** | The single "boot" endpoint that returns everything needed to open an editor or viewer. Already a BFF | `ConsolidatedAPIController` | Gateway composition endpoint |
| **Datasource** | A connection *identity* — name + which plugin + workspace. Holds no credentials | `Datasource` | `datasources` table |
| **Datasource Storage** | The actual configuration for a datasource **in one environment**, including secrets | `DatasourceStorage` | `datasource_storages` |
| **Derived property** | A widget property computed from others (e.g. `Table1.selectedRow`). Defined as data in the widget config | `derivedProperties` in widget config | Ported unchanged |
| **DSL** | *Domain-Specific Language* — the JSON tree describing a page's widgets. **The product's core data structure** | `Layout.dsl` (a `JSONObject`) | `jsonb`. Frozen across the rewrite |
| **Edit mode / View mode** | Draft vs published. Stamped through the whole model as `unpublishedX` / `publishedX` | `ApplicationMode` | `draft_*` / `published_*` columns |
| **Environment** | A named configuration slot for a datasource (dev/staging/prod). CE only ever has one | `environmentId` on `DatasourceStorage` | `environments` table |
| **Evaluation version** | Which semantics the binding evaluator uses. Bumping it changes what expressions mean, so it's pinned per application | `Application.evaluationVersion` | Same column |
| **Fixed layout** | The original absolute grid layout system (rows/columns) | `src/layoutSystems/fixedlayout` | Ported |
| **Fork** | Copy an application into another workspace, remapping every internal id | `ApplicationForkingService` | A saga inside Application Service |
| **Instance** | One self-hosted Appsmith deployment | `Tenant` + `Organization` (two near-duplicate collections) | One `instances` row |
| **JS Object** | See **Action Collection** | | |
| **Layout** | The container for a page's DSL plus its computed on-load plan. **Not its own collection** — embedded in the page | `Layout` inside `PageDTO` | Columns on `pages` |
| **Mock datasource** | Sample datasources backed by the embedded Postgres, used in tutorials | `MockDataService` | Same, in Datasource Service |
| **On-load actions** | The ordered plan of which queries run when a page opens. A list of *waves*; actions within a wave run in parallel | `Layout.layoutOnLoadActions` — `List<Set<DslExecutableDTO>>` | `pages.draft_onload_plan`. A frozen client contract |
| **Page** | One screen in an application. Owns a DSL | `NewPage` (again, `New` is historical) | `pages` table |
| **Permission Group** | A **role**. Holds its own member list | `PermissionGroup` | `roles` + `role_assignments` |
| **Plugin** | A **connector** — Postgres, REST, S3, OpenAI. 25 of them | `Plugin` + a module under `appsmith-plugins/` | `plugins` in Execution Service |
| **Policy** | `{permission → [permissionGroupIds]}`, embedded on **every** document. The inline ACL | `Policy` in `BaseDomain.policyMap` | `permission_grants` (truth) + `authz_grants` (projections) |
| **Publish / Deploy** | Copy draft → published for every entity in an application | `ApplicationPageService.publish` | One transaction |
| **Refactor** | Rename an entity and rewrite every binding that references it | `refactors/` package | Application Service |
| **Snapshot** | A coarse point-in-time backup of an application, taken before risky operations. **Not** a version history | `ApplicationSnapshot` | `application_snapshots` |
| **Static URL** | A custom human-readable slug for a published application | `StaticUrlSettings` | `static_urls` table |
| **Theme** | Styling applied to an application. System themes plus per-app customisations | `Theme` | `themes` table |
| **Trigger** | A plugin call that populates a dynamic dropdown in the editor ("list the sheets in this spreadsheet") | `PluginExecutor.trigger` | Execution Service REST endpoint |
| **Usage Pulse** | An anonymous activity beacon used to compute DAU/WAU/MAU | `UsagePulse` | `usage_events` |
| **Widget** | A UI component on the canvas. 82 types | `src/widgets/<Name>` | Angular components + ported config |
| **Workspace** | The tenancy and permission boundary. Contains applications and datasources | `Workspace` | `workspaces` table |

---

## Technical terms in the Java codebase

| Term | Means |
|---|---|
| **`CE` / `CEImpl` suffix** | Community Edition. `FooServiceImpl extends FooServiceCEImpl` is a **licensing build-time seam**, not a design pattern. Read the `CE` class — it has the logic. **Not reproduced in the target** |
| **`Solution`** | A class that deliberately spans bounded contexts: `ActionExecutionSolution`, `PolicySolution`. These are the seams that become sagas or gateway composition |
| **`Mono<T>` / `Flux<T>`** | Project Reactor. `Task<T>` and `IAsyncEnumerable<T>`. See the [Java Primer](../00-orientation/02-java-to-dotnet-primer.md) |
| **PF4J** | Plugin Framework for Java. Loads connectors into separate classloaders **in the same JVM** — no process isolation |
| **Mongock** | MongoDB migration framework. 83 changelog classes; **a primary source of undocumented behaviour** |
| **Lombok** | Annotation processor that generates getters, constructors, builders and loggers. If a class "has no getters," it has getters |
| **`@Document`** | Spring Data Mongo: this class is a collection |
| **`@Encrypted`** | Field encrypted at rest by a Mongo event listener |
| **`@JsonView`** | Jackson: which fields serialise in which context (public API / internal / git export). The target uses separate DTOs per view instead |
| **`BaseDomain`** | The base class on every persisted entity: id, audit fields, `deletedAt`, `policyMap` |
| **`RefAwareDomain`** | Adds git-branch awareness: `baseId`, `refName`, `refType` |
| **RTS** | *Realtime Service.* The Node/Express/Socket.IO process. Does AST parsing, DSL migration, some git ops, and presence sockets |
| **`ExecuteActionDTO`** | The request shape for running a query (multipart, because params can carry blobs) |
| **`ActionExecutionResult`** | The result of running a query. **Never persisted today** |
| **`ArtifactExchangeJson`** | The serialised form of an entire application — used by export, import, fork and git |
| **`sanitiseToExportDBObject()`** | Strips ids, policies, workspace ids and secrets before export |
| **`policyMap` vs `policies`** | Two representations of the same ACL; `policies` (a `Set`) is deprecated but still populated for git-diff stability |
| **`DatasourceContext`** | A pooled connection, cached in-process by `(datasourceId, environmentId)` |
| **`invalids`** | Validation errors on a datasource configuration, produced by the plugin |

---

## Terms introduced by the target architecture

| Term | Means |
|---|---|
| **BFF** | Backend for Frontend. The gateway's composition endpoints — one client call fanning out to several services |
| **`authz_grants`** | The local, event-fed authorization projection in each resource-owning service. Joined in every query |
| **`permission_grants`** | The authoritative grant table in `identity_db` |
| **Authorization projection** | A service's local replica of "which roles may do what to my resources" |
| **Transactional outbox** | Events written in the same database transaction as the state change, published later by a dispatcher. Prevents "committed but never published" |
| **Inbox** | A consumed-message-id table making consumers idempotent under at-least-once delivery |
| **Saga** | A multi-service operation with compensating actions. **Exactly three exist**: Fork, Import, Workspace deletion |
| **Connector worker** | An isolated process running one family of connectors, with CPU/memory/time limits |
| **Worker pool** | A group of connector workers sharing a risk profile (SQL, HTTP, Cloud, AI, NoSQL). The **JS worker is not pooled** — one per execution |
| **Internal JWT** | A ~60-second token minted by the gateway per request, carrying user id and role ids to downstream services. Services never see the browser cookie |
| **`secret_ref`** | A pointer into the secrets manager. No service database holds secret material |
| **Projection lag** | Time between an event being published and a consumer's projection reflecting it. Budget: p99 < 5s for authorization |
| **Golden files** | Recorded request/response pairs captured from the Java connectors, used as the .NET connector test suite |
| **Application corpus** | A set of real page DSLs exported as fixtures, driving the evaluation-port harness, visual regression and widget prioritisation |

---

## Naming collisions to watch for

| Term | Which meaning where |
|---|---|
| **Application** | (a) an Appsmith app a user builds; (b) the .NET **Application Service**; (c) the Clean Architecture **Application layer**. Context disambiguates; these docs say "Application Service" and "Application layer" explicitly |
| **Action** | An Appsmith query — **not** a Redux action, and **not** a C# `Action` delegate |
| **Environment** | A datasource config slot (dev/staging/prod for a *connection*) — **not** a deployment environment |
| **Organization / Tenant** | Two legacy collections in the Java code, collapsed to `Instance` in the target. If you see both, you're reading current-state docs |
| **Plugin** | An Appsmith connector — **not** a .NET plugin assembly or an Angular plugin |
| **Policy** | The Appsmith inline ACL entry — **not** an ASP.NET authorization policy |
| **New*** (`NewAction`, `NewPage`) | Historical names. These *are* the Action and Page entities; there is no "old" one |

---

[← Risks & ADRs](../03-execution/04-risks-and-adrs.md) · [Index](../README.md)
