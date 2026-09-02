# Delivery Tracking — Parallel Delivery & Dependency Management

[← Index](../README.md) · [← Epics, Features & User Stories](01-epics-features-stories.md)

---

How to run a large team against the [263-story backlog](01-epics-features-stories.md) without holding developers idle behind a service another squad hasn't finished. This doc is the rationale for the **Tier** tag on every feature in that backlog and every story row in [`02-jira-import.csv`](02-jira-import.csv) — read it before you start assigning work, not after.

The short version: almost every cross-service dependency in this system is a dependency on a *contract* — an endpoint shape or an event schema — not on a *finished service*. Freeze the contract early, mock it, and the downstream squad never waits. Only a handful of features are genuinely blocked until real, integrated behaviour exists.

---

## 1. The Tier model

Every feature in [Epics, Features & User Stories](01-epics-features-stories.md) is tagged with one of three tiers, derived directly from the [Service Dependency Matrix](../02-target-architecture/02-target-topology.md#3a-service-dependency-matrix).

| Tier | Meaning | When to assign |
|---|---|---|
| **0** | No cross-service call or event in or out. Pure schema, CRUD, or internal logic inside one service's own database. | Immediately. No limit on how many squads run Tier 0 work at once. |
| **1** | Depends on exactly one contract — one REST endpoint's request/response shape, or one event's payload shape — not on the upstream feature being finished. | Immediately, against a mock generated from the contract. The feature's first story is always a **contract-stub story** (§3) that freezes the contract in 1–3 points; everything else in the feature follows right behind it. |
| **2** | Depends on real, integrated behaviour: either an intra-service dependency (the thing it needs is in the *same* service, just a different feature) or a verified real-world property a mock can't stand in for (a saga's rollback path, a byte-identical diff, a live cutover rehearsal). | Only once the specific named dependency is actually Done. This is the only tier where "wait" is the correct instruction. |

Across the backlog: **144 Tier 0 stories, 100 Tier 1 stories (including 26 contract-stub stories), 19 Tier 2 stories.** The overwhelming majority of the backlog — Tier 0 plus the mock-driven part of Tier 1 — is parallelizable from day one, regardless of team size.

---

## 2. How to assign, concretely

1. **Staff every Tier 0 feature first, in parallel, across as many squads as you have.** There is no dependency-based reason to sequence any of it. It is large enough on its own to fully occupy a big team for the first stretch of the build — see the epic-by-epic breakdown in [Epics, Features & User Stories](01-epics-features-stories.md), where every feature not listed in §5 below is Tier 0.
2. **For every Tier 1 feature, the owning squad's first pull is the contract-stub story**, not the feature's "real" first story. That story is 1–3 points and Highest priority — it should be the fastest thing in that squad's queue.
3. **The moment a contract-stub story is Done, every squad consuming that contract can pull their dependent stories**, coded against the generated mock. They do not wait for the producing squad to finish the rest of that feature.
4. **A small follow-up story — swapping the mock for the real call — lands later**, once the producing squad's feature is actually done. This is the only place a "wait" shows up in a Tier 1 feature, and it's a few points, not the whole feature.
5. **Staff Tier 2 features last, and only the specific blocked story** — never hold an entire squad idle for a Tier 2 item elsewhere in their own epic while Tier 0/1 work in that same epic is sitting in `Backlog` unpulled. Kanban pull discipline handles this automatically if the board is set up per [`00-jira-setup-guide.md` §6](00-jira-setup-guide.md#6-suggested-workflow-kanban-board-per-epic): a squad's `Ready` column should never run empty while a Tier 2 item is stuck in `Blocked`.

---

## 3. The contract-stub story

Every Tier 1 feature in the CSV has a synthesized first story titled `Freeze and publish contract: <artifact>`, labelled `contract-stub,expedite`, at `Highest` priority, worth 2 points. Its acceptance criteria are the same for every one of the 26:

- The contract is published as an OpenAPI stub (for a REST endpoint) or a JSON Schema / event class definition (for a broker event) — not the full implementation behind it.
- A mock server or fixture payload is generated from that contract and shared with every consuming squad.
- Any breaking change to the contract after publishing pings every squad listed against that row in the [Service Dependency Matrix](../02-target-architecture/02-target-topology.md#3a-service-dependency-matrix) — this is the practical enforcement of [ADR-012](../03-execution/04-risks-and-adrs.md#adr-012--rest-for-every-synchronous-internal-call-no-grpc)'s point that REST gives no compile-time guarantee the way a schema compiler would.

Treat this as a house rule going forward, not just a backlog artifact: **any new cross-service dependency gets a contract-stub story before its consuming stories are estimated**, the same way a new dependency gets a new row in the Service Dependency Matrix before anything calls it.

---

## 4. The one real "everyone waits" dependency

`BuildingBlocks.*` (Abstractions, Observability, Messaging, Auth, Persistence — under `PLAT`) is not on the Tier list below, and deliberately so: it isn't a Tier 1 runtime contract, it's a **compile-time library dependency**. Every other service references these packages directly; nothing outside `PLAT` compiles without them. This is the one dependency in the whole programme where "wait" is unavoidable rather than a design gap — but it's a matter of days for the core packages (`Abstractions`, `Auth`, `Persistence`), not a phase. Staff `PLAT` first and heaviest at the very start, and treat its packages hitting the internal feed as the true "go" signal for every other squad's Tier 0 work, ahead of anything else in this doc.

---

## 5. Reference: every Tier 1 and Tier 2 feature

### Tier 1 — contract-stub unlocks these

| Epic | Feature | Matrix row(s) | Contract to freeze |
|---|---|---|---|
| PLAT | API Gateway Walking Skeleton | S1, S2 | Identity's Session Validation REST API request/response shape |
| APP | Authorization Projection | A1 | Identity's `PermissionGrantChanged` / `RoleAssignmentChanged` event schema |
| APP | Git Integration Surface | S14–S17 | Git Versioning's OpenAPI stub for `/internal/v1/git/commit\|push\|pull\|branches` |
| APP | AI Assistant Orchestration | S7 | Query Execution's OpenAPI stub for `POST /internal/v1/execution/assistant/generate` |
| DS | Plugin Catalog Replica | A7 | Query Execution's `PluginCatalogUpdated` event schema |
| DS | Authorization Projection | A1 | Identity's `PermissionGrantChanged` / `RoleAssignmentChanged` event schema |
| EXEC | SQL Connector Worker Pool | A6 | Datasource's `DatasourceConfigChanged` event schema |
| EXEC | HTTP Connector Worker Pool | A6 | Datasource's `DatasourceConfigChanged` event schema |
| EXEC | NoSQL / Cloud / AI Worker Pools | A6, A4 | Datasource's `DatasourceConfigChanged` + Identity's `AIAssistantConfigChanged` event schemas |
| EXEC | AI Assistant Provider Dispatch | A4 | Identity's `AIAssistantConfigChanged` event schema |
| EXEC | Connection Pooling & Introspection | A6 | Datasource's `DatasourceConfigChanged` event schema |
| GIT | Byte-Stable Serialization | — | Application Service's artifact JSON shape, frozen as a JSON Schema document (the one true data-shape dependency in the matrix — see note below) |
| RT | SignalR Hub & Auth | S18 | Identity's Session Validation REST API (same contract as S1) |
| RT | Version Push | A9, A10 | Application's `ApplicationPublished` + Git's `CommitCreated` event schemas |
| NOTIF | Event Consumers | A11 | Every epic's own event schema, adopted incrementally as each one freezes |
| NOTIF | Product Alerts | A11 | Same event catalog as Event Consumers |
| NOTIF | Usage & Audit | A11 | Same event catalog as Event Consumers |
| GW | Consolidated API Composition | S3–S5 | Application's and Datasource's OpenAPI stubs |
| NG | Published-App Viewer (C2) | S3, S4 | Application's and Datasource's OpenAPI stubs |
| NG | Editor IDE (C4) | S6, S8, S9, S14–S17 | Application's, Query Execution's and Git's OpenAPI stubs |
| NG | AI Assistant Client Surfaces | S7 | Query Execution's `assistant/generate` stub + Identity's AI config contract |
| MIG | Identity Migration | — | Identity Service's finalized `identity_db` DDL |
| MIG | Permission Grants Migration | — | Identity Service's finalized `permission_grants` DDL |
| MIG | Application Data Migration | — | Application Service's finalized `application_db` DDL |
| MIG | Secrets Migration | — | Datasource Service's finalized secrets-manager schema / `secret_ref` shape |
| MIG | Git Data Migration | — | Git Service's finalized `git_db` DDL |

**On `Byte-Stable Serialization` specifically:** this is the one row in the whole table where a mocked endpoint isn't enough — the Git squad's serializer needs the *exact* field-for-field shape of Application's artifact JSON, not just a plausible-looking sample. Freeze the JSON Schema document in week one and validate the serializer against it continuously as Application's export logic evolves, rather than treating the contract-stub as a one-time handoff. This is also the pairing to watch most closely for schema drift — see [Roadmap §9](../03-execution/02-roadmap-and-sequencing.md#9-risk-checkpoints), "Is git serialisation byte-identical to Java's?"

### Tier 2 — genuinely sequenced

| Epic | Feature | Blocked on |
|---|---|---|
| APP | Fork Saga | Datasource's real Clone/rollback endpoints — the compensation path (S13) needs a real clone to roll back, not a mock |
| APP | Import Saga | Same real clone/rollback dependency as Fork Saga |
| APP | Publish (Atomic) | Application's own Layout, Actions and Themes CRUD — intra-service, nothing to atomically commit until those exist |
| GIT | Auto-Commit | A working Application publish/save loop to trigger against meaningfully |
| GW | Contract Parity Verification | A live Java instance to response-diff against, plus a feature-complete .NET Gateway |
| NG | Hardening (C7) | A feature-complete real backend across every service — response-diffs against live Java by definition |
| MIG | Verification & Dry Runs | A feature-complete target system to dry-run against |
| MIG | Cutover Execution | A signed-off dry run and a feature-complete system — cutover is definitionally last |

---

## 6. Worked example

Application Service's **Authorization Projection** feature (Tier 1, depends on Identity's `A1` event) plays out like this:

1. Week 1: Identity squad pulls `Authorization Projection`'s contract-stub story (it's theirs to own, since they produce the event) — 2 points, publishes the `PermissionGrantChanged` / `RoleAssignmentChanged` schema and a fixture payload.
2. Same week, Application squad pulls the real `authz_grants` projection-table and consumer stories, coded and tested against that fixture. They are never idle waiting for Identity's full RBAC feature to ship.
3. Weeks later, once Identity's own Roles & Permission Grants feature is actually Done and publishing real events, Application squad pulls a small follow-up story — swap the fixture-driven consumer for the real broker subscription — a few points, not a re-build.
4. Datasource squad, which needs the identical event for its own Authorization Projection feature, does the same thing in parallel off the same contract-stub — no re-negotiation needed.

Three squads, one contract, zero idle time.

---

[← Epics, Features & User Stories](01-epics-features-stories.md) · [Index](../README.md)
