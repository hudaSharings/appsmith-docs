# Delivery Tracking — Epics, Features & User Stories
[<- Index](../README.md) · [<- Risks & ADRs](../03-execution/04-risks-and-adrs.md)
---
A JIRA-ready backlog derived directly from the architecture: **each service is an Epic**, grouped into **Features**, broken into **User Stories** a developer can pick up and estimate. Every story is grounded in a real endpoint, table or roadmap step from the rest of this documentation set — nothing here is generic scaffolding.
**Companion files:**
- [`00-jira-setup-guide.md`](00-jira-setup-guide.md) — how to load this into JIRA: hierarchy, components, labels, story points, workflow
- [`02-jira-import.csv`](02-jira-import.csv) — the same backlog as a CSV, ready for JIRA's CSV importer

---
## Epics at a glance
| Epic | Service | Phase | Features | Stories |
|---|---|---|---|---|
| [PLAT — Platform & Foundation](#platform-foundation) | `Platform` | Phase A | 5 | 23 |
| [IAM — Identity & Access Service](#identity-access-service) | `Identity` | Phase A | 8 | 28 |
| [APP — Application Service](#application-service) | `Application` | Phase B | 14 | 40 |
| [DS — Datasource Service](#datasource-service) | `Datasource` | Phase B | 7 | 15 |
| [EXEC — Query Execution Service](#query-execution-service) | `Execution` | Phase C | 8 | 24 |
| [GIT — Git Versioning Service](#git-versioning-service) | `Git` | Phase D | 7 | 16 |
| [RT — Realtime Collaboration Service](#realtime-collaboration-service) | `Realtime` | Phase E | 4 | 7 |
| [NOTIF — Notifications & Telemetry Service](#notifications-telemetry-service) | `Notifications` | Phase F | 5 | 9 |
| [GW — API Gateway BFF Hardening](#api-gateway-bff-hardening) | `Gateway` | Phase G | 5 | 8 |
| [NG — Angular Client](#angular-client) | `Client` | Parallel, C0-C7 | 8 | 29 |
| [MIG — Data Migration & Cutover](#data-migration-cutover) | `Migration` | Pre-cutover | 8 | 18 |

**Total: 11 epics · 79 features · 217 user stories.**

---

## PLAT — Platform & Foundation
**Component:** `Platform` &nbsp;·&nbsp; **Roadmap phase:** Phase A

> Shared building blocks, CI/CD, local dev environment and the gateway walking skeleton. Every other epic depends on this one shipping first.

### Feature: Shared BuildingBlocks Libraries
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement BuildingBlocks.Abstractions**<br><span style="font-size:90%">As a platform engineer, I want a shared library of IClock, IUserContext and Result<T>, so that every service uses the same core primitives instead of inventing its own.</span> | - IClock and TimeProvider wired for testability<br>- IUserContext contract defined (userId, instanceId, roleIds, correlationId)<br>- Result<T> pattern documented with usage examples<br>- Published as a versioned NuGet package to the internal feed | 3 | High |
| **Implement BuildingBlocks.Observability**<br><span style="font-size:90%">As a platform engineer, I want OpenTelemetry wired for ASP.NET Core, HttpClient, Npgsql, gRPC and MassTransit, so that every service emits consistent traces, metrics and structured logs out of the box.</span> | - OTLP exporter configured via IConfiguration<br>- Correlation id propagated through HTTP headers, gRPC metadata and broker message headers<br>- Serilog structured logging to stdout with resource attributes (service.name/version/instance)<br>- /health/live and /health/ready helpers included | 5 | High |
| **Implement BuildingBlocks.Messaging**<br><span style="font-size:90%">As a platform engineer, I want a MassTransit + RabbitMQ package with outbox and inbox conventions baked in, so that no service has to hand-roll idempotent, transactionally-safe event publishing.</span> | - AddEntityFrameworkOutbox extension method with sane defaults<br>- Inbox table + idempotent-consumer base class<br>- Retry + delayed-redelivery policy preset<br>- Sample producer and consumer used in an integration test | 8 | High |
| **Implement BuildingBlocks.Auth**<br><span style="font-size:90%">As a platform engineer, I want middleware that validates the internal JWT and populates IUserContext, so that services never read HttpContext directly for identity.</span> | - JWT signature validated against the shared JWKS<br>- IUserContext populated from claims (userId, instanceId, roleIds, correlationId)<br>- Middleware registered with one extension method<br>- Unit tests cover expired, malformed and valid tokens | 5 | High |
| **Implement BuildingBlocks.Persistence**<br><span style="font-size:90%">As a platform engineer, I want EF Core conventions for soft delete and audit columns, so that every service gets deletedAt filtering and created/updated stamping for free.</span> | - Global query filter applied via one extension method<br>- SaveChanges interceptor stamps created_at/updated_at/created_by/updated_by<br>- Delete() converts to a deletedAt write, never a real DELETE, on annotated entities<br>- Verified against a sample DbContext in an integration test | 5 | High |
| **Port the permission cascade graph as a pure function**<br><span style="font-size:90%">As a platform engineer, I want the hierarchy and lateral permission graphs ported from PolicyGeneratorCE as pure C# functions, so that every service that needs authz_grants derivation uses one tested implementation, never a fork.</span> | - CascadeToChild(parent, child, permission) implemented and unit tested against the Java enum values<br>- Implied(permission) implemented for same-resource lateral grants<br>- Zero I/O — fully unit-testable, no database or HTTP<br>- Test suite ports the equivalent cases from PolicyGeneratorCE | 8 | High |

### Feature: CI/CD & Architecture Enforcement
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Set up solution-wide build configuration**<br><span style="font-size:90%">As a platform engineer, I want Directory.Build.props and Directory.Packages.props at the repo root, so that every project shares LangVersion, Nullable, TreatWarningsAsErrors and one package version each.</span> | - Nullable reference types enabled, warnings as errors<br>- Central package management enforced — no per-project <PackageReference Version><br>- global.json pins the SDK version | 2 | High |
| **Add architecture tests to the service template**<br><span style="font-size:90%">As a tech lead, I want a NetArchTest suite enforcing the Domain/Application/Infrastructure/Api layer rules, so that a layer violation fails the build instead of surviving code review.</span> | - Domain project may not reference EF Core, ASP.NET or MassTransit<br>- Application project may not reference concrete Infrastructure types<br>- Rules run as part of every service's unit test project<br>- Template documented so new services start with the tests included | 5 | High |
| **Build the CI pipeline**<br><span style="font-size:90%">As a platform engineer, I want a pipeline that builds, tests, checks formatting and builds container images for every service, so that every merge to main is verified the same way regardless of which service changed.</span> | - dotnet build / dotnet test / dotnet format --verify-no-changes all gate the merge<br>- Container image built and pushed per service on merge<br>- Pipeline runs in under a target wall-clock budget agreed with the team | 8 | High |
| **Add .proto contract diff checking to CI**<br><span style="font-size:90%">As a tech lead, I want the pipeline to diff every service's .proto files against the previous release, so that a breaking contract change is caught before it reaches a consumer.</span> | - Diff tool flags removed fields, retyped fields and removed RPCs as breaking<br>- Breaking changes fail the build unless the PR carries a contract-break label<br>- Non-breaking additions pass silently | 5 | Medium |
| **Add EF Core migrations as a deploy-time job**<br><span style="font-size:90%">As a platform engineer, I want a dedicated Kubernetes Job (or compose step) that applies pending migrations before a service starts, so that eight replicas never race to apply the same migration.</span> | - No service calls Database.Migrate() in Program.cs<br>- Migration job runs to completion before the deployment rollout proceeds<br>- Job failure blocks the deployment | 3 | High |

### Feature: Local Dev Environment (.NET Aspire)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Author the Aspire AppHost**<br><span style="font-size:90%">As a developer, I want one command that starts Postgres, Redis, RabbitMQ and every service locally, so that I don't need to hand-configure eight services and three pieces of infrastructure to start working.</span> | - dotnet run on the AppHost starts all configured resources<br>- Six Postgres databases provisioned automatically<br>- Aspire dashboard shows logs and traces across every service | 5 | High |
| **Wire service discovery for Identity, Application and Datasource**<br><span style="font-size:90%">As a developer, I want the AppHost to inject connection strings and service URLs automatically, so that I never hand-edit an appsettings.json to point one service at another locally.</span> | - WithReference() wiring covers every declared sync dependency<br>- Services resolve each other by Aspire-assigned names, not hardcoded ports<br>- Works after a machine restart with no manual steps | 5 | Medium |
| **Add the Angular app to the AppHost**<br><span style="font-size:90%">As a frontend developer, I want the Angular dev server to start alongside the backend via AddNpmApp, so that I get the whole stack running with one command during frontend work too.</span> | - ng serve launches through the AppHost<br>- Angular app's API base URL points at the gateway automatically<br>- Hot reload still works for both sides | 3 | Medium |

### Feature: API Gateway Walking Skeleton
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Stand up YARP routing for all seven services**<br><span style="font-size:90%">As a platform engineer, I want a reverse-proxy routing table mapping /api/v1/* prefixes to their owning service, so that the gateway can route a request without any service knowing about any other.</span> | - Routes configured for Identity, Application, Datasource paths at minimum for the skeleton<br>- Unmatched routes return a clean 404, not a proxy error<br>- Routing table is data-driven, not hardcoded per route | 5 | High |
| **Implement cookie session and CSRF**<br><span style="font-size:90%">As a security engineer, I want a __Host- prefixed HttpOnly session cookie plus double-submit CSRF, so that the Angular client's auth model matches what the current React client already does.</span> | - Session cookie is HttpOnly, Secure, SameSite=Lax<br>- XSRF-TOKEN cookie set and validated via header on mutating requests<br>- CSRF failure returns a clear 403 with no state change | 5 | High |
| **Wire session validation with a Redis cache**<br><span style="font-size:90%">As a platform engineer, I want the gateway to call Identity's ValidateSession over gRPC and cache the result in Redis, so that most requests don't pay a network hop for authentication.</span> | - Cache hit path skips the gRPC call entirely<br>- Cache miss falls back to gRPC and re-populates the cache<br>- TTL is short and configurable | 5 | High |
| **Implement internal JWT minting**<br><span style="font-size:90%">As a platform engineer, I want the gateway to mint a short-lived signed JWT for every downstream call, so that services never see the browser's session cookie.</span> | - Token TTL is ~60 seconds and never persisted<br>- Token carries userId, instanceId, roleIds and correlationId<br>- Downstream services validate it against a shared JWKS | 3 | High |
| **Implement the response envelope and authorization filter**<br><span style="font-size:90%">As a frontend developer, I want every gateway response wrapped in {responseMeta, data} and every authz failure returned as 404, so that the client's existing unwrap-and-error-handling logic needs no changes.</span> | - Envelope applied once as a filter, not per endpoint<br>- Authorization failures return 404 with no existence leak, matching current behaviour<br>- Envelope shape verified against a sample of current Java responses | 3 | High |
| **Implement gateway rate limiting**<br><span style="font-size:90%">As a platform engineer, I want token-bucket rate limits per user, per workspace and per endpoint, so that one bad client can't degrade the platform for everyone else.</span> | - Login and signup have tighter limits than general CRUD<br>- Limit breach returns 429 with a Retry-After header<br>- Limits are configurable without a redeploy | 5 | Medium |

### Feature: Reference Material Capture
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Build the connector golden-file capture harness**<br><span style="font-size:90%">As a tech lead, I want tooling that records request/response pairs from each running Java plugin, so that every rewritten connector has an objective correctness bar before it ships.</span> | - Harness captures datasource config + action config in, ActionExecutionResult out<br>- Captures error shapes, type coercion and pagination, not just the happy path<br>- Output format is directly consumable as a .NET test fixture | 8 | Highest |
| **Export the application DSL corpus**<br><span style="font-size:90%">As a tech lead, I want a few hundred real page DSLs exported from the Java system as fixtures, so that the evaluation-engine port, visual regression and widget prioritisation all have real data to work from.</span> | - Corpus spans a representative range of widget usage, not just simple pages<br>- Fixtures are anonymised of any customer-identifying content<br>- Corpus is versioned and reusable across every phase that needs it | 5 | High |
| **Catalogue the Mongock changelogs**<br><span style="font-size:90%">As a tech lead, I want all 83 migration changelog classes read and summarised for migration-relevant behaviour, so that the schema and data-migration design don't silently drop a years-old data fix.</span> | - Each changelog's intent is summarised in one line<br>- Changelogs with an equivalent needed in the new schema are flagged explicitly<br>- Findings feed directly into the Data Migration epic | 8 | Medium |

---

## IAM — Identity & Access Service
**Component:** `Identity` &nbsp;·&nbsp; **Roadmap phase:** Phase A

> Authenticate people and be the single source of truth for what they may do: users, workspaces, memberships, roles and permission grants.

### Feature: Instance & Tenancy Foundation
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Create the identity_db schema**<br><span style="font-size:90%">As a backend developer, I want EF Core migrations for instances, users, workspaces and their supporting tables, so that the service has a real database to build against.</span> | - Matches the DDL in Database per Service §5.1<br>- Migrations run cleanly against a fresh Postgres instance<br>- Soft-delete and audit conventions from BuildingBlocks.Persistence applied | 5 | High |
| **Collapse Tenant and Organization into one Instance record**<br><span style="font-size:90%">As a backend developer, I want the domain model to represent instance-level config as a single row, so that the two-collection ambiguity in the current system doesn't carry into the new one.</span> | - One instances row per deployment, holding branding and enabled-auth-method config<br>- No Tenant/Organization duplication anywhere in the schema<br>- Migration mapping documented for the data-migration team | 5 | High |
| **Implement instance configuration endpoints**<br><span style="font-size:90%">As an instance administrator, I want to read and update instance-level configuration, so that I can set branding and enabled login methods without touching the database directly.</span> | - GET returns the current configuration<br>- PUT updates it and is restricted to instance admins<br>- Response matches the current /tenants/current shape at the gateway | 3 | Medium |

### Feature: User Registration & Authentication
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement signup**<br><span style="font-size:90%">As a new user, I want to sign up with an email and password, so that I get an account, a default workspace and admin access to it in one step.</span> | - Creates the user, a default workspace, three roles (Admin/Developer/Viewer) and the role assignment in one transaction<br>- Duplicate email returns a clear validation error<br>- UserSignedUp event published via the outbox | 8 | High |
| **Implement login**<br><span style="font-size:90%">As a registered user, I want to log in with my email and password, so that I get an authenticated session.</span> | - Password verified with Argon2id<br>- Disabled or unverified accounts are rejected with a clear reason<br>- Returns UserContext (userId, instanceId, roleIds) to the caller | 5 | High |
| **Implement first-user / instance-admin signup**<br><span style="font-size:90%">As the person deploying a fresh instance, I want the very first signup to automatically receive instance-admin rights, so that there's a bootstrapping path with no manual database edit.</span> | - Only applies when zero users exist on the instance<br>- Grants MANAGE_INSTANCE_CONFIGURATION-equivalent rights<br>- Subsequent signups follow the normal path | 3 | Medium |
| **Implement GET /users/me**<br><span style="font-size:90%">As the Angular client, I want an endpoint returning the current user's profile, so that the app shell can render the signed-in user without a second round trip after login.</span> | - Returns profile, workspace list summary and feature-flag-relevant fields<br>- 401 when no valid session/token is present<br>- Response shape matches the current contract | 2 | High |
| **Implement profile update**<br><span style="font-size:90%">As a user, I want to update my display name and photo, so that my profile reflects who I am.</span> | - Photo upload writes to object storage, not the database<br>- Validation matches current field constraints<br>- Update is reflected immediately in GET /users/me | 3 | Medium |

### Feature: Password Reset & Email Verification
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement forgot-password**<br><span style="font-size:90%">As a user who forgot their password, I want to request a reset link, so that I can regain access to my account.</span> | - Issues a short-lived token and publishes UserPasswordResetRequested<br>- Response is identical whether or not the email exists, to avoid account enumeration<br>- Token expiry matches the documented window | 5 | High |
| **Implement reset-password**<br><span style="font-size:90%">As a user with a valid reset token, I want to set a new password, so that I can log in again.</span> | - Token is single-use and expires after the documented window<br>- New password is hashed with Argon2id<br>- Invalid or expired tokens return a clear, non-leaking error | 3 | High |
| **Implement email verification**<br><span style="font-size:90%">As a newly signed-up user, I want to verify my email address via a link, so that the platform knows the address is real.</span> | - Verification token issuance publishes UserEmailVerificationRequested<br>- Verify endpoint marks the account verified and is idempotent on re-click<br>- Resend-verification endpoint respects a rate limit | 3 | Medium |

### Feature: OAuth / OIDC Login
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement Google OAuth2 login**<br><span style="font-size:90%">As a user, I want to sign in with my Google account, so that I don't need a separate password for this platform.</span> | - Standard authorization-code flow with PKCE<br>- First-time login auto-provisions a user record linked via external_logins<br>- Existing email match links rather than duplicates the account | 5 | Medium |
| **Implement GitHub OAuth2 login**<br><span style="font-size:90%">As a user, I want to sign in with my GitHub account, so that I have another social login option.</span> | - Same flow shape as Google login<br>- Handles GitHub's email-may-be-private case explicitly | 5 | Medium |
| **Implement generic OIDC provider support**<br><span style="font-size:90%">As an instance administrator, I want to configure an arbitrary OIDC provider (e.g. for enterprise SSO), so that my organisation can use its existing identity provider.</span> | - Provider metadata configurable without a code change<br>- Claims mapping is configurable<br>- Documented setup guide for a common provider (Okta/Azure AD) as a worked example | 8 | Low |

### Feature: Workspace Management
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement workspace creation with default roles**<br><span style="font-size:90%">As a user, I want to create a workspace and automatically get Administrator/Developer/App Viewer roles for it, so that I have a working permission structure from the moment the workspace exists.</span> | - Three roles created with the correct default grants, matching the current AppsmithRole definitions<br>- Creator is assigned to Administrator<br>- All of this happens in one local transaction | 8 | High |
| **Implement workspace CRUD**<br><span style="font-size:90%">As a workspace administrator, I want to view, rename and delete my workspace, so that I can manage its lifecycle.</span> | - Rename validates uniqueness where required<br>- Delete initiates the WorkspaceDeleted saga rather than deleting synchronously<br>- Only Administrators can delete a workspace | 5 | High |
| **Implement workspace list for the current user**<br><span style="font-size:90%">As the Angular client, I want an endpoint returning every workspace the current user belongs to, so that the home screen can render the workspace switcher.</span> | - Returns only workspaces the user is a member of<br>- Includes the user's effective role per workspace<br>- Matches the current /workspaces/home contract | 3 | High |
| **Implement workspace member list and role change**<br><span style="font-size:90%">As a workspace administrator, I want to view members and change a member's role, so that I can manage who has what access.</span> | - Role change publishes RoleAssignmentChanged<br>- Only users with assign:permissionGroups may change roles<br>- Cannot remove the last Administrator from a workspace | 5 | High |
| **Implement invite users to a workspace**<br><span style="font-size:90%">As a workspace administrator, I want to invite people by email to a specific role, so that my team can start collaborating.</span> | - Creates a placeholder user if the email doesn't exist yet<br>- Publishes WorkspaceMemberAdded for the Notifications service to send the invite email<br>- Bulk-invite (multiple emails in one call) is supported | 8 | High |
| **Implement leave-workspace**<br><span style="font-size:90%">As a workspace member, I want to remove myself from a workspace, so that I can stop being associated with a workspace I no longer need.</span> | - The last remaining Administrator cannot leave without transferring the role first<br>- Publishes WorkspaceMemberRemoved<br>- Session/role-set cache invalidated immediately for that user | 3 | Medium |

### Feature: Roles & Permission Grants (RBAC)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement role and permission_grants schema and CRUD**<br><span style="font-size:90%">As a backend developer, I want tables and endpoints for roles and their grants, so that the authorization model has somewhere to live.</span> | - Matches the identity_db DDL for roles/role_assignments/permission_grants<br>- Unique constraint prevents duplicate grants<br>- System roles (Admin/Developer/Viewer) are flagged is_system and protected from deletion | 8 | High |
| **Apply the permission cascade graph on resource creation**<br><span style="font-size:90%">As a backend developer, I want new resources to receive grants derived from the workspace's roles via the ported permission graph, so that a new application/page/action is correctly accessible the moment it's created.</span> | - Uses BuildingBlocks.Authorization's ported graph, not a local reimplementation<br>- Verified against the current PolicyGeneratorCE cascade cases<br>- Covers both hierarchy (parent-to-child) and lateral (same-resource) grants | 8 | High |
| **Publish permission change events via the outbox**<br><span style="font-size:90%">As a backend developer, I want PermissionGrantChanged and RoleAssignmentChanged published transactionally whenever grants or assignments change, so that downstream services can keep their authz_grants projections current.</span> | - Every grant/assignment mutation writes to the outbox in the same transaction<br>- Event payload carries enough detail for a consumer to apply a targeted update<br>- Verified end-to-end with a test consumer | 5 | High |
| **Pre-seed the anonymous role**<br><span style="font-size:90%">As a platform engineer, I want an anonymous role created automatically for every instance, so that making an application public has a role to grant, matching today's behaviour.</span> | - Anonymous role exists exactly once per instance<br>- Role id is discoverable by services that need to grant it (e.g. on changeAccess)<br>- Covered by an integration test for the public-app flow | 3 | Medium |

### Feature: Session Validation gRPC API
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the ValidateSession gRPC endpoint**<br><span style="font-size:90%">As the API Gateway, I want a fast gRPC call that resolves a session to a UserContext, so that the gateway can authenticate every request without hitting the database each time.</span> | - Response includes userId, instanceId, email, roleIds and anonymous/instance-admin flags<br>- p99 latency documented and load-tested<br>- Unknown/expired sessions return a clear unauthenticated result, not an exception | 5 | High |
| **Implement RevokeUserSessions**<br><span style="font-size:90%">As the API Gateway, I want a gRPC call that immediately invalidates a user's cached sessions, so that hard revocations (removed from workspace, deactivated) take effect instantly.</span> | - Called on WorkspaceMemberRemoved and UserDeactivated<br>- Gateway's Redis session cache is cleared for that user within the call<br>- Covered by an integration test asserting the next request is unauthenticated | 5 | High |

### Feature: Identity Event Publishing
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Wire outbox publishing for every identity event**<br><span style="font-size:90%">As a backend developer, I want every domain event listed in the Contracts & Events catalog published reliably, so that every consuming service (Application, Datasource, Notifications, Gateway) gets a complete, ordered event stream.</span> | - Every event type in the catalog has a corresponding outbox write path<br>- Dispatcher publishes to RabbitMQ within the documented latency budget<br>- Verified with an integration test per event type | 5 | High |
| **Implement the WorkspaceDeleted saga initiator**<br><span style="font-size:90%">As a workspace administrator, I want deleting a workspace to kick off a tracked cascade across Application, Datasource and Git, so that deletion completes correctly even though the data lives in other services.</span> | - Workspace is marked deleting immediately and disappears from listings<br>- Waits for acknowledgement from all three participants before finalising<br>- Times out to an operator-visible alert rather than hanging silently forever | 8 | Medium |

---

## APP — Application Service
**Component:** `Application` &nbsp;·&nbsp; **Roadmap phase:** Phase B

> Everything a user authors: applications, pages, layouts, actions, JS objects, themes — one transactional aggregate, kept together so publish stays atomic.

### Feature: Application CRUD
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Create the application_db schema**<br><span style="font-size:90%">As a backend developer, I want EF Core migrations for applications and application_pages, so that the service has a database matching the target design.</span> | - Matches Database per Service §5.2<br>- draft_*/published_* columns present on applications<br>- base_id/ref_name unique constraint in place for git branching | 5 | High |
| **Implement create application**<br><span style="font-size:90%">As a user, I want to create a new application in a workspace, so that I can start building.</span> | - Auto-creates a default Page1 with an empty layout<br>- Name collisions get a sequence suffix (e.g. 'Untitled application 3')<br>- Policies/grants inherited from the workspace via the cascade graph | 8 | High |
| **Implement application CRUD endpoints**<br><span style="font-size:90%">As a user, I want to view, rename and delete my applications, so that I can manage their lifecycle.</span> | - Delete is a soft delete (deleted_at), not a hard delete<br>- Rename updates the slug consistently<br>- Read enforces the authz_grants join, returning 404 not 403 on denial | 5 | High |
| **Implement the application list for the home screen**<br><span style="font-size:90%">As the Angular client, I want an endpoint returning applications in a workspace with summary metadata, so that the home screen renders without N+1 calls.</span> | - Includes name, icon, color, last-edited and last-deployed timestamps<br>- Excludes soft-deleted applications<br>- Paginated for workspaces with many applications | 3 | High |
| **Implement change-access (make public/private)**<br><span style="font-size:90%">As a user, I want to toggle whether my published application is publicly viewable, so that I can share it without requiring a login.</span> | - Publishes ApplicationAccessChanged for Identity to grant/revoke the anonymous role<br>- Change takes effect for new requests immediately<br>- Only users with makePublic rights may toggle it | 5 | Medium |
| **Implement clone application**<br><span style="font-size:90%">As a user, I want to duplicate an application within the same workspace, so that I can experiment without risking the original.</span> | - Clone gets a fresh id and base_id, name suffixed appropriately<br>- All pages, actions, collections and theme are duplicated<br>- Clone appears immediately in the application list | 5 | Medium |

### Feature: Page & Layout Management
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Create the pages schema**<br><span style="font-size:90%">As a backend developer, I want the pages table with draft/published columns as designed, so that the draft/published model has a home.</span> | - draft_layout and published_layout are jsonb<br>- Unique (base_id, ref_name) constraint<br>- Partial index on application_id where deleted_at is null | 5 | High |
| **Implement page CRUD endpoints**<br><span style="font-size:90%">As a user, I want to create, read, update and delete pages within an application, so that I can structure my application into multiple screens.</span> | - Create inserts with an empty draft_layout<br>- Delete is soft, honoured by publish's hard-delete-of-removed-pages step<br>- Read (edit and view variants) matches current contract shapes | 5 | High |
| **Implement page reorder and make-default**<br><span style="font-size:90%">As a user, I want to reorder pages and set which one is the default, so that navigation and the initial view load correctly.</span> | - application_pages ordering updates atomically<br>- Exactly one page can be default per application<br>- Reflected immediately in the application_pages projection used by page load | 3 | Medium |
| **Implement clone page**<br><span style="font-size:90%">As a user, I want to duplicate a page including its actions, so that I can reuse a page layout as a starting point elsewhere.</span> | - Clones the page's actions and action collections along with it<br>- New page gets a fresh id and base_id<br>- Bindings referencing the cloned actions are rewritten to the new ids | 5 | Medium |

### Feature: Canvas DSL Save & Binding Analysis
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the binding extractor**<br><span style="font-size:90%">As a backend developer, I want an Esprima-based parser that walks a DSL and extracts every {{ }} binding's referenced identifiers, so that the on-load dependency engine has real input to work with.</span> | - Handles nested widget trees of arbitrary depth<br>- Handles multi-statement JS bindings, not just simple property lookups<br>- Unit tested against a representative sample of binding syntaxes from the DSL corpus | 8 | High |
| **Implement the layout save endpoint**<br><span style="font-size:90%">As a user, I want to save my canvas edits, so that my changes persist.</span> | - Writes to draft_layout as jsonb with no field-name escaping needed<br>- Triggers binding analysis and on-load plan recomputation in the same request<br>- Response matches the current LayoutDTO contract | 5 | High |
| **Verify binding-analysis parity against the Java+RTS baseline**<br><span style="font-size:90%">As a tech lead, I want the new in-process analyser's output compared against the Java system's RTS-backed output across the full DSL corpus, so that we know binding semantics haven't silently changed before this ships.</span> | - Diff run across every fixture in the DSL corpus<br>- Zero unexplained mismatches, or every mismatch triaged and signed off<br>- This is a go/no-go checkpoint per the roadmap — result recorded | 8 | Highest |

### Feature: On-Load Action Dependency Engine
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the dependency DAG builder**<br><span style="font-size:90%">As a backend developer, I want a graph built from binding edges with cycle detection, so that the on-load plan can be computed correctly and cycles are surfaced as errors, not infinite loops.</span> | - Cycle detection reports which entities form the cycle<br>- Graph construction is a pure function, unit-testable without a database | 8 | High |
| **Implement topological sort into waves**<br><span style="font-size:90%">As a backend developer, I want the dependency graph sorted into parallelisable waves, so that the client receives an onload plan matching today's layoutOnLoadActions shape.</span> | - Output type is a list of sets, matching the current wave structure exactly<br>- Actions with no dependencies land in the earliest possible wave<br>- Verified against the DSL corpus's existing onload plans | 5 | High |
| **Persist the on-load plan and run-behaviour flags**<br><span style="font-size:90%">As a backend developer, I want the computed plan written to draft_onload_plan alongside per-action executeOnLoad/runBehaviour flags, so that the editor and viewer both have a correct execution order.</span> | - Plan recomputes on every layout save, not just on demand<br>- runBehaviour toggle endpoints update the plan without requiring a full layout save | 3 | High |

### Feature: Actions & JS Objects CRUD
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the actions schema and CRUD**<br><span style="font-size:90%">As a backend developer / user, I want the actions table and endpoints for creating, updating and deleting queries, so that users can define what their application queries.</span> | - draft_config/published_config store the ActionConfiguration as jsonb<br>- plugin_id and datasource_id reference Execution/Datasource services by id, not by embedded copy<br>- CRUD enforces authz_grants exactly as pages do | 5 | High |
| **Implement JS Objects (action_collections) CRUD**<br><span style="font-size:90%">As a user, I want to create and edit JS Objects containing multiple functions, so that I can write reusable logic bound from the canvas.</span> | - draft_body stores the JS source as text, not jsonb<br>- Update-body endpoint matches the current PUT .../body contract<br>- Delete cascades to functions defined within it | 5 | High |
| **Implement move-action between pages**<br><span style="font-size:90%">As a user, I want to move a query from one page to another, so that I can reorganise my application.</span> | - Updates page_id and re-runs onload-plan computation for both the old and new page<br>- Bindings referencing the action by name continue to resolve | 3 | Medium |
| **Implement executeOnLoad and runBehaviour toggle endpoints**<br><span style="font-size:90%">As a user, I want to control whether and how a query runs automatically, so that I can tune my application's load-time behaviour.</span> | - Toggle takes effect on the next onload-plan computation<br>- Matches the current PUT .../executeOnLoad/{id} and .../runBehaviour/{id} contracts | 3 | Medium |

### Feature: Refactor / Rename with Binding Rewrite
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement entity rename with binding rewrite**<br><span style="font-size:90%">As a user, I want to rename a query or widget and have every binding that references it update automatically, so that I don't have to manually find and fix every reference myself.</span> | - Scans every page's draft_layout and every action's draft_config for references to the old name<br>- Rewrite is atomic — either every reference updates or none do<br>- Matches the current PUT /layouts/refactor and /actions/refactor contracts | 8 | Medium |

### Feature: Themes & Custom JS Libraries
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement themes CRUD and default assignment**<br><span style="font-size:90%">As a user, I want to select and customise a theme for my application, so that my application has consistent styling.</span> | - New applications get a default system theme assigned automatically<br>- Custom theme properties stored as jsonb<br>- Applying a theme updates draft_theme_id without touching published_theme_id | 5 | Medium |
| **Implement custom JS library add/remove**<br><span style="font-size:90%">As a user, I want to import a JS library and use it in my bindings, so that I can use third-party functionality in my logic.</span> | - Library metadata (name/version/url/defs) stored once and referenced, not duplicated per application<br>- Add/remove endpoints match current .../libraries/{id}/add and /remove contracts | 5 | Medium |

### Feature: Snapshots & Static URLs
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement application snapshots**<br><span style="font-size:90%">As a user, I want the system to take a snapshot before risky operations and let me restore from one, so that I have a safety net.</span> | - Snapshot stores a compressed artifact JSON, not live entity rows<br>- Restore endpoint round-trips correctly against the snapshot format<br>- Old snapshots are prunable | 5 | Low |
| **Implement static URL slugs**<br><span style="font-size:90%">As a user, I want to set a custom human-readable URL for my published application, so that I can share a memorable link.</span> | - Slug uniqueness enforced instance-wide<br>- Verify-availability endpoint matches current contract<br>- Published-app lookup by slug is a single indexed query | 3 | Low |

### Feature: Publish (Atomic)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement publish as a single transaction**<br><span style="font-size:90%">As a user, I want to publish my application and have every page/action/theme change go live together, so that my published app is never left in a half-updated state.</span> | - All published_* column updates happen inside one BEGIN/COMMIT<br>- Soft-deleted pages/actions are hard-deleted as part of the same transaction<br>- Matches Database per Service §3's publish SQL | 8 | High |
| **Prove publish atomicity under fault injection**<br><span style="font-size:90%">As a tech lead, I want a test that kills the process mid-publish and verifies the database shows either the full old state or the full new state, never a mix, so that the 'publish is atomic' claim is verified, not assumed.</span> | - Test injects a failure between two of the publish sub-steps<br>- Post-crash state is verified as fully-rolled-back<br>- This is a go/no-go checkpoint per the roadmap — result recorded | 5 | Highest |
| **Publish the ApplicationPublished event**<br><span style="font-size:90%">As a backend developer, I want the outbox to emit ApplicationPublished on successful publish, so that Realtime and Notifications can react.</span> | - Event carries applicationId, workspaceId, publishedAt, publishedBy<br>- Written in the same transaction as the publish itself | 3 | High |

### Feature: Export/Import Serialization
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the export serializer**<br><span style="font-size:90%">As a backend developer, I want a serializer that strips ids/policies/secrets from every entity type, matching sanitiseToExportDBObject, so that exported applications are portable and don't leak internal state.</span> | - Covers Application, Page, Action, ActionCollection, Theme, CustomJSLib<br>- Output is deterministic — same input always produces the same JSON | 8 | High |
| **Implement the full application export endpoint**<br><span style="font-size:90%">As a user, I want to export my application as a single JSON file, so that I can back it up or move it elsewhere.</span> | - Matches the current GET /applications/export/{id} contract shape<br>- Includes a call to Datasource Service for config (secrets excluded) | 5 | High |
| **Implement the import serializer with id remapping**<br><span style="font-size:90%">As a backend developer, I want an importer that assigns fresh ids to every entity and remaps every internal reference, so that importing doesn't collide with or corrupt existing data.</span> | - Remap table covers pageId, actionId, collectionId, datasourceId, themeId references<br>- Import is transactional — a failure leaves no orphaned rows<br>- Imported datasources arrive isConfigured:false, matching current behaviour | 8 | High |

### Feature: Fork Saga
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement fork orchestration**<br><span style="font-size:90%">As a user, I want to fork an application into another workspace, so that I can build on an existing application without affecting the original.</span> | - Local transaction creates the new application tree with fresh ids<br>- CloneDatasources gRPC call made to Datasource Service for referenced datasources<br>- Success returns the new application id | 8 | High |
| **Implement fork compensation**<br><span style="font-size:90%">As a backend developer, I want a compensating rollback when the datasource clone step fails, so that a partial fork never leaves orphaned data.</span> | - Calls the idempotent DeleteClonedDatasources endpoint on failure<br>- Deletes the newly created application tree<br>- Returns a clear failure reason to the caller, not a generic 500 | 8 | High |

### Feature: Import Saga
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement import orchestration**<br><span style="font-size:90%">As a user, I want to import an application from an uploaded JSON file, so that I can bring in an application built elsewhere.</span> | - Same orchestration and compensation shape as Fork<br>- Handles a source that references datasources not yet present in the target workspace | 8 | Medium |
| **Implement the datasources-needed-for-import endpoint**<br><span style="font-size:90%">As the Angular client, I want to know which datasources an import will need before committing to it, so that I can prompt the user to configure them.</span> | - Matches the current GET .../import/{workspaceId}/datasources contract<br>- Distinguishes datasources that already exist in the target workspace from ones that don't | 3 | Medium |

### Feature: Authorization Projection
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the authz_grants table and event consumer**<br><span style="font-size:90%">As a backend developer, I want a local authorization projection fed by PermissionGrantChanged events, so that every resource read stays a single indexed query with no cross-service call.</span> | - Consumer is idempotent (inbox table) under at-least-once delivery<br>- Projection lag is measured and alerts above the 5s p99 budget<br>- Matches Security & AuthZ §2's design | 8 | High |
| **Implement the Authorized<T>() query helper**<br><span style="font-size:90%">As a backend developer, I want one shared query predicate applied to every protected-resource read and write, so that no endpoint can accidentally skip the authorization join.</span> | - Every existing endpoint is verified to use it, not a hand-rolled check<br>- Zero rows returns 404, matching current behaviour<br>- Covered by the per-service authorization checklist in the security doc | 5 | High |
| **Implement the projection-rebuild admin endpoint**<br><span style="font-size:90%">As an operator, I want to rebuild authz_grants from scratch from Identity's current state, so that projection drift is recoverable without a database restore.</span> | - Rebuild is safe to run against a live, in-use projection table<br>- Progress is observable, not a black box<br>- Documented in the service's runbook | 5 | Medium |

### Feature: Git Integration Surface
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement artifact assembly for commit**<br><span style="font-size:90%">As a backend developer, I want Application Service to assemble the full exportable artifact plus a call to Datasource Service for configs before handing off to Git, so that Git Service never needs to understand the domain model.</span> | - Assembly reuses the export serializer from the Export/Import feature<br>- Datasource configs fetched via GetDatasourceConfigs with secrets excluded | 5 | Medium |
| **Implement Commit/Push/Pull gRPC client calls**<br><span style="font-size:90%">As a user, I want to commit, push and pull my application's git state from the editor, so that I can version-control my application.</span> | - Pull's returned artifact JSON is imported using the same import path as manual import<br>- Commit/Push status is returned synchronously to the caller, matching current UX | 5 | Medium |

---

## DS — Datasource Service
**Component:** `Datasource` &nbsp;·&nbsp; **Roadmap phase:** Phase B

> Manage connection configuration safely and separately from anything that executes — the config half of the plugin story.

### Feature: Datasource & Environment CRUD
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Create the datasource_db schema**<br><span style="font-size:90%">As a backend developer, I want EF Core migrations for datasources, environments and datasource_storages, so that the service has a database matching the target design.</span> | - Matches Database per Service §5.3<br>- No secret material columns present — secret_ref only | 3 | High |
| **Implement datasource CRUD endpoints**<br><span style="font-size:90%">As a user, I want to create, view, rename and delete datasources, so that I can manage my workspace's connections.</span> | - Delete is soft; in-use datasources are handled per the documented deletion contract<br>- Name uniqueness enforced per workspace | 5 | High |
| **Implement default-environment auto-creation**<br><span style="font-size:90%">As a backend developer, I want a default environment created automatically when a workspace is created, so that CE users never have to think about the environment concept.</span> | - Exactly one default environment per workspace<br>- Model still supports adding more environments for a future EE scenario | 2 | Medium |

### Feature: Datasource Storage & Secrets
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Create the datasource_storages schema with secret references**<br><span style="font-size:90%">As a backend developer, I want storage rows that hold non-secret config as jsonb and a secret_ref pointer, so that no credential ever lands in this database.</span> | - Unique (datasource_id, environment_id) constraint<br>- configuration column never contains password/key/token fields | 5 | High |
| **Integrate the secrets manager**<br><span style="font-size:90%">As a backend developer, I want datasource storage writes to route secret fields through ISecretStore, so that credentials are stored, versioned and rotatable outside this database.</span> | - Write path splits config into non-secret and secret halves automatically<br>- Delete of a datasource also deletes its secret via the store<br>- Covered by an integration test against the chosen provider's test/dev instance | 8 | High |
| **Implement update-storage-config endpoint**<br><span style="font-size:90%">As a user, I want to edit a datasource's connection details, so that I can fix or change how it connects.</span> | - Matches the current PUT /datasources/datasource-storages contract<br>- isConfigured and invalids fields update based on validation result | 5 | High |
| **Implement test-connection**<br><span style="font-size:90%">As a user, I want to test a datasource's connection before saving it, so that I know it works before I rely on it.</span> | - Delegates to Execution.TestConnection over gRPC with a timeout<br>- Returns a clear pass/fail with a human-readable reason on failure | 5 | High |

### Feature: Plugin Catalog Replica
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the plugin catalog replica and consumer**<br><span style="font-size:90%">As a backend developer, I want a local read-only plugins table kept current via PluginCatalogUpdated events, so that datasource creation and the connector picker work without calling Execution synchronously.</span> | - Consumer is idempotent<br>- Staleness budget (documented at <1h) is met in testing | 5 | Medium |
| **Implement plugin catalog endpoints**<br><span style="font-size:90%">As the Angular client, I want GET /plugins and GET /plugins/{id}/form, so that the datasource creation UI can render the connector picker and its config form.</span> | - Form schema returned matches what Execution publishes<br>- Response cacheable at the gateway given the low change frequency | 3 | Medium |

### Feature: Config Change Publishing
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Publish DatasourceConfigChanged with a monotonic version**<br><span style="font-size:90%">As a backend developer, I want every config write to publish an event carrying an incrementing version number, so that Query Execution's cache can detect and discard out-of-order delivery.</span> | - Version increments strictly on every write to the same (datasourceId, environmentId)<br>- Verified with a test that delivers events out of order and confirms the cache resolves to the latest | 5 | High |
| **Publish DatasourceDeleted and handle cascading cleanup**<br><span style="font-size:90%">As a backend developer, I want deletion to notify Application (summary cleanup) and Execution (pool/cache eviction), so that no service holds a dangling reference to a deleted datasource.</span> | - Event published in the same transaction as the delete<br>- Application Service's datasource_summaries row is removed on consumption | 5 | Medium |

### Feature: OAuth2 for SaaS Datasources
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement SaaS OAuth2 authorize/token endpoints**<br><span style="font-size:90%">As a user, I want to authorize a SaaS connector (e.g. Google Sheets) via OAuth2, so that I can connect without pasting raw credentials.</span> | - Matches the current /saas/{id}/oauth and /saas/{id}/token contracts<br>- Refresh-token handling verified against at least one real SaaS provider | 8 | Low |

### Feature: CloneDatasources for Fork
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement CloneDatasources gRPC**<br><span style="font-size:90%">As Application Service, I want a call that clones a set of datasources into a target workspace and returns the id remap, so that the Fork saga can complete its second local transaction.</span> | - Clone includes storage config but re-marks isConfigured:false where secrets can't be safely copied across workspace boundaries, per current export behaviour<br>- Returns a complete oldId→newId map | 8 | High |
| **Implement idempotent DeleteClonedDatasources**<br><span style="font-size:90%">As Application Service, I want a compensating call that safely deletes cloned datasources even if called twice, so that fork compensation is safe to retry.</span> | - Second call on already-deleted ids is a no-op, not an error<br>- Deletes the associated secrets too | 5 | High |

### Feature: Authorization Projection
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the authz_grants table and event consumer**<br><span style="font-size:90%">As a backend developer, I want the same local authorization projection pattern as Application Service, so that datasource reads stay a single indexed query.</span> | - Shares the BuildingBlocks.Authorization graph implementation<br>- Consumer is idempotent and lag-monitored | 5 | High |

---

## EXEC — Query Execution Service
**Component:** `Execution` &nbsp;·&nbsp; **Roadmap phase:** Phase C

> Run all 25 connector types in isolated, resource-capped worker pools. The largest reliability improvement in the programme.

### Feature: gRPC Router & Audit
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Create the execution_db schema**<br><span style="font-size:90%">As a backend developer, I want the plugins, plugin_forms, datasource_config_cache and partitioned execution_audit tables, so that the service has somewhere to route from and record to.</span> | - execution_audit is partitioned by time as designed<br>- Matches Database per Service §5.4 | 5 | High |
| **Implement the ExecuteAction gRPC router**<br><span style="font-size:90%">As Application Service, I want a stateless endpoint that dispatches to the correct worker pool by plugin type, so that query execution has a single entry point regardless of connector.</span> | - Router itself contains no connector logic<br>- Every call carries and enforces a deadline | 5 | High |
| **Implement the datasource config cache consumer**<br><span style="font-size:90%">As a backend developer, I want a local jsonb cache kept current via DatasourceConfigChanged, with staleness detection, so that execution reads config without a network hop on the hot path.</span> | - Version field used to detect and resolve staleness against the caller's expectation<br>- Synchronous refetch path exists for the rare stale-cache case | 5 | High |

### Feature: Worker Process Isolation Model
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the worker dispatch model**<br><span style="font-size:90%">As a platform engineer, I want each worker pool as a separately deployable container with a defined IPC/gRPC contract to the router, so that connector families can scale and fail independently.</span> | - Router-to-worker contract documented and versioned<br>- Adding a new worker pool requires no router code change beyond configuration | 8 | High |
| **Enforce CPU, memory and wall-clock limits**<br><span style="font-size:90%">As a platform engineer, I want container-level resource limits on every worker pool, with the deadline propagated from the gRPC call, so that a runaway connector can't degrade the platform.</span> | - Limits configurable per worker pool<br>- A hostile connector test suite (infinite loop, huge allocation, huge result set) is contained without affecting other pools — this is a go/no-go checkpoint | 8 | Highest |
| **Enforce network egress policy on workers**<br><span style="font-size:90%">As a security engineer, I want workers restricted to customer-system egress only, with no path to internal services or the secrets manager, so that a compromised connector can't pivot into the platform.</span> | - Network policy verified with a test that attempts a call to an internal service from inside a worker and confirms it's blocked<br>- Documented in the service's runbook | 5 | High |
| **Enforce result-size caps**<br><span style="font-size:90%">As a platform engineer, I want a hard cap on response size at the router, so that a query returning an enormous result set fails cleanly instead of exhausting memory.</span> | - Cap is configurable<br>- Failure returns a clear, actionable error to the user, not a generic timeout | 3 | Medium |

### Feature: SQL Connector Worker Pool
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Port the Postgres connector**<br><span style="font-size:90%">As a user, I want to run queries against a Postgres datasource, so that I can build applications on Postgres data.</span> | - Golden-file suite captured from postgresPlugin passes<br>- Connection pooling scoped per (datasourceId, environmentId) at the worker-pool level | 8 | High |
| **Port the MySQL connector**<br><span style="font-size:90%">As a user, I want to run queries against a MySQL datasource, so that I can build applications on MySQL data.</span> | - Golden-file suite captured from mysqlPlugin passes | 8 | High |
| **Port the remaining SQL connectors**<br><span style="font-size:90%">As a user, I want MSSQL, Oracle, Redshift, Snowflake and Databricks support, so that the platform covers the same relational connector breadth as today.</span> | - Each has its own golden-file suite, ported in this order by observed traffic<br>- No connector ships without its suite passing | 13 | Medium |

### Feature: HTTP Connector Worker Pool
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Port the REST API connector**<br><span style="font-size:90%">As a user, I want to call arbitrary REST APIs as a datasource, so that I can integrate with external HTTP services.</span> | - Golden-file suite captured from restApiPlugin passes, including auth modes and pagination<br>- Handles the current SSRF-relevant hint messages (localhost warnings) equivalently | 8 | High |
| **Port the GraphQL connector**<br><span style="font-size:90%">As a user, I want to query GraphQL APIs as a datasource, so that I can integrate with GraphQL services.</span> | - Golden-file suite captured from graphqlPlugin passes | 5 | Medium |
| **Port the SaaS connector framework**<br><span style="font-size:90%">As a user, I want OAuth2-backed SaaS connectors (e.g. Google Sheets) to work end-to-end, so that I can connect to SaaS platforms, not just raw HTTP.</span> | - Integrates with Datasource Service's SaaS OAuth2 endpoints<br>- Golden-file suite for at least one SaaS connector passes | 5 | Medium |

### Feature: JS Worker
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the sandboxed JS worker**<br><span style="font-size:90%">As a user, I want my JS Object functions to execute safely on the server when used as datasource-independent logic, so that server-side JS behaves correctly without risking the platform.</span> | - Jint sandbox with no fs/net/process access, memory and statement-count limits<br>- A fresh worker is used per execution, never reused across users or requests | 8 | High |
| **Verify no state leaks between JS executions**<br><span style="font-size:90%">As a security engineer, I want a test that writes a global variable in one execution and asserts its absence in the next, so that one user's JS execution can never affect another's.</span> | - Test passes consistently across repeated runs<br>- Documented as a permanent regression test, not a one-off check | 5 | High |

### Feature: NoSQL / Cloud / AI Worker Pools
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Port the NoSQL connectors**<br><span style="font-size:90%">As a user, I want MongoDB, Redis and Elasticsearch support, so that I can build applications on document/key-value/search data.</span> | - Golden-file suites pass for each, ported in traffic order | 8 | Medium |
| **Port the remaining NoSQL connectors**<br><span style="font-size:90%">As a user, I want DynamoDB, Firestore and ArangoDB support, so that the platform covers the same NoSQL breadth as today.</span> | - Golden-file suites pass for each | 8 | Low |
| **Port the cloud/storage connectors**<br><span style="font-size:90%">As a user, I want S3, Google Sheets, Lambda and SMTP support, so that I can integrate with common cloud services.</span> | - Golden-file suites pass for each<br>- S3 connector reused later for object-storage-backed assets if applicable | 8 | Medium |
| **Port the AI connectors**<br><span style="font-size:90%">As a user, I want OpenAI, Anthropic, Google AI and the Appsmith AI connectors to work as datasources, so that I can build AI-powered applications.</span> | - Golden-file suites pass for each<br>- Streaming responses (if used) are handled correctly through the gRPC contract | 8 | Medium |

### Feature: Connection Pooling & Introspection
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement worker-tier connection pooling**<br><span style="font-size:90%">As a platform engineer, I want pools scoped per (datasourceId, environmentId) at the worker-pool level, not per API replica, so that the platform has a real, global connection budget per datasource for the first time.</span> | - N execution replicas no longer means N× connections against a customer database<br>- Stale-connection eviction and retry-once logic ported from DatasourceContextServiceCEImpl | 8 | High |
| **Implement GetStructure schema introspection**<br><span style="font-size:90%">As a user, I want to see my datasource's schema in the editor, so that I can browse tables/collections while writing a query.</span> | - Result cached in Datasource Service's datasource_structures on the caller's behalf<br>- Matches the current GET /datasources/{id}/structure contract | 5 | Medium |
| **Implement the Trigger endpoint**<br><span style="font-size:90%">As a user, I want dynamic dropdown data (e.g. list of sheets, list of buckets), so that the editor can offer contextual choices while I configure an action.</span> | - Matches current POST /datasources/{id}/trigger and /plugins/{id}/trigger contracts | 5 | Low |

### Feature: Golden-File Test Harness
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Build the golden-file replay tooling**<br><span style="font-size:90%">As a tech lead, I want a test harness that loads captured Java-plugin request/response pairs and replays them against the .NET connector, so that every connector port has an automated, objective pass/fail bar.</span> | - Harness runs as part of each connector's test suite in CI<br>- Failures show a clear diff between expected and actual output | 8 | High |
| **Gate connector merges on golden-file suite results**<br><span style="font-size:90%">As a tech lead, I want CI to block merging a connector port without its golden-file suite passing, so that no connector reaches production without verified fidelity.</span> | - Gate is enforced at the CI level, not just by convention<br>- Documented in the contribution guide for the Execution team | 3 | High |

---

## GIT — Git Versioning Service
**Component:** `Git` &nbsp;·&nbsp; **Roadmap phase:** Phase D

> Version-control application artifacts against a real git remote — commit, push, pull, merge, branch, and deploy-key management.

### Feature: Repository Model
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Create the git_db schema**<br><span style="font-size:90%">As a backend developer, I want tables for repositories, branches, deploy keys and git profiles, so that the service has somewhere to track git state.</span> | - Matches Database per Service §5.5<br>- git_repositories.artifact_id is unique — one repo per artifact | 5 | High |
| **Implement the LibGit2Sharp wrapper and working-tree management**<br><span style="font-size:90%">As a backend developer, I want a clean abstraction over LibGit2Sharp with a managed working-tree volume per repository, so that git operations have a reliable local execution environment.</span> | - Working trees are placed on a documented persistent volume path<br>- Wrapper exposes only the operations the service actually needs, not the full LibGit2Sharp surface | 8 | High |

### Feature: Byte-Stable Serialization
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the artifact JSON to file-tree serializer**<br><span style="font-size:90%">As a backend developer, I want a serializer producing the same pages/queries/jsobjects/datasources/jslibs directory layout as today, so that existing git-connected applications continue to diff sensibly after migration.</span> | - JS Objects still write as real .js files, not embedded JSON<br>- Directory names match GitDirectoriesCE exactly | 8 | High |
| **Prove byte-identical serialization against the Java baseline**<br><span style="font-size:90%">As a tech lead, I want a golden test comparing .NET serializer output to Java output across the application corpus, so that a user's first commit after migration is not a whole-repo diff.</span> | - Zero byte differences across the corpus, or every difference explicitly triaged and fixed<br>- This is a go/no-go checkpoint per the roadmap — blocking for migrating git-connected applications | 8 | Highest |

### Feature: Connect / Commit / Push / Pull
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement connect-to-git**<br><span style="font-size:90%">As a user, I want to connect my application to a git remote, so that I can start version-controlling it.</span> | - Generates an SSH keypair via the secrets manager, never storing the raw private key in this database<br>- Tests connectivity before confirming the connection succeeded | 5 | High |
| **Implement commit with the Redis lock**<br><span style="font-size:90%">As a user, I want to commit my current changes with a message, so that I have a checkpoint I can return to.</span> | - Redis lock keyed on baseArtifactId carried over unchanged, with retry/backoff<br>- Commit fails cleanly and releases the lock if serialization or git-add fails partway | 8 | High |
| **Implement push**<br><span style="font-size:90%">As a user, I want to push my committed changes to the remote, so that my team and the remote history stay in sync.</span> | - Uses the SSH deploy key resolved from the secrets manager at call time, never persisted in memory longer than needed<br>- Push failure (auth, network) returns a specific, actionable error | 5 | High |
| **Implement pull**<br><span style="font-size:90%">As a user, I want to pull the latest remote changes into my application, so that I can incorporate changes made elsewhere.</span> | - Returns artifact JSON to Application Service, which performs the actual database import<br>- Git Service itself never writes application data | 5 | High |

### Feature: Branching
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement create/checkout/delete branch**<br><span style="font-size:90%">As a user, I want to create, switch between and delete branches, so that I can work on multiple versions of my application in parallel.</span> | - Branch creation triggers the entity-tree duplication contract with Application Service<br>- Checkout switches the active branched ids the client works against | 8 | High |
| **Implement the entity-tree duplication contract**<br><span style="font-size:90%">As a backend developer, I want a well-defined call from Git to Application Service that duplicates an application's entity tree for a new branch, so that branch creation produces a correct, independent copy every time.</span> | - Every page/action/collection gets a fresh id but shares the base_id of its source<br>- Contract documented in the service's Contracts project | 8 | High |
| **Implement branch protection**<br><span style="font-size:90%">As a user, I want to mark a branch as protected against direct commits, so that my team can enforce a review process on important branches.</span> | - Protected-branch commit attempts are rejected with a clear message<br>- Matches current GET/POST .../protected contract | 3 | Medium |

### Feature: Merge & Status
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement status**<br><span style="font-size:90%">As a user, I want to see what's changed between my database state and the git working tree, so that I know what a commit would include before making it.</span> | - Diff is computed without mutating either side<br>- Matches current GET /git/status/app/{id} contract | 5 | Medium |
| **Implement merge and conflict detection**<br><span style="font-size:90%">As a user, I want to merge one branch into another and be warned about conflicts before it happens, so that I don't lose work to a silent overwrite.</span> | - Merge-status endpoint runs a dry-run check before the real merge is attempted<br>- Conflicts are reported with enough detail to resolve them | 8 | Medium |
| **Implement discard**<br><span style="font-size:90%">As a user, I want to discard my local database changes and reset to the last commit, so that I can undo experimentation I don't want to keep.</span> | - Resets via the same import path pull uses<br>- Requires explicit confirmation given it's a destructive operation | 3 | Medium |

### Feature: Auto-Commit
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement auto-commit with progress reporting**<br><span style="font-size:90%">As the platform, I want a background job that commits DSL/schema-migration changes automatically, with a pollable progress endpoint, so that users' next manual commit isn't a huge unrelated diff.</span> | - Progress endpoint matches current GET .../auto-commit/progress/app/{id} contract<br>- Job is safe to run concurrently with the artifact-level Redis lock | 8 | Medium |

### Feature: Deploy Keys
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement deploy key management via the secrets manager**<br><span style="font-size:90%">As a backend developer, I want deploy keys generated, stored and resolved exclusively through ISecretStore, so that private keys never land in this database in plaintext, unlike today.</span> | - git_deploy_keys stores only a public key and a private_key_ref<br>- Key resolution happens just-in-time for a push, never cached longer than the call | 5 | High |

---

## RT — Realtime Collaboration Service
**Component:** `Realtime` &nbsp;·&nbsp; **Roadmap phase:** Phase E

> Show who else is in an app, where their cursor is, and push version-available notifications — with the cross-instance backplane today's RTS lacks.

### Feature: SignalR Hub & Auth
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Scaffold the SignalR hub**<br><span style="font-size:90%">As a backend developer, I want a hub project with connection authentication delegated to Identity & Access, so that only authenticated (or correctly anonymous) users can join a presence room.</span> | - Connection auth reuses the same internal-JWT validation as other services<br>- Unauthenticated connections to a private application's room are rejected | 5 | High |

### Feature: Redis Backplane
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Wire the SignalR Redis backplane**<br><span style="font-size:90%">As a backend developer, I want StackExchangeRedis backplane configuration for SignalR, so that presence works correctly across multiple service replicas.</span> | - Configuration documented and covered by an integration test using two hub instances | 5 | High |
| **Verify multi-replica presence**<br><span style="font-size:90%">As a tech lead, I want a test with two service replicas and two simulated browser connections confirming each sees the other's presence, so that we've proven the one capability gap the current RTS process has.</span> | - Test runs in CI against a real Redis instance (Testcontainers)<br>- This is the concrete 'genuine improvement over today' claim — verified, not assumed | 5 | High |

### Feature: Presence & Cursors
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement room join/leave**<br><span style="font-size:90%">As a user, I want to join a presence room scoped to the application and page I'm editing, so that my collaborators can see I'm here.</span> | - Room key is (applicationId, pageId)<br>- Leaving (including disconnect) is detected and broadcast promptly | 3 | Medium |
| **Implement live cursor broadcast**<br><span style="font-size:90%">As a user, I want to see my collaborators' cursor positions on the canvas in real time, so that we can avoid working on the same widget by accident.</span> | - Cursor updates are throttled to avoid flooding the hub<br>- Stale cursors are cleared on disconnect | 5 | Medium |
| **Implement the collaborator indicator list**<br><span style="font-size:90%">As a user, I want to see who else is currently in this application, so that I know who I'm collaborating with.</span> | - List updates on join/leave without a page refresh<br>- Matches the visual behaviour of the current product | 3 | Medium |

### Feature: Version Push
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement version-published push**<br><span style="font-size:90%">As a user, I want to be notified in the editor when someone else publishes a new version while I'm working, so that I don't keep editing against stale assumptions.</span> | - Consumes ApplicationPublished and CommitCreated from the broker<br>- Push doesn't force a reload — it surfaces a dismissible notice | 3 | Medium |

---

## NOTIF — Notifications & Telemetry Service
**Component:** `Notifications` &nbsp;·&nbsp; **Roadmap phase:** Phase F

> Everything that happens after a user action and must never slow it down — email delivery with real retry semantics, alerts, usage and audit.

### Feature: Event Consumers
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Create the notifications_db schema**<br><span style="font-size:90%">As a backend developer, I want tables for templates, delivery_log, alerts and usage_events, so that the service has somewhere to record what it does.</span> | - Matches Database per Service §5.6<br>- delivery_log indexed for the retry-due query | 5 | High |
| **Implement consumers for every domain event**<br><span style="font-size:90%">As a backend developer, I want a consumer registered for every event type published by every other service, so that nothing that should trigger a notification or audit entry is silently missed.</span> | - Every event in the Contracts & Events catalog has a corresponding consumer, even if some just write an audit row<br>- All consumers are idempotent via the inbox pattern | 8 | High |

### Feature: Email Delivery
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Port the email templates**<br><span style="font-size:90%">As a user, I want to receive welcome, verification, password-reset and invite emails matching today's content, so that the transition doesn't change what I see in my inbox.</span> | - All four current templates ported with equivalent content<br>- Templates stored in notification_templates, not hardcoded | 5 | High |
| **Implement SMTP delivery**<br><span style="font-size:90%">As a backend developer, I want an SMTP client integration used by the delivery pipeline, so that templated emails actually get sent.</span> | - Configurable SMTP provider settings<br>- Delivery attempt always writes to delivery_log, success or failure | 5 | High |

### Feature: Retry, DLQ, Delivery Log
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement retry with exponential backoff**<br><span style="font-size:90%">As a backend developer, I want failed deliveries retried automatically with backoff and a capped attempt count, so that a transient SMTP failure doesn't become a permanently lost email.</span> | - Retry schedule matches the documented intervals<br>- next_retry_at drives the retry query, not a fixed polling scan of everything | 8 | High |
| **Implement the dead-letter queue and operator visibility**<br><span style="font-size:90%">As an operator, I want deliveries that exhaust retries to land in a DLQ with a visible status, so that I know when something is actually broken instead of it silently vanishing, unlike today.</span> | - DLQ entries are queryable by an admin endpoint<br>- This directly closes the current system's confirmed swallowed-email-failure gap | 5 | High |

### Feature: Product Alerts
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement product alert CRUD and acknowledgement**<br><span style="font-size:90%">As an instance administrator / user, I want to publish product alerts and let users acknowledge them, so that important announcements reach users and I can see who's seen them.</span> | - Matches current GET /product-alert/alert contract for consumption<br>- Acknowledgement is per-user, per-alert | 3 | Low |

### Feature: Usage & Audit
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement usage event ingestion**<br><span style="font-size:90%">As the platform, I want anonymous usage-pulse-equivalent events ingested and stored, so that DAU/WAU/MAU can still be computed post-migration.</span> | - Matches current POST /usage-pulse contract at the gateway<br>- user_hash is hashed, never the raw email, matching the schema design | 3 | Medium |
| **Implement the append-only audit log**<br><span style="font-size:90%">As an instance administrator, I want every domain event to be recorded in a queryable audit log, so that I can answer 'who changed this permission' or 'who ran this query', which is impossible today.</span> | - Covers permission changes, workspace membership changes, datasource config changes, publish/delete/fork, and query executions<br>- This is a genuinely new capability versus the current system, which has no audit trail at all | 8 | Medium |

---

## GW — API Gateway BFF Hardening
**Component:** `Gateway` &nbsp;·&nbsp; **Roadmap phase:** Phase G

> Harden the composition/BFF layer built in the Phase A skeleton — the endpoints the Angular client actually depends on for a fast, resilient page load.

### Feature: Consolidated API Composition
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the editor consolidated-api endpoint**<br><span style="font-size:90%">As the Angular client, I want one call that fans out to Identity, Application and Datasource in parallel and returns a single payload, so that opening the editor doesn't require ten round trips.</span> | - Each slice carries its own status, matching today's per-slice error isolation<br>- Matches the current GET /consolidated-api/edit contract shape | 8 | High |
| **Implement the viewer consolidated-api endpoint**<br><span style="font-size:90%">As the Angular client, I want the published-app equivalent of the editor composition endpoint, so that viewing a published app is also a single round trip.</span> | - Works correctly for anonymous requests against public applications<br>- Matches the current GET /consolidated-api/view contract shape | 5 | High |
| **Implement the applications-home composition endpoint**<br><span style="font-size:90%">As the Angular client, I want a single call returning everything the home screen needs, so that the home screen loads without a waterfall of requests.</span> | - Matches current GET /applications/home contract | 3 | Medium |
| **Implement the search-entities composition endpoint**<br><span style="font-size:90%">As a user, I want to search across applications, pages, queries and datasources from one box, so that I can find things quickly regardless of type.</span> | - Fans out to the owning services in parallel<br>- Matches current GET /search-entities contract | 5 | Low |

### Feature: Rate Limiting & Policies
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement per-endpoint rate-limit policies**<br><span style="font-size:90%">As a platform engineer, I want documented, configurable limits distinct from the Phase A defaults, tuned per endpoint class, so that high-value endpoints (login, query execution) are protected appropriately without over-throttling ordinary CRUD.</span> | - Policy set is reviewed against the current bucket4j-based limits as a baseline<br>- Limits are adjustable per environment without a redeploy | 5 | Medium |

### Feature: Asset Proxying
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the asset proxy/redirect**<br><span style="font-size:90%">As a user, I want asset URLs to resolve through the gateway to object storage, so that existing links and embedded references (e.g. in exported apps) keep working.</span> | - GET /assets/{id} redirects or streams from object storage<br>- Remains on the anonymous allow-list, matching current behaviour | 3 | Medium |

### Feature: Anonymous Allow-List
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the configurable public-route allow-list**<br><span style="font-size:90%">As a security engineer, I want the gateway's anonymous-access route list to be data-driven and match the current permitAll set exactly, so that public application viewing keeps working without accidentally exposing a private route.</span> | - List reviewed line-by-line against the current SecurityConfig permitAll set<br>- Covered by a test asserting every listed route is reachable without a session and nothing else is | 3 | High |

### Feature: Contract Parity Verification
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Run response-diff testing against the live Java system**<br><span style="font-size:90%">As a tech lead, I want an automated comparison of the new gateway's composition-endpoint payloads against the Java system's, for a representative set of applications, so that we know the client-facing contract hasn't silently drifted before cutover.</span> | - Diff run covers the editor, viewer and home composition endpoints<br>- Zero unexplained differences, or every difference explicitly signed off as an intentional improvement<br>- This is a go/no-go checkpoint per the roadmap | 8 | Highest |

---

## NG — Angular Client
**Component:** `Client` &nbsp;·&nbsp; **Roadmap phase:** Parallel, C0-C7

> Replace the React SPA. Largest single workstream in the programme — sized and staffed accordingly, running in parallel from Phase A.

### Feature: Workspace Skeleton (C0)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Scaffold the Angular workspace**<br><span style="font-size:90%">As a frontend developer, I want a standalone-components, zoneless Angular workspace matching the target project layout, so that the team has a consistent foundation to build every feature on.</span> | - Standalone components throughout, no NgModules<br>- Zoneless change detection configured from day one, not retrofitted later | 5 | High |
| **Implement the HTTP layer and interceptors**<br><span style="font-size:90%">As a frontend developer, I want one HttpClient configuration with credentials, CSRF, envelope-unwrap and correlation-id interceptors, so that every feature gets consistent API behaviour for free.</span> | - withCredentials true on every call<br>- responseMeta envelope unwrapped once, not per feature<br>- Mirrors today's single Api.ts pattern | 5 | High |
| **Implement auth flow screens**<br><span style="font-size:90%">As a user, I want login, signup and password-reset screens, so that I can get into the application.</span> | - Wired against the real Identity & Access endpoints, not mocks, by the end of this story<br>- Matches current form validation behaviour | 8 | High |
| **Implement the routing shell**<br><span style="font-size:90%">As a frontend developer, I want lazy-loaded routes separating the editor and viewer bundles, so that a published-app visitor never downloads the editor's code.</span> | - Viewer bundle size measured and tracked against the stated performance target<br>- Editor route only loads after authentication | 5 | High |

### Feature: Evaluation Engine Port (C1)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Copy the evaluation engine, AST and DSL packages**<br><span style="font-size:90%">As a frontend developer, I want workers/Evaluation, packages/ast and packages/dsl copied into the Angular project with no logic changes, so that years of correctness work in the binding evaluator isn't lost or rewritten.</span> | - No React dependency remains in the copied code<br>- Package boundaries preserved so future upstream fixes can still be tracked | 5 | High |
| **Implement the EvaluationService wrapper**<br><span style="font-size:90%">As a frontend developer, I want an Angular service that owns the Web Worker and exposes its output as signals, so that the rest of the app can consume evaluation results idiomatically.</span> | - dataTree and evalErrors exposed as signals<br>- postMessage contract with the worker is unchanged from the ported code | 5 | High |
| **Verify evaluation parity against the React baseline**<br><span style="font-size:90%">As a tech lead, I want the ported engine's output compared against the current React app's output across the full DSL corpus, so that we know the port didn't silently change binding behaviour before any UI is built on top of it.</span> | - Diff run covers every fixture in the corpus<br>- This is the single highest-priority checkpoint in the whole client plan — result recorded before Stage C2 begins | 8 | Highest |

### Feature: Published-App Viewer (C2)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the DSL renderer host**<br><span style="font-size:90%">As a frontend developer, I want a widget-registry-driven renderer using NgComponentOutlet to turn a DSL tree into a live page, so that published applications can actually be viewed.</span> | - Registry is keyed on the DSL type string, matching the target design<br>- Renders a nested widget tree of arbitrary depth | 8 | High |
| **Implement on-load action execution**<br><span style="font-size:90%">As a frontend developer, I want the viewer to execute the server-computed onload plan against the real API on page load, so that a published app's initial data loads correctly.</span> | - Waves execute in order; actions within a wave run in parallel<br>- Errors in one action don't block unrelated actions in the same wave | 8 | High |
| **Implement binding resolution and re-render**<br><span style="font-size:90%">As a user, I want widget properties bound to data to update when that data changes, so that the app feels alive and reactive, not static.</span> | - Only affected widgets re-render on a data change, not the whole tree<br>- Verified against the performance target for canvas re-render latency | 8 | High |

### Feature: Core Widget Set (C3)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Port Table, Text, Button, Input and Select widgets**<br><span style="font-size:90%">As a user, I want the five most commonly used widgets available in the new client, so that the majority of real applications can already be viewed.</span> | - Each has a passing visual-regression comparison against the React version<br>- Property-pane config ported unchanged from the React widget's config directory | 13 | High |
| **Port Container, Form, List, Modal and Tabs widgets**<br><span style="font-size:90%">As a user, I want the core layout and structural widgets available, so that applications with real structure, not just flat pages, render correctly.</span> | - Container/List nesting behaviour matches the React version<br>- Modal and Tabs state behaviour verified against the corpus | 13 | High |
| **Port Chart and JSON Form widgets**<br><span style="font-size:90%">As a user, I want data-visualisation and dynamic-form widgets available, so that applications using them for reporting or complex input work correctly.</span> | - Chart types supported match the current widget's documented set<br>- JSON Form schema-driven rendering verified against a corpus sample | 8 | Medium |
| **Port the property-pane config for every ported widget**<br><span style="font-size:90%">As a frontend developer, I want each widget's config/ directory (property pane schema, defaults, derived properties, autocomplete) carried over unchanged, so that the editing experience for these widgets matches today's without re-authoring the config.</span> | - Config files are copied, not re-derived from the rendering component<br>- Derived properties (e.g. Table1.selectedRow) verified to compute correctly | 8 | High |

### Feature: Editor IDE (C4)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement canvas drag/drop/resize editing**<br><span style="font-size:90%">As a user, I want to add, move and resize widgets on the canvas, so that I can actually build an application, not just view one.</span> | - Geometry maths (topRow/leftColumn/grid units) ported exactly from the DSL semantics<br>- Works with at least the fixedlayout and autolayout systems per the scope decision | 13 | High |
| **Implement the property pane framework**<br><span style="font-size:90%">As a user, I want a dynamic form that renders the correct controls for the selected widget's properties, so that I can configure any widget without the editor needing widget-specific UI code.</span> | - Driven entirely by the ported property-pane config, not hardcoded per widget<br>- Reactive forms, typed, matching the target design's stated approach | 13 | High |
| **Implement the entity explorer**<br><span style="font-size:90%">As a user, I want a tree view of my application's pages, queries, JS objects and datasources, so that I can navigate my application's structure.</span> | - Reflects create/rename/delete in real time<br>- Matches the current entity-explorer's grouping and iconography intent | 8 | High |
| **Implement the query editor and JS editor**<br><span style="font-size:90%">As a user, I want to write and test queries and JS Object functions in the editor, so that I can build my application's logic.</span> | - Query editor supports running a query and seeing the result inline<br>- JS editor has at minimum syntax highlighting and inline error surfacing | 13 | High |
| **Implement the debugger panel**<br><span style="font-size:90%">As a user, I want to see evaluation errors and failed query executions in one place, so that I can diagnose a broken binding without guessing.</span> | - Surfaces evaluation errors from EvaluationService directly<br>- Matches the diagnostic value of the current debugger, per Frontend Architecture §6 | 8 | High |
| **Implement undo/redo**<br><span style="font-size:90%">As a user, I want to undo and redo my canvas edits, so that I can recover from a mistake without starting over.</span> | - Ported from the ReplayDSL approach, not reimplemented from scratch<br>- Works across widget add/move/resize/delete and property changes | 5 | Medium |

### Feature: Remaining Widgets (C5)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Port the remaining widget tail**<br><span style="font-size:90%">As a user, I want the widgets outside the core set to become available, in order of real usage frequency, so that applications that depend on less common widgets are eventually fully supported.</span> | - Priority order is driven by a scan of the real application corpus, not intuition<br>- Each widget carries its own visual-regression check before being marked done | 21 | Medium |

### Feature: Peripheral UI (C6)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the git UI**<br><span style="font-size:90%">As a user, I want to connect, commit, push, branch and merge from the editor, so that I can use git version control without leaving the app.</span> | - Covers the same operations as the current Git panel<br>- Status and merge-conflict UI clearly surfaces what the Git Service reports | 13 | Medium |
| **Implement the datasource UI**<br><span style="font-size:90%">As a user, I want connection forms per connector type, driven by the plugin catalog's form schema, so that I can create and configure any of the 25 datasource types.</span> | - Form renders dynamically from the schema, not hardcoded per connector<br>- Test-connection button wired to the real endpoint | 13 | Medium |
| **Implement workspace, settings and admin screens**<br><span style="font-size:90%">As a user / administrator, I want to manage workspace members, roles and instance settings from the UI, so that I don't need direct API access to administer the platform.</span> | - Member/role management wired to the real Identity endpoints<br>- Instance settings restricted to instance admins in the UI, matching the backend enforcement | 8 | Medium |
| **Implement the template gallery**<br><span style="font-size:90%">As a user, I want to browse and instantiate application templates, so that I can start from a working example instead of a blank canvas.</span> | - Matches current /app-templates browsing and import contracts | 5 | Low |

### Feature: Hardening (C7)
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Port the Cypress/Playwright suites**<br><span style="font-size:90%">As a QA engineer, I want the existing E2E suites ported to run against the new Angular client, so that the acceptance gate for each phase is real, not assumed.</span> | - Suites run against the new client in CI, not just locally<br>- A phase's slice of the suite is green before that phase is marked done, per the roadmap's definition of done | 13 | High |
| **Build the visual regression suite**<br><span style="font-size:90%">As a QA engineer, I want automated screenshot comparison between the React and Angular renderers across the application corpus, so that rendering differences are caught systematically, not by spot-checking.</span> | - Runs against every fixture in the corpus<br>- Flags differences above a defined pixel-diff threshold for review | 8 | High |
| **Run the performance hardening pass**<br><span style="font-size:90%">As a frontend developer, I want the app measured and tuned against the stated FCP/TTI/re-render targets, so that the rewrite doesn't reproduce the current client's weight.</span> | - Each target in Angular Frontend §7 is measured and either met or has a documented gap and plan<br>- Editor and viewer bundles both measured separately | 8 | Medium |
| **Run an accessibility audit**<br><span style="font-size:90%">As a user relying on assistive technology, I want the editor and viewer to meet a documented accessibility bar, so that the platform is usable by everyone, not just the happy-path user.</span> | - Automated audit (e.g. axe) integrated into CI<br>- Manual keyboard-navigation pass completed for the core editing flow | 5 | Medium |

---

## MIG — Data Migration & Cutover
**Component:** `Migration` &nbsp;·&nbsp; **Roadmap phase:** Pre-cutover

> Move MongoDB data into six PostgreSQL databases once, forward-only, verified at every stage, executed at cutover after repeated dry runs.

### Feature: Migration Tooling
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Scaffold the Appsmith.Migration console application**<br><span style="font-size:90%">As a backend developer, I want a standalone, checkpointed pipeline application, so that the migration can be run, re-run from scratch, or resumed reliably.</span> | - Each stage is independently checkpointed<br>- Re-running from a clean target reproduces the same result | 5 | High |
| **Implement the id-map with deterministic UUIDs**<br><span style="font-size:90%">As a backend developer, I want every Mongo ObjectId mapped to a UUIDv5 derived from (namespace, collection, mongoId), so that repeated dry runs produce identical ids and are directly comparable.</span> | - Id map persisted and kept after migration for support lookups<br>- Every subsequent stage resolves references exclusively through this table | 5 | High |

### Feature: Identity Migration
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the Tenant+Organization to Instance transform**<br><span style="font-size:90%">As a backend developer, I want the two legacy collections merged into one instances row with disagreements logged, so that the collapse decision (ADR / Domain Model §8) is executed correctly, not silently.</span> | - Every field disagreement between the two source documents is logged for review<br>- No data is silently dropped without a logged reason | 5 | High |
| **Implement user, workspace and membership extraction and load**<br><span style="font-size:90%">As a backend developer, I want users, workspaces, workspace_members, roles and role_assignments migrated from their Mongo collections, so that everyone's account and access exists correctly on day one of the new system.</span> | - Row counts match source document counts exactly, per workspace<br>- Bcrypt hashes preserved as-is; migrated to Argon2id lazily on next login, not during migration | 8 | High |

### Feature: Permission Grants Migration
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the policyMap to permission_grants inversion**<br><span style="font-size:90%">As a backend developer, I want every document's embedded policyMap inverted into rows in the authoritative grant table, so that the new system's authorization model starts with a complete, correct picture.</span> | - Streams and batch-COPYs rather than materialising in memory<br>- Unknown permission strings fail the migration loudly, never silently skipped<br>- Covers soft-deleted documents too, so restores remain correct | 13 | High |
| **Verify authz_grants projection rebuild from the migrated grants**<br><span style="font-size:90%">As a tech lead, I want Application and Datasource Service's projection-rebuild endpoints run against the freshly migrated permission_grants table, so that the rebuild mechanism itself is validated as part of migration, not left untested until an incident.</span> | - Rebuilt projections match a sample of expected access decisions<br>- This exercises the same code path that runs in production for projection recovery | 8 | High |

### Feature: Application Data Migration
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement application/page/action/collection/theme extraction and load**<br><span style="font-size:90%">As a backend developer, I want the full editing-domain entity tree migrated with draft/published fields mapped to the new column model, so that every existing application opens correctly in the new system.</span> | - unpublishedX/publishedX pairs map correctly to draft_*/published_* columns per Database per Service §3<br>- Applications never published end up with NULL published_* as designed, not an error | 13 | High |
| **Implement the DSL unescape transform**<br><span style="font-size:90%">As a backend developer, I want Mongo's escaped '.' and '$' characters in DSL keys unescaped during migration, so that the new system never has to reproduce that escaping logic at all.</span> | - Every migrated draft_layout parses as valid, correctly-keyed JSON<br>- The escape/unescape code path is confirmed absent from every service after migration | 5 | High |

### Feature: Secrets Migration
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement the secrets decrypt-and-resplit job**<br><span style="font-size:90%">As a security engineer, I want a hardened, isolated job that decrypts existing datasource secrets and writes them into the secrets manager, so that every credential moves safely out of the database exactly once.</span> | - Job has no network egress except to the secrets manager<br>- No secret value is logged at any point<br>- Old encryption key is rotated out after the job completes | 8 | High |
| **Migrate git deploy keys**<br><span style="font-size:90%">As a security engineer, I want existing SSH private keys moved into the secrets manager with only a reference kept in git_db, so that git operations keep working post-migration without keys ever sitting in plaintext in a database.</span> | - Every existing git-connected application's key is migrated and verified functional against its remote | 5 | High |

### Feature: Git Data Migration
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement git repository/branch metadata extraction**<br><span style="font-size:90%">As a backend developer, I want git_repositories, git_branches and git_profiles populated from the source system, so that branch tracking and protection rules survive the migration.</span> | - Every existing (base_id, ref_name) pair migrates without collision<br>- Duplicate pairs found in source data fail the migration loudly rather than silently picking one | 5 | Medium |
| **Verify clean git status for every migrated repository**<br><span style="font-size:90%">As a tech lead, I want git status run against every migrated, git-connected application's working tree, so that the byte-stable-serialization guarantee is proven per-repository, not just on the golden test corpus.</span> | - A dirty tree for any repository blocks cutover for that application<br>- Results logged per repository for the cutover go/no-go decision | 5 | High |

### Feature: Verification & Dry Runs
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Implement row-count and referential-integrity verifiers**<br><span style="font-size:90%">As a backend developer, I want automated checks comparing source and target row counts and confirming zero orphaned foreign-key references, so that structural correctness is verified mechanically, not by spot-checking.</span> | - Runs as a migration stage, not a manual follow-up<br>- Any failure aborts the run | 5 | High |
| **Implement the permission-fidelity sampling test**<br><span style="font-size:90%">As a security engineer, I want a sample of (user, resource, permission) triples evaluated against both the Java system and the new system, requiring a 100% match, so that the migration hasn't silently changed who can access what.</span> | - Sample size and selection method documented<br>- Any mismatch blocks the migration — this is a go/no-go checkpoint | 8 | Highest |
| **Implement the application round-trip export diff verifier**<br><span style="font-size:90%">As a backend developer, I want every application exported from both systems, normalised and diffed, so that structural equality is proven per application, not assumed from aggregate checks.</span> | - Diff tool ignores expected differences (ids) and flags everything else<br>- Report is reviewable per application, not just pass/fail overall | 8 | High |
| **Execute the three dry runs**<br><span style="font-size:90%">As a tech lead, I want a correctness dry run, a completeness dry run, and a timed rehearsal dry run, in that order, so that the real cutover has no surprises.</span> | - Dry run 1 fixes transformer bugs with no time pressure<br>- Dry run 2 achieves all-green verification<br>- Dry run 3 measures wall-clock duration and determines the acceptable freeze window | 13 | High |

### Feature: Cutover Execution
| Story | Acceptance Criteria | Pts | Priority |
|---|---|---|---|
| **Automate the cutover runbook**<br><span style="font-size:90%">As an operator, I want the freeze/snapshot/migrate/verify/switch sequence scripted, not run by hand from a checklist, so that cutover is repeatable and low-risk under pressure.</span> | - Every step in the runbook (Data Migration §9) has a corresponding automated action or explicit manual gate<br>- Aborting mid-sequence leaves the system in a known, documented state | 8 | High |
| **Run a rollback drill**<br><span style="font-size:90%">As an operator, I want a rehearsal of switching the ingress back to the Java system, so that if cutover needs to be aborted, the team has actually done it before, not just planned it.</span> | - Drill executed against a non-production environment<br>- Time-to-rollback measured and documented | 5 | Medium |

---

**Grand total: 217 user stories · 1328 story points** (Fibonacci: 1,2,3,5,8,13,21). Points are a starting estimate for sprint planning, not a commitment — recalibrate after the team's first few sprints on this codebase.

[<- Risks & ADRs](../03-execution/04-risks-and-adrs.md) · [Index](../README.md)
