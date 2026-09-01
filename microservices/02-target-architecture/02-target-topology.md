# Target — Topology

[← Index](../README.md) · [← Service Inventory](01-service-inventory.md)

---

## 1. System context

```mermaid
C4Context
    title System context — Appsmith Platform (target)

    Person(builder, "App Builder", "Builds internal tools, writes queries, publishes apps")
    Person(enduser, "End User", "Uses published apps, may be anonymous")

    System(appsmith, "Appsmith Platform", "Gateway + 7 services + Angular client")

    System_Ext(idp, "OAuth / OIDC Provider", "Google, GitHub, SAML SSO")
    System_Ext(targets, "Customer Databases & APIs", "The systems users' apps actually query")
    System_Ext(email, "SMTP / Email Provider", "Transactional email")
    System_Ext(gitremote, "Git Remote", "GitHub, GitLab, Bitbucket")
    System_Ext(secrets, "Secrets Manager", "Datasource credentials, deploy keys")
    System_Ext(objstore, "Object Storage", "Images, logos, exports")

    Rel(builder, appsmith, "Builds, runs queries, publishes", "HTTPS / WebSocket")
    Rel(enduser, appsmith, "Uses published apps", "HTTPS")
    Rel(appsmith, idp, "Authenticates via", "OAuth2 / OIDC")
    Rel(appsmith, targets, "Executes user-defined queries against", "many protocols")
    Rel(appsmith, email, "Sends notifications via", "SMTP")
    Rel(appsmith, gitremote, "Commits and pushes app source to", "Git / SSH")
    Rel(appsmith, secrets, "Resolves credentials from", "HTTPS")
    Rel(appsmith, objstore, "Stores and serves assets", "S3 API")
```

## 2. Container diagram

```mermaid
C4Container
    title Container diagram — Appsmith Platform (target)

    Person(builder, "App Builder")

    System_Boundary(appsmith, "Appsmith Platform") {
        Container(spa, "Angular SPA", "Angular + TypeScript", "Editor and app viewer; hosts the evaluation Web Worker")
        Container(gateway, "API Gateway / BFF", ".NET 10, YARP", "Entry point, auth, rate limit, page-load composition")

        Container(identity, "Identity &amp; Access", ".NET 10", "AuthN, users, workspaces, RBAC truth")
        Container(application, "Application", ".NET 10", "Apps, pages, DSL, actions, themes, publish, fork/import/export")
        Container(datasource, "Datasource", ".NET 10", "Connection configs, plugin catalog replica")
        Container(execrouter, "Query Execution API", ".NET 10, gRPC", "Routes to connector workers, audits")
        Container(workers, "Connector Workers", ".NET 10, isolated processes", "SQL / NoSQL / HTTP / Cloud / AI pools + per-execution JS worker")
        Container(git, "Git Versioning", ".NET 10 + LibGit2Sharp", "Commit, push, pull, merge, deploy keys")
        Container(realtime, "Realtime Collaboration", ".NET 10, SignalR", "Presence, cursors, version push")
        Container(notify, "Notifications &amp; Telemetry", ".NET 10", "Email with retry + DLQ, alerts, usage")

        ContainerDb(identity_db, "identity_db", "PostgreSQL")
        ContainerDb(app_db, "application_db", "PostgreSQL")
        ContainerDb(ds_db, "datasource_db", "PostgreSQL")
        ContainerDb(exec_db, "execution_db", "PostgreSQL")
        ContainerDb(git_db, "git_db", "PostgreSQL")
        ContainerDb(notify_db, "notifications_db", "PostgreSQL")

        ContainerQueue(broker, "Message Broker", "RabbitMQ + MassTransit")
        ContainerDb(redis, "Redis", "Sessions, git lock, SignalR backplane, caches")
    }

    Rel(builder, spa, "Uses", "HTTPS")
    Rel(spa, gateway, "REST", "HTTPS, cookie session")
    Rel(spa, realtime, "Presence", "WebSocket")

    Rel(gateway, identity, "ValidateSession", "gRPC")
    Rel(gateway, application, "CRUD + composition", "gRPC")
    Rel(gateway, datasource, "CRUD", "gRPC")

    Rel(application, execrouter, "ExecuteAction", "gRPC sync")
    Rel(application, datasource, "GetDatasourceConfigs", "gRPC sync")
    Rel(application, git, "Commit / Push / Pull", "gRPC sync")
    Rel(datasource, execrouter, "TestConnection / GetStructure", "gRPC sync")
    Rel(execrouter, workers, "Dispatch", "IPC / gRPC")

    Rel(identity, broker, "Permission + membership events", "AMQP")
    Rel(datasource, broker, "DatasourceConfigChanged", "AMQP")
    Rel(application, broker, "Domain events", "AMQP")
    Rel(broker, application, "Permission events", "AMQP")
    Rel(broker, datasource, "Permission + catalog events", "AMQP")
    Rel(broker, execrouter, "Config changed", "AMQP")
    Rel(broker, notify, "All domain events", "AMQP")
    Rel(broker, realtime, "ApplicationPublished", "AMQP")

    Rel(identity, identity_db, "SQL")
    Rel(application, app_db, "SQL")
    Rel(datasource, ds_db, "SQL")
    Rel(execrouter, exec_db, "SQL")
    Rel(git, git_db, "SQL")
    Rel(notify, notify_db, "SQL")

    Rel(gateway, redis, "Sessions, rate limits")
    Rel(git, redis, "Lock by artifact id")
    Rel(realtime, redis, "Backplane")
```

## 3. Communication design

**Default is async.** Every synchronous edge below has to justify itself with either "a user is waiting for this result" or "an existing behaviour breaks if this becomes eventual."

| From → To | Interaction | Pattern | Why this and not the other |
|---|---|---|---|
| Angular → Gateway | Everything | Sync REST | Browser |
| Angular → Realtime | Presence | WebSocket (SignalR) | Push semantics |
| Gateway → Identity | Validate session | **Sync gRPC** | Every request needs an auth decision. Cached in Redis with a short TTL to keep this off the hot path |
| Gateway → Application/Datasource | CRUD + composition fan-out | **Sync gRPC**, parallel | Mirrors `ConsolidatedAPIController`. Per-slice error isolation preserved |
| Application → Query Execution | `ExecuteAction` | **Sync gRPC** | The user clicked Run and is watching a spinner. An async callback has no meaning here |
| Datasource → Query Execution | `TestConnection`, `GetStructure` | **Sync gRPC** | Interactive "Test" click; schema tree render |
| Application → Datasource | `GetDatasourceConfigs` (for git export) | **Sync gRPC** | Needs a point-in-time consistent snapshot at commit time. Low volume — only on commit, not on every read |
| Application → Datasource | Clone datasources during Fork | **Sync gRPC + compensation** | Part of the Fork saga; needs a definitive success/fail to decide commit or rollback |
| Application → Git | Commit / Push / Pull | **Sync gRPC** | User-triggered, needs immediate status |
| Identity → everyone | Permission / membership changed | **Async event** | Avoids a network hop on the *most frequent operation in the system*. Each service keeps a local authorization projection ([D3](../00-orientation/01-executive-summary.md#d3--authorization-is-replicated-not-called)) |
| Datasource → Execution | `DatasourceConfigChanged` | **Async event** | Keeps the execution hot path local, mirroring today's in-process cache |
| Execution → Datasource | `PluginCatalogUpdated` | **Async event** | Rare, and never blocks a user action |
| Everyone → Notifications | Domain events | **Async event** | Must never add latency or a failure mode to the originating flow |
| Application → Realtime | `ApplicationPublished` | **Async event** | Notifying connected clients must not block publish |

### Rules for the synchronous graph

1. **No cycles.** If service A calls B synchronously, B never calls A synchronously. Currently satisfied.
2. **Depth ≤ 2** from the gateway. Gateway → Application → Execution is the deepest chain.
3. **Every sync call has a timeout, a retry policy, and a circuit breaker** (Polly / `Microsoft.Extensions.Resilience`).
4. **Every sync call degrades explicitly.** If Datasource Service is down, the editor still opens — the datasource panel shows an error. Per-slice degradation, exactly as the current consolidated API does it.

## 4. Deployment shape

Two supported profiles, one architecture.

### 4.1 Self-hosted / single-node (matches today's ethos)

```mermaid
flowchart TB
    subgraph HOST["One host — docker compose"]
        NGX[Caddy/nginx + Angular static]
        GW[Gateway]
        SVC["7 service containers"]
        PG[(PostgreSQL<br/>6 databases, one cluster)]
        RD[(Redis)]
        MQ[(RabbitMQ)]
        MIN[(MinIO — object storage)]
    end
    NGX --> GW --> SVC
    SVC --> PG & RD & MQ & MIN
```

One Postgres cluster with **six logically isolated databases**. No service ever has credentials for another's database. Physically consolidating them is a *deployment* choice, not an architectural compromise.

### 4.2 Cloud / Kubernetes

- Each service a Deployment with its own HPA. Connector worker pools scale independently.
- Managed Postgres (separate instances or one instance with separate databases + separate roles), managed Redis, managed AMQP.
- Git Service needs a `ReadWriteMany` volume or per-pod clone-on-demand.
- Angular build on a CDN.

### 4.3 Local development

**.NET Aspire** as the developer inner loop: `dotnet run` on the AppHost project starts all services, Postgres, Redis and RabbitMQ in containers, wires connection strings and service discovery automatically, and gives a dashboard with traces and logs across all eight services. This replaces the current "one giant container with supervisord" developer experience and is a meaningful quality-of-life improvement — see [.NET 10 Standards §9](04-dotnet-10-standards.md).

## 5. Resilience posture

| Failure | Behaviour |
|---|---|
| Identity & Access down | Gateway serves requests with cached session validations until TTL expiry, then 503. No new logins |
| Application Service down | Editor and viewer unavailable. Everything else fine |
| Datasource Service down | Editor opens; the datasource panel degrades. **Query execution still works** (Execution has a local config replica) |
| Query Execution down | Editing works; running queries fails with a clear error. Published apps degrade to static |
| A single connector worker pool down | Only that connector family fails. Others unaffected — this is the point of the split |
| Git Service down | Git operations fail; everything else fine |
| Realtime down | Presence indicators vanish. No functional impact |
| Notifications down | Events queue in the broker. Nothing user-facing breaks |
| Broker down | Sync flows all work. Projections go stale — bounded and monitored. Outbox retains unpublished events |
| Redis down | Sessions lost (users re-login), git operations blocked (lock unavailable), presence degraded |

**Read that table as the design's thesis:** in the current monolith, every one of those rows is "the whole platform is down."

## 6. Observability

- **OpenTelemetry throughout** — already the standard in the current codebase, carried forward rather than replaced.
- **Traces** propagate across gRPC metadata *and* broker message headers, so a Fork saga is one trace.
- **Correlation id** minted at the gateway, stamped on every log line and every message.
- **Metrics that matter**: per-connector execution latency and error rate (new — invisible today), projection lag per service, saga completion/compensation counts, broker queue depth and DLQ size, gateway composition slice failure rate.
- **The gateway's `responseMeta` envelope is preserved**, so client error handling doesn't change.

---

[← Service Inventory](01-service-inventory.md) · [Next: Database per Service →](03-database-per-service.md)
