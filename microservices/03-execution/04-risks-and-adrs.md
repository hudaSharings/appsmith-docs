# Execution — Risks & Architecture Decision Records

[← Index](../README.md) · [← Data Migration](03-data-migration.md)

---

## Part 1 — Architecture Decision Records

Each ADR states the decision, the alternatives, and what would make us revisit it. **A decision you can't revisit isn't recorded properly.**

---

### ADR-001 — Merge Auth, User and Workspace into one Identity & Access service

**Status:** Accepted · **Supersedes:** the original HLD's services 2, 3 and 4

**Context.** The HLD proposed separate Authentication, User and Workspace services. In the current code, signup writes a user *and* a workspace *and* three roles *and* a role assignment in one flow; invite writes a user *and* a role assignment; workspace creation writes a workspace *and* three roles.

**Decision.** One service owning users, credentials, workspaces, memberships, roles and grants.

**Alternatives rejected.** Three services — would turn signup, invite and workspace-create into three-participant sagas for data that has one lifecycle and one team.

**Consequences.** Signup/invite/workspace-create stay single local transactions. Security isolation comes from module boundaries and separate credential tables, not from a network boundary. The service is larger than each of the three would be.

**Revisit if.** Authentication needs to scale independently (e.g. an SSO gateway serving other products), or a separate team takes ownership of identity with a genuinely different release cadence.

---

### ADR-002 — Keep Application, Page, Action, Theme in one service

**Status:** Accepted · **The load-bearing decision**

**Context.** These are edited, published, forked and deleted together. The current code has zero cross-collection joins; splitting them would *create* the join problem that doesn't exist. Publish today is five unguarded writes.

**Decision.** One Application Service owning applications, pages, layouts, actions, action collections, themes, JS libraries and snapshots.

**Alternatives rejected.** A Page service and an Action service — manufactures cross-service consistency problems for a purely internal editing surface with no independent scaling need, and turns publish into a five-participant saga.

**Consequences.** Publish is one transaction (better than today). Fork/import is one local transaction plus one compensating call. The service has the largest endpoint surface (~75) — a team-sizing problem, not an architecture problem.

**Revisit if.** Editing traffic and published-app-viewing traffic diverge enough to need independent scaling. Even then, prefer splitting *read* (viewer) from *write* (editor) over splitting by entity type.

---

### ADR-003 — Separate Datasource config from Query Execution

**Status:** Accepted

**Context.** Today all 25 connectors run in the API process with no sandbox, no CPU/memory cap and no enforced timeout. A runaway connector degrades the whole platform.

**Decision.** Datasource Service owns configuration; Query Execution Service owns running things, in isolated per-connector-family worker processes.

**Alternatives rejected.** One combined service (the HLD's position) — keeps the current risk. `AssemblyLoadContext` isolation — reproduces PF4J's situation, since only a process boundary gives enforceable limits.

**Consequences.** One extra network hop on query execution, mitigated by an event-replicated config cache. A flaky customer database can no longer affect config editing. Per-connector-family scaling and metrics become possible.

**Revisit if.** The hop measurably hurts p99 execution latency in a way the cache can't absorb — but note the hop is *inside* the datacentre while the query itself crosses the internet.

---

### ADR-004 — Replicate authorization; do not call a permission service

**Status:** Accepted · **The highest-consequence decision**

**Context.** Every document today embeds its own ACL (`policyMap`), so an authorization check costs **zero** extra round trips — it's a clause in the query you were already making. This is the most frequent operation in the system.

**Decision.** Identity & Access owns the truth (`permission_grants`). Each resource-owning service keeps a local `authz_grants` projection fed by events, and joins it in every query.

**Alternatives rejected.** A synchronous permission call per read — converts a 0-hop check into 1+ network hops on the hottest path and makes Identity a hard availability dependency for everything. Embedding grants on each entity row — reproduces today's write amplification without the benefit of an explicit event stream.

**Consequences.** Read performance matches today. An eventual-consistency window on *role-definition* changes (p99 < 5s budget). Membership changes and hard revocations take effect immediately via session and role-set-cache invalidation. Every service needs a projection rebuild path.

**Revisit if.** The measured projection lag exceeds budget under real load, or a compliance requirement demands strictly synchronous revocation for all cases (not just member removal).

---

### ADR-005 — Keep cookie sessions; don't move to browser bearer tokens

**Status:** Accepted

**Context.** The current client uses a Redis-backed cookie session + CSRF token, `withCredentials: true` everywhere. The Angular rewrite is already large.

**Decision.** Cookie session at the gateway; short-lived internal JWTs minted per request for service-to-service calls.

**Alternatives rejected.** Browser-held bearer tokens — adds a token-refresh state machine to the client rewrite and puts a stealable credential in browser storage. A hybrid — worst of both.

**Consequences.** Sessions are revocable instantly, which is what makes ADR-004's eventual consistency safe. The gateway must be a single origin. Non-browser API clients need a separate token-issuance path (not required for v1).

**Revisit if.** Third-party API clients or a public API become a requirement.

---

### ADR-006 — RabbitMQ + MassTransit, not Kafka

**Status:** Accepted

**Context.** The current system has **no broker at all**. Appsmith's deployment ethos is self-hosted, single-footprint.

**Decision.** RabbitMQ with MassTransit, including the transactional outbox and inbox patterns.

**Alternatives rejected.** Kafka — operational overhead disproportionate to a system whose event model needs no log replay, partitioned ordering at scale, or stream processing. Postgres-as-a-queue — tempting for the lean profile, but reimplements a broker badly. No broker (sync everywhere) — would force ADR-004 into the synchronous model we rejected.

**Consequences.** One new piece of infrastructure to operate. MassTransit gives outbox, retry, DLQ and saga support in one library. Transport abstraction keeps a future Kafka swap contained.

**Revisit if.** Event volume or replay requirements grow beyond what RabbitMQ comfortably serves.

---

### ADR-007 — Two columns for draft/published, not two rows

**Status:** Accepted

**Decision.** `draft_*` and `published_*` columns on one row.

**Alternatives rejected.** Two rows with a state discriminator — doubles row count, makes "the page" ambiguous, complicates every FK. A separate snapshot table — publish becomes a cross-table tree copy and view-mode reads need a join.

**Consequences.** 1:1 mapping from the current model (lowest migration risk). Publish is `UPDATE … SET published_x = draft_x`. View-mode reads are one indexed row fetch. Wider rows; `published_*` is NULL until first publish and every read path must handle that.

**Revisit if.** Multi-version publishing or scheduled publishing becomes a product requirement — that would need a real version table.

---

### ADR-008 — Port the evaluation engine; don't rewrite it

**Status:** Accepted

**Context.** `src/workers/Evaluation` is a reactive dependency-graph evaluator with years of correctness work: cycle detection, incremental re-evaluation, a sandboxed global API, evaluation-version compatibility. It is **framework-independent TypeScript** communicating over `postMessage`.

**Decision.** Copy it into the Angular project and wrap it in a service.

**Alternatives rejected.** Rewriting it in Angular idiom — it has no React dependency, so there is nothing to rewrite *for*, and it is the most likely single way to sink the project. Moving evaluation server-side — a product change, and it would put arbitrary user JS on the server for the canvas as well as for queries.

**Consequences.** Binding semantics are preserved by construction, which is what makes existing applications keep working. The Angular project carries some non-idiomatic TypeScript. Client stage C1 exists specifically to prove this early.

**Revisit if.** C1 shows the engine can't be lifted cleanly — in which case the whole client plan needs re-planning, which is exactly why C1 comes first.

---

### ADR-009 — Parallel build with one cutover, not strangler fig

**Status:** Accepted

**Context.** Language *and* database *and* client are all changing. There is no production installed base requiring incremental zero-downtime migration.

**Decision.** Build the .NET system alongside; migrate data once; cut over.

**Alternatives rejected.** Strangler fig — would require a Mongo-reading .NET data layer *and* a Postgres one, plus shared session and authorization across the language boundary. Big-bang with no reference — loses the ability to verify behaviour against a working system.

**Consequences.** The Java system stays as the behavioural reference until cutover, which is valuable. A cutover event with a write freeze. No incremental production risk, but also no incremental production feedback — mitigated by shipping phases against the E2E suite continuously.

**Revisit if.** A production fleet with real tenants appears before cutover.

---

### ADR-010 — Binding analysis moves in-process; RTS shrinks to presence only

**Status:** Accepted

**Context.** The Java server cannot parse JavaScript, so it makes an HTTP call to the Node RTS service on every layout save to extract binding dependencies.

**Decision.** Application Service parses bindings in-process with Esprima. RTS's remaining responsibility — presence — becomes the SignalR-based Realtime service.

**Alternatives rejected.** Keeping a Node AST sidecar — preserves a network hop and a second runtime for something .NET can do natively.

**Consequences.** One fewer hop on the critical path of every canvas edit. One fewer runtime to operate. Requires verifying that Esprima's output matches the Acorn-based `packages/ast` used by both the current server path and the client evaluator — checkpoint B1.3.

**Revisit if.** Esprima's parse results diverge from the client's in ways that can't be reconciled; fall back to a small AST sidecar with the same contract.

---

## Part 2 — Risk register

Ordered by expected impact. Each risk has an owner-shaped mitigation, not a platitude.

| # | Risk | Impact | Likelihood | Mitigation | Early signal |
|---|---|---|---|---|---|
| R1 | **The client is underestimated.** 708k LOC, 82 widgets, a custom layout engine | Critical | High | Staff it as the largest workstream from Phase A. Cut scope explicitly (widget subset, two layout systems). Port rather than rewrite the evaluator | C1 slips, or the widget corpus scan shows a long tail of widgets in real use |
| R2 | **Evaluation engine can't be lifted cleanly** | Critical | Low–Medium | Client stage C1 proves it headless in the first weeks, before any UI | C1 output diffs against React on the DSL corpus |
| R3 | **Connector fidelity loss** — 25 connectors with years of edge cases | High | High | Golden-file capture from the Java plugins is a **blocking prerequisite** for each connector. Port in traffic order | Golden-file failures on the first connector |
| R4 | **Worker isolation doesn't actually contain a hostile connector** | High | Medium | Checkpoint C.2 tests it deliberately — infinite loop, huge allocation, huge result set — before porting 25 connectors | The hostile-connector test |
| R5 | **Git serialisation isn't byte-stable**, so every user's first commit is a whole-repo diff | High | Medium | Checkpoint D.2: byte-identical golden test against Java output across an application corpus. Blocking for migrating git-connected apps | The serialisation golden test |
| R6 | **Behaviour lost that nobody wrote down** (83 Mongock changelogs, DSL migrations, defaulting rules) | High | High | Read the changelogs before schema finalisation. Characterisation-test against the running Java system. E2E suite as the gate | Divergences found during phase E2E porting |
| R7 | **Projection lag exceeds budget under load**, widening the authorization window | Medium–High | Medium | Lag is a first-class metric with an alert. Load-test in Phase B. Fast-path session kill covers the common revocations | Phase B exit criteria under load |
| R8 | **New infrastructure the team hasn't run** — RabbitMQ, gRPC, outbox, sagas | Medium | Medium | Phase A ships `BuildingBlocks` and a working skeleton before any domain service depends on it. Platform team staffed first | Services forking BuildingBlocks locally |
| R9 | **Application Service becomes a distributed monolith** — large surface, many owners | Medium | Medium | Split the team, not the service. Enforce internal module boundaries with architecture tests. Resist ADR-002 pressure | PRs routinely touching unrelated modules |
| R10 | **Migration window too long** for an acceptable freeze | Medium | Medium | Timed dry run 3 measures it. Mitigations in preference order: parallelise per workspace; pre-migrate immutable data; only then consider CDC | Dry run 3 |
| R11 | **Secrets migration exposure** — the one moment every credential is plaintext in one process | Medium | Low | Isolated hardened job, no egress except the secrets manager, no value logging, rotate the old key afterwards | Design review of the migration job |
| R12 | **Contract drift between services** | Medium | Medium | `.proto` in `Contracts`, diffed in CI; breaking changes need a label + ADR. Consumer-driven contract tests | CI contract diff failures |
| R13 | **Scope creep — "while we're rewriting it, let's also…"** | Medium | High | The DSL is frozen. New features are a separate backlog. Every scope change needs an ADR | Feature requests entering phase plans |
| R14 | **Saga sprawl** — sagas used where a transaction would do | Medium | Medium | Exactly three sagas are sanctioned. A fourth needs an ADR | Code review |
| R15 | **Existing applications render differently in Angular** | Medium | Medium | Visual regression on a real application corpus, React vs Angular, screenshot-diffed. Geometry maths ported exactly | Stage C3 |
| R16 | **Cutover rollback loses post-cutover writes** | Medium | Low | Shadow period, write freeze, smoke test on real data, elevated monitoring for 24h. Java system kept cold | Cutover runbook step 9 |

---

## Part 3 — Open questions

These need answers from product or leadership, not from engineering. Each blocks something specific.

| # | Question | Blocks | Default if unanswered |
|---|---|---|---|
| Q1 | **Which layout systems must v1 support?** (`fixedlayout`, `autolayout`, `anvil`) | Client stage C3–C4 sizing | `fixedlayout` + `autolayout`; drop `anvil` |
| Q2 | **Which widgets are in the v1 set?** | Client C3 | Top ~25 by real-corpus usage |
| Q3 | **Is Enterprise (EE) functionality in scope?** — multi-org, SSO/SAML, audit log UI, branch protection | Identity service scope; whether `instances` needs a real tenancy layer | **CE scope only.** The design leaves room to add multi-org inside Identity without touching other services |
| Q4 | **Are the AI features (`openAiPlugin`, `anthropicPlugin`, AI assistant, AI config) in v1?** | Execution worker pools, Application Service surface | Port the AI connectors; defer the AI-assistant product surface |
| Q5 | **Is there any production data to migrate, or is this greenfield?** | The entire [Data Migration](03-data-migration.md) plan and the cutover risk profile | Assume real data exists; the plan is written for that. If greenfield, most of §4's hard transformations disappear |
| Q6 | **What is the acceptable cutover write-freeze window?** | Whether dry run 3 passes, and whether CDC is needed | 4 hours |
| Q7 | **Self-hosted single-node, cloud/Kubernetes, or both?** | Secrets-manager choice, Postgres topology, Git working-tree storage | Both — the design supports it, but "both" costs more than either |
| Q8 | **Does the published-app viewer need SSR/prerender?** | Client architecture | No SSR for v1; revisit if FCP targets aren't met |
| Q9 | **Which secrets manager?** | Phase B2.2 | An `ISecretStore` abstraction is built regardless; pick the implementation before B2.2 starts |
| Q10 | **Is server-side JS evaluation for the canvas ever wanted?** | Whether the evaluation engine stays client-only long-term | No. Out of scope — it is a product change, not a re-architecture |

---

[← Data Migration](03-data-migration.md) · [Next: Glossary →](../99-reference/01-glossary.md)
