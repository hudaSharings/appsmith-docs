# Current System — Cross-Cutting Concerns

[← Index](../README.md) · [← Frontend Architecture](08-frontend-architecture.md)

---

The things that touch every service boundary. Each row here becomes a shared building block in the target — see [.NET 10 Standards](../02-target-architecture/04-dotnet-10-standards.md).

---

## 1. Configuration

| Mechanism | Detail |
|---|---|
| `application.properties` + `application-ce/ee.properties` | Spring property files, mostly thin — real config comes from env vars |
| `APPSMITH_*` environment variables | The actual configuration surface. `entrypoint.sh` and `run-with-env.sh` marshal them |
| `configurations/EnvManager` + `InstanceAdminController` | **Admin UI can rewrite the env file and restart the container** (`POST /api/v1/admin/restart`) |
| `Config` collection | Instance-level key/value settings stored in the database (instance id, install info) |

The env-file-rewrite-and-restart mechanism is a single-container artefact. In the target, configuration is per-service `IConfiguration`/`IOptions` with a proper settings store; "restart the instance" is not an application capability.

## 2. Feature flags

- `FeatureFlagService` + `CacheableFeatureFlagHelper` — flags are fetched from a remote cloud service **hourly** by a `@Scheduled` job (`fetchFeatures`), cached in Redis per user/instance, and exposed to the client at `GET /api/v1/users/features`.
- Flag changes can trigger **data migrations** (`featureFlagMigration` helpers) — flags aren't purely presentational.
- `CachedFeatures` / `FeatureFlagEnum` define the flag surface.

Target: a proper flag provider (OpenFeature-compatible) shared across services. The *purpose* survives; the in-process polling job does not.

## 3. Caching

Two layers:

**`reactive-caching` module** — a small generic abstraction with a Redis implementation, exposing:
- `@Cache(cacheName, key)` / `@CacheEvict`
- `@DistributedLock` — the only cross-pod coordination primitive in the system

**What is actually cached** (`repositories/ce/CacheableRepositoryHelperCEImpl.java`):

| Cache | Key | Why it matters |
|---|---|---|
| `permissionGroupsForUser` | `email + organizationId` | **This is already an authorization projection.** Every request reads it |
| `organization` | `organizationId` | Instance config on every request |
| `basePageId → baseApplicationId` | `basePageId` | Avoids a page lookup on view-mode URLs |
| Feature flags | user / instance | Hourly refresh |
| Anonymous user's permission groups | pre-filled at startup | Public app access |

Plus Redis **pub/sub** for broadcasting plugin-install events to all pods (`RedisListenerConfig`).

## 4. Database migrations

**Mongock**, 83 changelog classes under `migrations/db/ce/` (`Migration001_*` … `Migration083_*`), plus three legacy `DatabaseChangelog0/1/2.java` files.

Separately, **JSON schema migrations** (`migrations/JsonSchemaMigration.java`, `JsonSchemaVersions.java`) upgrade exported/git artifact JSON between versions — this is what auto-commit exists to absorb.

> **These 83 changelogs are a primary source of undocumented behaviour.** They encode years of data fixes, defaulting rules, and backfills that exist nowhere else. Read them before finalising the target schema. Several will translate into one-off transforms in the [Data Migration](../03-execution/03-data-migration.md) plan rather than into EF Core migrations.

## 5. Analytics & telemetry

| Channel | Detail |
|---|---|
| **Segment** | `AnalyticsServiceCEImpl` wraps the Segment Java SDK, which has its own internal async queue. Events for signup, login, app create/publish, action execute, datasource create, git ops |
| **Usage pulse** | `UsagePulse` collection + `POST /api/v1/usage-pulse` — anonymous activity beacons used to compute DAU/WAU/MAU |
| **Instance ping** | `pingSchedule` (6h) and `pingStats` (24h) `@Scheduled` jobs report instance health/usage to Appsmith's cloud, Redis-locked |
| **Sentry** | `sentry-spring-boot-starter` for error reporting |
| **Micrometer + Prometheus** | `micrometer-registry-prometheus`, `/actuator` endpoints |
| **OpenTelemetry** | `opentelemetry-bom`, OTLP exporter, `micrometer-tracing-bridge-otel`, custom spans (`ActionSpan`, `OnLoadSpan`). RTS also has OTel instrumentation |

**OpenTelemetry already being here is important** — the target standardises on it rather than introducing something new.

## 6. Email

`notifications/EmailSender.java` and `EmailServiceCEImpl`:

```java
Mono.fromCallable(() -> javaMailSender.send(message))
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe();     // ← nothing awaits this; exceptions are logged and dropped
```

- Templates are Thymeleaf-ish HTML under `src/main/resources/email/`.
- Emails sent: welcome, email verification, password reset, workspace invite, admin instance alerts, test email.
- **No queue, no retry, no delivery record, no dead-letter.** If SMTP is down, the mail is gone and the user sees success.

This is a confirmed defect, and it is the concrete justification for a Notifications service with a broker, retries, a delivery log and a DLQ.

## 7. Scheduled jobs

All Spring `@Scheduled`, in-process. See [Runtime Topology §4](02-runtime-topology.md#4-scheduled-work) for the full table. Summary of what must change:

| Job | Target home |
|---|---|
| Instance ping / usage stats | Notifications & Telemetry Service, `BackgroundService` |
| Feature-flag fetch | Shared flag provider, per-service SDK refresh |
| Remote plugin catalog update | Query Execution Service (owns the plugin catalog) |
| Release-notes refresh | Gateway or Notifications; currently has **no distributed lock** |
| Log cleanup | **Delete it.** Log retention is a platform concern, and it assumes local disk |
| Git auto-commit | Git Versioning Service |

## 8. Error handling & API envelope

- `AppsmithException` + `AppsmithError` — an enum of every error with an app error code, HTTP status, message template, and error type. Roughly 200 entries.
- A global `WebExceptionHandler` wraps everything in a consistent envelope:

```jsonc
{ "responseMeta": { "status": 200, "success": true, "error": null }, "data": { } }
```

**Every client call unwraps `.data`.** The target gateway must preserve this envelope shape or the Angular client's HTTP layer changes everywhere. Recommendation: keep it — it costs nothing and removes a whole class of migration bug.

Note the deliberate 404-instead-of-403 behaviour from [Permissions §2](04-permissions-and-acl.md#2-how-a-permission-check-actually-happens).

## 9. Validation

- Jakarta Bean Validation (`@NotNull`, `@Email`, `hibernate-validator`) on domain objects.
- Custom validators under `validations/`.
- Datasource configuration validity is delegated to the plugin (`PluginExecutor.validateDatasource` → `Datasource.invalids`).

Maps directly to FluentValidation or `System.ComponentModel.DataAnnotations` in .NET.

## 10. Rate limiting

`ratelimiting/` uses **bucket4j with a Redis backend** — currently applied narrowly (login attempts, some plugin trigger endpoints), not as a general API policy.

Target: this belongs at the gateway as a first-class concern, with per-user, per-workspace and per-endpoint policies.

## 11. Security posture — current gaps worth naming

| Gap | Detail | Target treatment |
|---|---|---|
| No plugin sandboxing | 25 connectors in the API process | Isolated worker processes ([Plugin Engine §8](06-plugin-execution-engine.md#8-the-target-design)) |
| Encryption key in the executing process | The same JVM that runs untrusted connector code holds the datasource-decryption key | Secrets manager; execution workers receive short-lived resolved credentials, not the master key |
| No key rotation | Single symmetric key from env | Versioned keys with rotation in the secrets manager |
| No audit log | Nothing records who changed a permission, config or datasource | The integration-event stream provides this by construction |
| Deploy keys in the database | SSH private keys encrypted in Mongo | Secrets manager |
| Assets served from the database | `GET /assets/{id}` is `permitAll` and streams `byte[]` from Mongo | Object storage with signed URLs |

## 12. Testing

| Layer | Tooling |
|---|---|
| Backend unit/integration | JUnit 5, Mockito, `reactor-test` (`StepVerifier`), embedded Mongo (`de.flapdoodle.embed.mongo`) |
| Backend perf | JMH benchmarks |
| Client unit | Jest |
| E2E | **Cypress** and **Playwright** (projects: smoke, sanity, regression, regression-git) |

**The Cypress/Playwright suites are the most valuable asset for the rewrite.** They are framework-agnostic behavioural specifications of the product. Keep them running against the Java system as the reference, and port them to run against the new system as the acceptance gate. See [Roadmap](../03-execution/02-roadmap-and-sequencing.md).

---

[← Frontend Architecture](08-frontend-architecture.md) · [Next: API Endpoint Catalog →](10-api-endpoint-catalog.md)
