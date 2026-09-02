# Target — .NET 10 Standards & Conventions

[← Index](../README.md) · [← Database per Service](03-database-per-service.md)

---

The opinionated stack. These are decisions, not options — consistency across eight services is worth more than each team's preferred variation. Deviations need an ADR ([Risks & ADRs](../03-execution/04-risks-and-adrs.md)).

---

## 1. Solution layout

Mono-repo, matching the current repo's single-repo shape. One solution file per service plus a shared building-blocks solution.

```
/appsmith-next
├── Directory.Build.props            central LangVersion, Nullable, TreatWarningsAsErrors
├── Directory.Packages.props         CENTRAL PACKAGE MANAGEMENT — one version per package, repo-wide
├── global.json                      pinned SDK
├── Appsmith.slnx
│
├── apphost/
│   └── Appsmith.AppHost/            .NET Aspire — the dev inner loop
│
├── gateway/
│   └── Appsmith.Gateway/            YARP + composition endpoints
│
├── services/
│   ├── identity/
│   │   ├── Appsmith.Identity.Api/              host, REST endpoints
│   │   ├── Appsmith.Identity.Application/      use cases
│   │   ├── Appsmith.Identity.Domain/           entities, value objects, domain events
│   │   ├── Appsmith.Identity.Infrastructure/   EF Core, outbox, publishers
│   │   ├── Appsmith.Identity.Contracts/        public DTOs + integration events (referenced by others)
│   │   └── tests/
│   │       ├── Appsmith.Identity.UnitTests/
│   │       └── Appsmith.Identity.IntegrationTests/
│   ├── application/    (same five projects)
│   ├── datasource/     (same five projects)
│   ├── execution/
│   │   ├── Appsmith.Execution.Api/             REST router
│   │   ├── Appsmith.Execution.Workers.Sql/
│   │   ├── Appsmith.Execution.Workers.NoSql/
│   │   ├── Appsmith.Execution.Workers.Http/
│   │   ├── Appsmith.Execution.Workers.Cloud/
│   │   ├── Appsmith.Execution.Workers.Ai/
│   │   ├── Appsmith.Execution.Workers.Js/
│   │   ├── Appsmith.Execution.Connectors.Abstractions/   IConnectorExecutor<T>
│   │   └── … Application / Domain / Infrastructure / Contracts / tests
│   ├── git/            (same five projects)
│   ├── realtime/
│   │   └── Appsmith.Realtime.Api/              SignalR hubs only — no Domain layer needed
│   └── notifications/  (same five projects)
│
├── shared/
│   ├── Appsmith.BuildingBlocks.Abstractions/   IClock, IUserContext, Result<T>, base types
│   ├── Appsmith.BuildingBlocks.Observability/  OTel wiring, correlation ids
│   ├── Appsmith.BuildingBlocks.Messaging/      MassTransit + outbox/inbox conventions
│   ├── Appsmith.BuildingBlocks.Auth/           session/token validation, IUserContext population
│   ├── Appsmith.BuildingBlocks.Authorization/  authz_grants projection + the permission graph
│   └── Appsmith.BuildingBlocks.Persistence/    EF Core conventions, soft delete, audit interceptors
│
└── client/
    └── appsmith-web/                           Angular workspace
```

### Layer rules

Enforced by an architecture test (`NetArchTest` / `ArchUnitNET`) in every service's unit test project:

| Layer | May reference | Must not reference |
|---|---|---|
| `Domain` | `BuildingBlocks.Abstractions` only | EF Core, ASP.NET, MassTransit, any other service |
| `Application` | `Domain`, `Contracts`, abstractions | EF Core, ASP.NET, concrete infrastructure |
| `Infrastructure` | `Domain`, `Application`, everything | — |
| `Api` | `Application`, `Infrastructure` (composition root only), `Contracts` | `Domain` internals directly |
| `Contracts` | Nothing | Everything — it must stay a leaf so other services can reference it |

**Why Clean Architecture here and not just a flat service?** Because the domain layer genuinely earns it: publish semantics, fork/import id-remapping, the permission cascade graph, and the on-load dependency ordering are real logic worth testing without a database. That is not true of every project; it is true of this one.

**Realtime is the exception** — it has no domain model, so it stays a single project. Don't create empty layers for symmetry.

---

## 2. Language & compiler settings

`Directory.Build.props`:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <InvariantGlobalization>true</InvariantGlobalization>
    <PublishAot>false</PublishAot>
  </PropertyGroup>
</Project>
```

Conventions:
- **Nullable reference types on, warnings as errors.** Non-negotiable — the Java code's `Optional`/null handling is inconsistent and we are not porting that.
- **`record` for DTOs, events and value objects; `class` for entities** with private setters and behaviour.
- **Primary constructors** for DI (the direct analogue of Lombok's `@RequiredArgsConstructor`).
- **`file`-scoped namespaces**, `var` where the type is obvious.
- **No `async void`.** Ever. **No `.Result` / `.Wait()`.** Ever.
- **`CancellationToken` threaded through every async method** to the database and every outbound call.
- **`TimeProvider`** injected instead of `DateTime.UtcNow`, so time is testable.

`.editorconfig` is committed and enforced; formatting is checked in CI (`dotnet format --verify-no-changes`).

---

## 3. API layer — Minimal APIs

Use **Minimal APIs with endpoint groups**, not MVC controllers. Reason: the shape of this API is verb-per-resource with fine-grained authorization, which suits explicit endpoint registration; and it keeps the request pipeline visible rather than buried in attributes.

```csharp
// Appsmith.Application.Api/Endpoints/PageEndpoints.cs
public static class PageEndpoints
{
    public static RouteGroupBuilder MapPageEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/v1/pages")
                       .WithTags("Pages")
                       .RequireAuthorization();

        group.MapGet("/{pageId:guid}", GetPage)
             .WithName("GetPage")
             .Produces<ApiResponse<PageDto>>();

        group.MapPut("/{pageId:guid}", UpdatePage)
             .WithName("UpdatePage");

        return group;
    }

    private static async Task<IResult> GetPage(
        Guid pageId,
        [AsParameters] PageQueryOptions options,
        IPageQueries queries,
        IUserContext user,
        CancellationToken ct)
    {
        var page = await queries.GetAsync(pageId, options.ViewMode, user, ct);
        return page is null ? ApiResults.NotFound() : ApiResults.Ok(page);
    }
}
```

Rules:
- One static class per resource, one file, registered from `Program.cs`.
- Handlers are static methods with explicit dependencies — trivially unit-testable, no controller base class.
- **Validation with FluentValidation**, wired as an endpoint filter, returning RFC 7807 `ProblemDetails` *inside* the legacy envelope (§4).
- **OpenAPI via `Microsoft.AspNetCore.OpenApi`**, one document per service, aggregated at the gateway.
- **Typed results** (`Results<Ok<T>, NotFound, ValidationProblem>`) so the OpenAPI document is accurate without attributes.

### 3.1 Preserve the response envelope

The Angular client (and the current React client) unwraps `.data` on every call. Keep the envelope — it costs nothing and removes a whole class of migration bug:

```jsonc
{ "responseMeta": { "status": 200, "success": true, "error": null }, "data": { } }
```

Implement it once as an endpoint filter in `BuildingBlocks`, not per endpoint. Same for the deliberate **404-instead-of-403** behaviour ([Permissions §2](../01-current-system/04-permissions-and-acl.md#2-how-a-permission-check-actually-happens)) — that lives in the authorization filter, not in each handler.

---

## 4. Internal communication — REST

- **REST/JSON for every synchronous call in the system, edge and internal alike.** No second protocol stack: the same `HttpClient`/Minimal-API story the Gateway needs for the browser is reused for service-to-service calls, under an `/internal/v1/...` prefix each service exposes alongside its public `/api/v1/...` surface. Full endpoint-by-endpoint list: [Service Contracts & Events §2](05-service-contracts-and-events.md#2-synchronous-contracts-rest).
- **Typed clients, generated, not hand-written.** Each service publishes an OpenAPI 3.1 document (`Microsoft.AspNetCore.OpenApi`) at `/internal/v1/openapi.json`; callers generate a typed C# client from it into their own `*.Contracts` project (`Microsoft.Extensions.ApiDescription.Client` / Kiota) at build time. This is what recovers protobuf's compile-time-checked request/response types without adopting protobuf.
- **`IHttpClientFactory` + named typed clients**, one per downstream service, registered with **Polly resilience handlers** (`Microsoft.Extensions.Http.Resilience`): timeout, retry with jitter on idempotent (`GET`) calls only, circuit breaker.
- **A timeout on every call, set explicitly.** REST has no built-in deadline propagation the way a gRPC call does — a call without an explicit `HttpClient.Timeout` (or a per-request `CancellationToken` from `CancellationTokenSource.CancelAfter`) is a bug.
- **Correlation id and internal auth travel as ordinary HTTP headers** — `X-Correlation-Id` and `Authorization: Bearer <internal JWT>` — attached by a shared `DelegatingHandler` in `BuildingBlocks.Observability`/`BuildingBlocks.Auth`, not protocol-level metadata.
- **Internal endpoints are network-isolated from the public surface**, not merely path-prefixed: the Gateway's YARP routing table only ever forwards `/api/v1/...`; `/internal/v1/...` is reachable service-to-service only, enforced by network policy. This is what keeps "REST for both edge and internal" from becoming "the internal API is accidentally public."

**Why REST and not gRPC**, given gRPC's genuine advantages here (binary transport, native deadlines, protobuf-enforced contracts): see [ADR-012](../03-execution/04-risks-and-adrs.md) and [Service Contracts & Events §1](05-service-contracts-and-events.md#1-contract-rules). Short version — one HTTP stack instead of two, ordinary tooling for debugging internal calls, and a YARP gateway that can proxy pass-through CRUD without protocol translation; the lost compile-time contract safety is recovered via generated clients and CI-diffed OpenAPI documents instead of a schema compiler.

```csharp
// Appsmith.Application.Infrastructure — a typed client calling Query Execution
public sealed class ExecutionApiClient(HttpClient http)
{
    public async Task<ActionExecutionResult> ExecuteActionAsync(
        ExecuteActionRequest request, CancellationToken ct)
    {
        using var response = await http.PostAsJsonAsync("/internal/v1/execution/actions/execute", request, ct);
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<ActionExecutionResult>(ct))!;
    }
}

// Program.cs
services.AddHttpClient<ExecutionApiClient>(c =>
    {
        c.BaseAddress = new Uri(config["Services:Execution:BaseUrl"]!);
        c.Timeout = TimeSpan.FromSeconds(30);
    })
    .AddHttpMessageHandler<InternalAuthHandler>()      // attaches the internal JWT
    .AddHttpMessageHandler<CorrelationIdHandler>()
    .AddStandardResilienceHandler();                    // Polly: retry, circuit breaker, timeout
```

---

## 5. Persistence — EF Core 10 + Npgsql

- **EF Core**, code-first, with `Npgsql.EntityFrameworkCore.PostgreSQL`.
- **One `DbContext` per service.** No shared context, no cross-service entity types.
- **Migrations in the `Infrastructure` project**, applied by a dedicated migration job at deploy time — **never** `Database.Migrate()` on service startup (eight replicas racing to migrate is a real outage).
- `IEntityTypeConfiguration<T>` per entity, never fluent config inline in `OnModelCreating`.
- **Compiled queries** for the hot paths (editor boot, action lookup, authorization filter).
- **`AsNoTracking()` by default for reads**; set `QueryTrackingBehavior.NoTrackingWithIdentityResolution` on the context and opt into tracking explicitly.
- **`jsonb` columns** mapped as `JsonDocument` or a POCO with `.HasColumnType("jsonb")` — see [Database per Service §2](03-database-per-service.md#2-translation-rules-mongodb--postgresql) for when to use it.
- **Soft delete via a global query filter** plus a `SaveChanges` interceptor that converts deletes to `deleted_at` writes.
- **Audit columns via a `SaveChanges` interceptor** reading `IUserContext` and `TimeProvider`.
- **Dapper is allowed** for genuinely complex read queries (the composition endpoints, the audit rollups). Don't fight EF Core to avoid it.

```csharp
public sealed class ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
    : DbContext(options)
{
    public DbSet<Domain.Application> Applications => Set<Domain.Application>();
    public DbSet<Page> Pages => Set<Page>();
    public DbSet<AppAction> Actions => Set<AppAction>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.HasDefaultSchema("app");
        b.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
        b.AddTransactionalOutboxEntities();   // MassTransit
        SoftDeleteConvention.Apply(b);        // BuildingBlocks.Persistence
    }
}
```

---

## 6. Messaging — MassTransit + RabbitMQ

**Why RabbitMQ and not Kafka.** Appsmith's deployment ethos is self-hosted and single-footprint — today it is literally one container plus an external Redis. RabbitMQ keeps that operational profile; Kafka does not. Nothing in the event model needs log replay, partitioned ordering at scale, or stream processing. If that changes, MassTransit's transport abstraction makes the swap contained.

**Non-negotiable patterns:**

| Pattern | Why |
|---|---|
| **Transactional outbox** (`AddEntityFrameworkOutbox`) | Without it a service can commit state and crash before publishing, permanently desyncing every projection |
| **Inbox / idempotent consumers** | The broker delivers at-least-once. Every consumer must tolerate duplicates |
| **Retry + circuit breaker + DLQ** on every consumer | The current system's email path fails silently. Never again |
| **Versioned event contracts** in `*.Contracts` | Additive-only changes; a breaking change is a new `V2` message type consumed alongside `V1` |
| **Correlation id on every message** | One trace across a saga |

```csharp
services.AddMassTransit(x =>
{
    x.SetKebabCaseEndpointNameFormatter();
    x.AddConsumers(typeof(PermissionGrantChangedConsumer).Assembly);

    x.AddEntityFrameworkOutbox<ApplicationDbContext>(o =>
    {
        o.UsePostgres();
        o.UseBusOutbox();
        o.QueryDelay = TimeSpan.FromSeconds(1);
    });

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(config.GetConnectionString("broker"));
        cfg.UseMessageRetry(r => r.Exponential(5,
            TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(2), TimeSpan.FromSeconds(2)));
        cfg.UseDelayedRedelivery(r => r.Intervals(
            TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(30), TimeSpan.FromHours(2)));
        cfg.ConfigureEndpoints(ctx);
    });
});
```

**Sagas.** Use MassTransit state machines (`MassTransitStateMachine<T>`) for the three flows that need one — Fork, Import, Workspace deletion cascade ([Target Golden Paths](07-target-golden-paths.md)). Persist saga state in the orchestrating service's own database. Do **not** use a saga where a local transaction suffices; publish is deliberately not a saga.

---

## 7. Authentication & authorization

- **Cookie session at the edge** ([D4](../00-orientation/01-executive-summary.md#d4--keep-the-cookie-session-at-the-edge)). The gateway validates the cookie against Identity & Access (Redis-cached) and mints a **short-lived internal JWT** carrying `userId`, `instanceId`, `roleIds[]` and `correlationId` for downstream calls.
- Services trust only the internal token, validated with a shared key/JWKS. They never see the browser cookie.
- `IUserContext` is populated from that token by shared middleware in `BuildingBlocks.Auth` and injected everywhere. **No service reads `HttpContext` directly.**
- **Resource authorization is a query predicate, not a policy attribute** — the `authz_grants` join from [Database per Service §4](03-database-per-service.md#4-authorization-how-policymap-becomes-a-projection). ASP.NET authorization policies handle only coarse checks (authenticated, instance-admin).

Full design: [Security & AuthZ](06-security-and-authz.md).

---

## 8. Observability

Standardise on **OpenTelemetry**, which is already the standard in the Java system.

```csharp
builder.Services.AddAppsmithObservability(builder.Configuration);   // BuildingBlocks.Observability
// → OTLP exporter, ASP.NET Core + HttpClient + Npgsql + MassTransit instrumentation,
//   resource attributes (service.name/version/instance), correlation-id enrichment,
//   Serilog structured logging to stdout, /health/live + /health/ready
```

- **One activity source per service**, named for it.
- **Custom spans** on the flows that matter: action execution (tagged with connector and worker pool), editor composition (per slice), git operations, saga steps.
- **Metrics**: per-connector latency/error rate, projection lag, saga outcomes, broker queue depth and DLQ size, composition slice failure rate.
- **Health checks**: `/health/live` (process alive) and `/health/ready` (database, broker, and critical dependencies). Kubernetes probes map to these.

---

## 9. Local development — .NET Aspire

`Appsmith.AppHost` is the developer entry point. `dotnet run` starts everything.

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("postgres").WithDataVolume().WithPgAdmin();
var identityDb    = postgres.AddDatabase("identity-db");
var applicationDb = postgres.AddDatabase("application-db");
var datasourceDb  = postgres.AddDatabase("datasource-db");
var executionDb   = postgres.AddDatabase("execution-db");
var gitDb         = postgres.AddDatabase("git-db");
var notifyDb      = postgres.AddDatabase("notifications-db");

var redis  = builder.AddRedis("redis");
var broker = builder.AddRabbitMQ("broker").WithManagementPlugin();

var identity = builder.AddProject<Projects.Appsmith_Identity_Api>("identity")
                      .WithReference(identityDb).WithReference(redis).WithReference(broker);

var execution = builder.AddProject<Projects.Appsmith_Execution_Api>("execution")
                       .WithReference(executionDb).WithReference(broker);

var datasource = builder.AddProject<Projects.Appsmith_Datasource_Api>("datasource")
                        .WithReference(datasourceDb).WithReference(broker).WithReference(execution);

var application = builder.AddProject<Projects.Appsmith_Application_Api>("application")
                         .WithReference(applicationDb).WithReference(broker)
                         .WithReference(execution).WithReference(datasource);

builder.AddProject<Projects.Appsmith_Gateway>("gateway")
       .WithReference(identity).WithReference(application).WithReference(datasource)
       .WithReference(redis).WithExternalHttpEndpoints();

builder.AddNpmApp("web", "../client/appsmith-web", "start")
       .WithHttpEndpoint(env: "PORT").WithExternalHttpEndpoints();

builder.Build().Run();
```

This gives every developer a one-command environment with a dashboard showing distributed traces across all eight services — a large improvement over today's "one container, supervisord, embedded Mongo" loop.

**Aspire is for development and orchestration manifests. Production deploys from the same container images via Kubernetes or compose**, not from the AppHost.

---

## 10. Testing

| Level | Tooling | Rule |
|---|---|---|
| **Unit** | xUnit v3, NSubstitute, Shouldly | Domain and Application layers. No database, no HTTP. Fast |
| **Architecture** | NetArchTest | Enforces the layer rules in §1. Fails the build |
| **Integration** | `WebApplicationFactory` + **Testcontainers** (Postgres, Redis, RabbitMQ) | Real database, real broker. One per service, covering endpoints → database |
| **Contract** | `WebApplicationFactory` against each service's live OpenAPI document, diffed with `oasdiff` against the previous release | Detects breaking contract changes before consumers do |
| **Connector golden files** | Recorded request/response pairs captured from the **Java plugins** | **Mandatory before rewriting any connector.** See [Plugin Engine §8](../01-current-system/06-plugin-execution-engine.md#8-the-target-design) |
| **E2E** | Playwright, ported from the existing suite | The acceptance gate. The current Cypress/Playwright suites are behavioural specifications of the product — the most valuable asset for this rewrite |

**Coverage targets**: Domain ≥ 90%, Application ≥ 80%. Infrastructure and Api are covered by integration tests, not by unit-test percentages.

---

## 11. Package baseline

Managed centrally in `Directory.Packages.props`.

| Concern | Package |
|---|---|
| Data | `Microsoft.EntityFrameworkCore`, `Npgsql.EntityFrameworkCore.PostgreSQL`, `Dapper` |
| Messaging | `MassTransit`, `MassTransit.RabbitMQ`, `MassTransit.EntityFrameworkCore` |
| REST clients | `Microsoft.Extensions.Http.Resilience`, `Microsoft.Extensions.ApiDescription.Client` (typed client codegen from OpenAPI) |
| Gateway | `Yarp.ReverseProxy` |
| Resilience | `Microsoft.Extensions.Http.Resilience`, `Polly` |
| Caching | `Microsoft.Extensions.Caching.Hybrid`, `Microsoft.Extensions.Caching.StackExchangeRedis` |
| Realtime | `Microsoft.AspNetCore.SignalR.StackExchangeRedis` |
| Validation | `FluentValidation.AspNetCore` |
| Observability | `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.*`, `Serilog.AspNetCore` |
| Git | `LibGit2Sharp` |
| JS parsing / execution | `Esprima` (AST for binding analysis), `Jint` (sandboxed JS in the JS worker) |
| Secrets | provider SDK (Vault / AWS Secrets Manager / Azure Key Vault) behind an `ISecretStore` abstraction |
| Testing | `xunit.v3`, `NSubstitute`, `Shouldly`, `Testcontainers.*`, `NetArchTest.Rules` |

**Do not add a package without adding it here first.** Central package management means one version repo-wide.

---

## 12. Things explicitly *not* adopted

Naming these prevents relitigating them per service.

| Not using | Why |
|---|---|
| **MVC controllers** | Minimal APIs with groups give the same structure with a visible pipeline |
| **MediatR** | An in-process mediator adds indirection without solving a problem we have. Application-layer handlers are called directly from endpoints. (If a future need for pipeline behaviours emerges, revisit with an ADR) |
| **A separate read model / full CQRS+ES** | The read and write models are the same shape here. Query classes separate from command handlers is enough |
| **Kafka** | See §6 |
| **`AssemblyLoadContext` for connector isolation** | Reproduces today's PF4J problem. Only a process boundary gives enforceable limits |
| **Reactive Extensions / `IObservable` chains** | `async`/`await` over Kestrel gives the same non-blocking behaviour. Do not port Reactor idioms |
| **A CE/EE inheritance split** | The Java code's `XxxCEImpl` pattern is a licensing artefact. There is no such split here |
| **Repo-per-service** | Nobody asked for it, and it makes cross-cutting changes painful at this team size |
| **`Database.Migrate()` at startup** | Replicas race. Migrations run as a deploy job |

---

[← Database per Service](03-database-per-service.md) · [Next: Service Contracts & Events →](05-service-contracts-and-events.md)
