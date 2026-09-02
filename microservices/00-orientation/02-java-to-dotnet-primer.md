# Java → .NET Primer: How to Read This Codebase

[← Index](../README.md) · [← Executive Summary](01-executive-summary.md)

---

You do not need to *write* Java for this project. You need to *read* it well enough to extract behaviour and contracts. This document gets you there. Everything below is grounded in constructs actually used in this repository.

---

## 1. The five-minute orientation

Open any backend file, e.g. `app/server/appsmith-server/src/main/java/com/appsmith/server/services/ce/ApplicationPageServiceCEImpl.java`.

```java
package com.appsmith.server.services.ce;      // = C# namespace. MUST match the folder path.

import com.appsmith.server.domains.Application;  // = using, but one type per line, no wildcards by convention

@Slf4j                                          // Lombok: generates a `log` field  → ILogger<T>
@RequiredArgsConstructor                        // Lombok: generates a ctor for all `final` fields → primary constructor / DI
@Service                                        // Spring stereotype → registered in DI container
public class ApplicationPageServiceCEImpl implements ApplicationPageServiceCE {

    private final ApplicationService applicationService;   // `final` = C# `readonly`. Injected by ctor.

    @Override                                              // = C# `override`, but also used for interface impls
    public Mono<Application> createApplication(Application application) { ... }
}
```

Four things to internalise immediately:

1. **`Mono<T>` is `Task<T>`. `Flux<T>` is `IAsyncEnumerable<T>`.** That single substitution makes 80% of this codebase readable.
2. **Lombok annotations generate code you can't see.** `@Getter`/`@Setter` mean the class has properties; `@RequiredArgsConstructor` means the constructor exists but isn't written down.
3. **`XxxCE` / `XxxCEImpl` is not a design pattern you should copy.** `CE` = Community Edition. The commercial edition subclasses these. It's a licensing seam. Ignore it — read the `CE` class, it holds the real logic.
4. **Folder = namespace.** `com/appsmith/server/services/ce/` ⇒ `com.appsmith.server.services.ce`.

---

## 2. Reactive Java ⇄ async C#

This is the highest-value section. Appsmith's backend is **Spring WebFlux**, which means *everything* returns `Mono` or `Flux` and nothing blocks. Project Reactor's operator chains look alien if you're used to `async/await`, but they map cleanly.

| Reactor (Java) | C# equivalent | Notes |
|---|---|---|
| `Mono<T>` | `Task<T>` (0-or-1 result) | `Mono.empty()` ≈ `Task<T?>` returning null |
| `Flux<T>` | `IAsyncEnumerable<T>` | A stream of many |
| `Mono.just(x)` | `Task.FromResult(x)` | |
| `Mono.empty()` | `Task.FromResult<T>(null)` | Critically: an empty Mono **skips all downstream operators** — this is a common source of silent no-ops in the Java code |
| `Mono.error(ex)` | `Task.FromException(ex)` | |
| `.map(f)` | `var x = await t; f(x)` | Synchronous transform |
| `.flatMap(f)` | `await f(await t)` | Transform returning another async op |
| `.zipWith(other)` / `Mono.zip(a,b,c)` | `await Task.WhenAll(a,b,c)` | Parallel fan-out. Used heavily in `ConsolidatedAPIService` |
| `.then(other)` | `await a; await b;` (discards a's value) | |
| `.thenReturn(x)` | `await a; return x;` | |
| `.switchIfEmpty(alt)` | `?? await alt` | Fallback when the source produced nothing |
| `.defaultIfEmpty(x)` | `?? x` | |
| `.filter(p)` | If false → becomes empty → **downstream is skipped** | Not `Where` on a collection. Watch for this |
| `.onErrorResume(f)` | `try { } catch { return await f(ex); }` | |
| `.doOnSuccess(a)` / `.doOnError(a)` | Side effect, doesn't change the value | Logging/analytics hooks |
| `.subscribeOn(Schedulers.boundedElastic())` | `Task.Run(...)` for blocking work | Moves blocking calls off the event loop |
| `.cache()` | Memoise the task result | Prevents re-execution on multiple subscribes |
| `.subscribe()` (no await) | **Fire and forget** | ⚠️ See §2.1 |
| `.block()` | `.GetAwaiter().GetResult()` | Rare in this codebase; a bug when present in request paths |
| `.contextWrite(...)` / `Mono.deferContextual` | `AsyncLocal<T>` / `IHttpContextAccessor` | How the current user is propagated |

### 2.1 The one Reactor idiom that will bite you

```java
Mono.fromCallable(() -> sendEmail(...))
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe();                              // ← nothing awaits this, errors vanish
```

This is `app/server/appsmith-server/src/main/java/com/appsmith/server/notifications/EmailSender.java`. It is fire-and-forget with **swallowed exceptions** — a real defect in the current system, and precisely why the target has a Notifications service with a broker, retries and a dead-letter queue. When you see `.subscribe()` with no return, read it as "this may silently never happen."

### 2.2 Reading a real chain

```java
return applicationService.findById(id, MANAGE_APPLICATIONS)      // Mono<Application>
    .switchIfEmpty(Mono.error(new AppsmithException(NO_RESOURCE_FOUND, ...)))
    .flatMap(app -> newPageService.findByApplicationId(app.getId())  // Flux<NewPage>
        .collectList()                                               // Flux → Mono<List>
        .map(pages -> Tuples.of(app, pages)))
    .flatMap(tuple -> publishPages(tuple.getT1(), tuple.getT2()))
    .thenReturn(Boolean.TRUE);
```

In C#:

```csharp
var app = await applicationService.FindByIdAsync(id, Permission.ManageApplications)
          ?? throw new AppsmithException(ErrorCode.NoResourceFound, ...);
var pages = await newPageService.FindByApplicationIdAsync(app.Id).ToListAsync();
await PublishPagesAsync(app, pages);
return true;
```

**When porting, write the C# version — not a Rx.NET translation of the Java version.** The reactive style is a Spring WebFlux artefact, not domain logic. ASP.NET Core's `async/await` over Kestrel gives you the same non-blocking behaviour with far less ceremony.

---

## 3. Spring ⇄ ASP.NET Core

| Spring / Java | ASP.NET Core / .NET | Where you'll see it here |
|---|---|---|
| `@RestController` + `@RequestMapping("/api/v1/x")` | `[ApiController]` + `[Route]`, or a Minimal API group | `controllers/ce/*ControllerCE.java` |
| `@GetMapping`, `@PostMapping("/{id}")` | `[HttpGet]`, `MapPost("/{id}", ...)` | |
| `@PathVariable`, `@RequestParam`, `@RequestBody` | `[FromRoute]`, `[FromQuery]`, `[FromBody]` | |
| `@Service`, `@Component`, `@Repository` | `services.AddScoped<TInterface, TImpl>()` | Spring auto-scans; .NET registers explicitly. **The .NET way is better here** — the Java DI graph is implicit and huge |
| `@Configuration` + `@Bean` | An extension method on `IServiceCollection` | `configurations/*.java` |
| `@Autowired` / constructor injection | Constructor injection | Same concept |
| `@Value("${appsmith.x}")` | `IOptions<T>` / `IConfiguration` | |
| `application.properties` | `appsettings.json` | `app/server/appsmith-server/src/main/resources/` |
| `SecurityWebFilterChain` | Authentication/Authorization middleware | `configurations/SecurityConfig.java` |
| `WebFilter` | Middleware (`app.Use(...)`) | `filters/` |
| `@Aspect` / AOP | Middleware, decorators, or interceptors | `aspect/GitRouteAspect.java` |
| `@Scheduled(fixedRate=...)` | `BackgroundService` + `PeriodicTimer`, or Quartz/Hangfire | `solutions/ce/ScheduledTaskCEImpl.java` |
| `@Cacheable` / `@CacheEvict` | `IDistributedCache`, `HybridCache` (.NET 9+) | `repositories/ce/CacheableRepositoryHelperCEImpl.java` |
| `ApplicationEventPublisher` | `IMediator` / channel / in-proc event | Narrowly used, git autocommit only |
| `WebClient` | `HttpClient` via `IHttpClientFactory` | `services/ce/AstServiceCEImpl.java` calls RTS |
| Spring Data repository interface | EF Core `DbContext` + repository, or EF directly | `repositories/ce/*RepositoryCE.java` |
| `@Document` (Spring Data Mongo) | An EF Core entity mapped to a table | `domains/*.java` |
| Mongock changelog | EF Core migration | `migrations/db/ce/Migration0xx_*.java` (83 of them) |

---

## 4. Lombok: the annotations that generate invisible code

Appsmith uses Lombok heavily. If a class "has no getters", it has getters.

| Annotation | Generates |
|---|---|
| `@Getter` / `@Setter` | Property accessors for every field |
| `@Data` | `@Getter @Setter` + `equals`/`hashCode`/`toString` ≈ a C# `record`-ish class |
| `@RequiredArgsConstructor` | Constructor taking all `final` fields — **this is how DI works** in most service classes |
| `@AllArgsConstructor` / `@NoArgsConstructor` | Full / parameterless constructors |
| `@Builder` | Fluent builder — `Policy.builder().permission(x).build()` |
| `@Slf4j` | `private static final Logger log` ⇒ use `ILogger<T>` |
| `@FieldNameConstants` | A nested `Fields` class of field-name string constants — used to build Mongo queries by field name. In EF Core you'd use `nameof()` or strongly-typed expressions |
| `@ToString`, `@EqualsAndHashCode` | As named |

**Rule of thumb:** if you see a class body with only fields and annotations, mentally add properties + a constructor.

---

## 5. Other Java-isms you'll hit in this repo

| Construct | What it means | C# analogue |
|---|---|---|
| `Optional<T>` | Explicit maybe-value | `T?` / nullable reference types |
| `Set<String>`, `List<T>`, `Map<K,V>` | Interfaces from `java.util` | `ISet<string>`, `IList<T>`, `IDictionary<K,V>` |
| `Instant` | UTC timestamp | `DateTimeOffset` (or `DateTime` with `Kind.Utc`) |
| `enum` with fields + constructor | Enums can carry data and methods | A `sealed class` with static instances, or an enum + extension methods. See `acl/AclPermission.java` — it's an enum where each value carries a permission string and a target type |
| `interface` with `default` methods | Interface with an implementation body | C# default interface methods. Used extensively on `PluginExecutor` |
| Generics with `<C>` | Same as C# generics… | …but **erased at runtime**. `PluginExecutor<C>` where `C` is the connection type |
| `@JsonView(Views.Public.class)` | Jackson: which fields serialise in which context (public API vs internal vs git export) | `System.Text.Json` doesn't have this. Use **separate DTOs per view** — cleaner and what the target design does |
| `throws` / checked exceptions | Compiler-enforced exception declarations | No equivalent; ignore |
| `static import` | `import static x.Y.Z;` then use `Z` bare | `using static` |
| `var` | Same as C# `var` | |
| `record` | Immutable data carrier | C# `record` |

---

## 6. Build & tooling map

| Java world | .NET world |
|---|---|
| Maven `pom.xml` | `.csproj` |
| Multi-module `<modules>` in a parent pom | A `.sln` with multiple projects |
| `mvn clean install` | `dotnet build` |
| `~/.m2` local repo | NuGet global packages folder |
| Maven Central | nuget.org |
| JUnit 5 + Mockito | xUnit + NSubstitute/Moq |
| Testcontainers (Java) | Testcontainers for .NET (same project, both languages) |
| Spring Boot fat jar | `dotnet publish` self-contained or framework-dependent |
| JVM | CLR |
| Jackson | `System.Text.Json` |
| SLF4J + Logback | `Microsoft.Extensions.Logging` + Serilog |
| Micrometer + OpenTelemetry | `System.Diagnostics.Metrics` + OpenTelemetry .NET |

### The module layout of this repo

```
app/server/
├── pom.xml                  ← parent (the .sln)
├── appsmith-interfaces/     ← shared contracts + plugin SPI  (a shared .Contracts library)
├── appsmith-plugins/        ← 25 connector modules            (plugin assemblies)
├── appsmith-server/         ← THE MONOLITH                    (everything else)
├── appsmith-git/            ← JGit wrapper, zero deps back    (cleanest boundary in the repo)
└── reactive-caching/        ← Redis cache abstraction
```

Dependency direction is one-way and clean: `appsmith-server → appsmith-git`, `appsmith-server → appsmith-interfaces`, `appsmith-plugins/* → appsmith-interfaces`. Nothing depends back on `appsmith-server`.

---

## 7. Two frameworks with no .NET equivalent you'll need explained

### PF4J (Plugin Framework for Java)
Loads each connector into its **own classloader** inside the same JVM, so two connectors can depend on different versions of the same library without conflict. Discovery is by package name (`configurations/PluginConfiguration.java`, `helpers/PluginExecutorHelper.java`).

**There is no process isolation.** A connector that leaks memory, holds threads, or crashes takes the whole server with it.

.NET's nearest equivalent is `AssemblyLoadContext` — but the target design deliberately does **not** use it. It puts connectors in separate **worker processes** instead, which is the actual fix. See [Plugin & Execution Engine](../01-current-system/06-plugin-execution-engine.md).

### Mongock
Versioned MongoDB migrations — the Mongo analogue of EF Core Migrations or Flyway. Each changelog class is annotated and applied once, tracked in a Mongo collection. There are 83 of them under `migrations/db/ce/`. They encode years of data fixes and are a **primary source of behaviour you won't find anywhere else** — read them before designing the target schema.

---

## 8. Naming conventions in this codebase

| Suffix / pattern | Meaning |
|---|---|
| `*CE` / `*CEImpl` | Community Edition base class. **The real implementation.** The non-CE class usually just extends it |
| `*Service` / `*ServiceImpl` | Standard Spring service + interface pair |
| `*Solution` | Cross-context orchestration — logic that spans bounded contexts. `ActionExecutionSolution`, `PolicySolution`, `UserAndAccessManagementService`. **These are the seams that become sagas or gateway composition in the target** |
| `*DTO` | Wire/transfer shape, distinct from the persisted domain object |
| `New*` (`NewAction`, `NewPage`) | Not "new" — historical rename. These are *the* Action and Page entities. `NewAction` replaced an older `Action` collection years ago |
| `Base*` | Abstract base class — `BaseDomain` is on every persisted entity |
| `*Helper` / `*Util` | Static-ish helpers |
| `Custom*RepositoryImpl` | Hand-written Mongo queries that Spring Data can't derive from a method name |
| `Artifact` | The abstraction over "a thing that can be git-versioned / exported" — today only Application implements it |

---

## 9. Quick-reference cheat sheet

```
Mono<T>              → Task<T>
Flux<T>              → IAsyncEnumerable<T>
.flatMap(f)          → await f(await x)
.map(f)              → f(await x)
Mono.zip(a,b)        → await Task.WhenAll(a,b)
.switchIfEmpty(e)    → ?? await e
.onErrorResume(f)    → try/catch
.subscribe()         → fire-and-forget (usually a bug)
@Service             → services.AddScoped<>()
@RestController      → [ApiController] / MapGroup
@Document            → EF Core entity → table
@Cacheable           → HybridCache / IDistributedCache
@Scheduled           → BackgroundService + PeriodicTimer
final field          → readonly field
Optional<T>          → T?
Instant              → DateTimeOffset
@Slf4j → log         → ILogger<T>
@RequiredArgsConstructor → primary constructor
XxxCEImpl            → read this one; ignore the subclass
```

---

## 10. When in doubt

- **Behaviour questions**: run the current system (`docker-compose up`) and observe it. The Java code is dense; the HTTP contract is not.
- **Contract questions**: [API Endpoint Catalog](../01-current-system/10-api-endpoint-catalog.md) has every route.
- **"Why is it like this?"**: usually the answer is in a Mongock migration or a git-sync constraint, not in the service class.

---

[← Executive Summary](01-executive-summary.md) · [Next: System Overview →](../01-current-system/01-system-overview.md)
