# Target — Database per Service

[← Index](../README.md) · [← Target Topology](02-target-topology.md)

---

**Six PostgreSQL databases. No service ever reads another's schema — enforced by credentials, not convention.**

This is a mechanical translation, not a risky one: the current system has [zero cross-collection joins](../01-current-system/03-domain-model-and-db.md#6-there-are-no-joins--and-that-is-good-news). There are no joins to eliminate. What we're doing is naming ownership that is already implicit.

---

## 1. Ownership map

| Database | Owner | Tables |
|---|---|---|
| `identity_db` | Identity & Access | `instances`, `users`, `user_profiles`, `user_preferences`, `external_logins`, `password_reset_tokens`, `email_verification_tokens`, `workspaces`, `workspace_members`, `roles`, `role_assignments`, `permission_grants`, `outbox` |
| `application_db` | Application | `applications`, `application_pages`, `pages`, `page_layouts`, `actions`, `action_collections`, `themes`, `custom_js_libs`, `application_js_libs`, `application_snapshots`, `static_urls`, `authz_grants`, `datasource_summaries`, `outbox`, `inbox` |
| `datasource_db` | Datasource | `datasources`, `environments`, `datasource_storages`, `datasource_structures`, `plugins` (replica), `authz_grants`, `outbox`, `inbox` |
| `execution_db` | Query Execution | `plugins`, `plugin_forms`, `plugin_templates`, `datasource_config_cache`, `execution_audit`, `outbox`, `inbox` |
| `git_db` | Git Versioning | `git_repositories`, `git_branches`, `git_profiles`, `git_deploy_keys`, `outbox` |
| `notifications_db` | Notifications & Telemetry | `notification_templates`, `delivery_log`, `product_alerts`, `alert_acknowledgements`, `usage_events`, `inbox` |

Realtime Collaboration has no database (Redis backplane only). The Gateway has no database.

**Enforcement:** each service gets a Postgres role that owns exactly its own database and has no grants anywhere else. In a single-cluster deployment this is one `CREATE ROLE` per service. There is no shared-schema escape hatch.

---

## 2. Translation rules: MongoDB → PostgreSQL

Apply these consistently. Deviating per-table is how a schema becomes incoherent.

| Mongo pattern | Postgres treatment | Rationale |
|---|---|---|
| `_id` ObjectId string | `uuid` PK, `gen_random_uuid()` default | ObjectIds are meaningless outside Mongo. Migration maps old→new in a lookup table |
| `createdAt` / `updatedAt` / `createdBy` / `modifiedBy` | `created_at timestamptz`, `updated_at timestamptz`, `created_by uuid`, `updated_by uuid` | Standard audit columns on every table |
| `deletedAt` soft delete | `deleted_at timestamptz NULL` + a **global EF Core query filter** + partial indexes `WHERE deleted_at IS NULL` | Preserves current semantics without every query remembering |
| `deleted` boolean (deprecated) | **Dropped** | Redundant with `deleted_at` |
| Embedded `policyMap` | **Not stored on the entity.** Replaced by an `authz_grants` projection table | See §4 |
| Embedded sub-document with a stable shape (`ApplicationPage`, `GitArtifactMetadata`) | A real table, or columns | Relational where the shape is known |
| Embedded free-form JSON (DSL, `datasourceConfiguration`, `actionConfiguration`, widget props) | **`jsonb` column** | These are genuinely schemaless and are read whole. Do not shred them |
| `unpublishedX` / `publishedX` pairs | **Two columns** on one row: `draft_*` and `published_*` | See §3 |
| `Set<String>` of foreign ids (`User.workspaceIds`) | A join table (`workspace_members`) | It's a many-to-many relation |
| `Set<String>` of scalars with no relation (`invalids`, `messages`) | `text[]` or `jsonb` | No relation to model |
| `@Encrypted` fields | `secret_ref text` pointing at the secrets manager | Ciphertext leaves the database entirely |
| `byte[]` (`Asset.data`) | Object storage; `asset_url text` in the row | Databases are not blob stores |
| Mongo field-name escaping (`.` → `．`) | **Deleted entirely** | `jsonb` has no illegal key characters. Remove the escape/unescape code |
| `baseId` / `refName` git branching | `base_id uuid` + `ref_name text` + unique `(base_id, ref_name)` | Same model, now enforceable |

### The `jsonb` rule

Use `jsonb` when **all** of these hold:
1. The shape is user-defined or evolves independently of the schema.
2. It is read and written as a whole.
3. You do not need to join or aggregate across its internals in SQL.

That is true for: page DSL, action configuration, datasource configuration, theme properties, plugin form schemas, widget property values.

That is **not** true for: permissions, memberships, page ordering, branch metadata. Those become tables.

When in doubt, prefer a table. `jsonb` is for genuine schemalessness, not for avoiding schema design.

---

## 3. The draft/published decision

The current model duplicates every editable entity into `unpublishedX` and `publishedX`. Three options were considered:

| Option | Verdict |
|---|---|
| **A. Two columns on one row** (`draft_layout jsonb`, `published_layout jsonb`) | ✅ **Chosen** |
| B. Two rows with a `state` discriminator | Doubles row count, makes "the page" ambiguous, complicates every FK |
| C. A separate `published_snapshots` table | Cleaner conceptually, but publish becomes a cross-table copy of the whole tree and view-mode reads need a join |

**Option A** is chosen because it maps 1:1 onto the existing model (lowest migration risk), keeps view-mode reads to a single indexed row fetch, and makes publish an `UPDATE ... SET published_x = draft_x` — genuinely atomic in one transaction.

```sql
CREATE TABLE pages (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id      uuid NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    base_id             uuid NOT NULL,          -- stable identity across git branches
    ref_name            text NOT NULL DEFAULT 'main',

    -- draft
    draft_name          text NOT NULL,
    draft_slug          text,
    draft_layout        jsonb NOT NULL DEFAULT '{}'::jsonb,   -- the DSL
    draft_onload_plan   jsonb NOT NULL DEFAULT '[]'::jsonb,   -- layoutOnLoadActions
    draft_is_hidden     boolean NOT NULL DEFAULT false,

    -- published (NULL until first publish)
    published_name      text,
    published_slug      text,
    published_layout    jsonb,
    published_onload_plan jsonb,
    published_is_hidden boolean,

    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    deleted_at          timestamptz,

    CONSTRAINT pages_base_ref_uq UNIQUE (base_id, ref_name)
);

CREATE INDEX pages_app_live_idx ON pages (application_id) WHERE deleted_at IS NULL;
CREATE INDEX pages_base_idx     ON pages (base_id);
```

Publish then becomes, for one application, inside one transaction:

```sql
BEGIN;
  UPDATE pages SET published_name = draft_name,
                   published_layout = draft_layout,
                   published_onload_plan = draft_onload_plan,
                   published_slug = draft_slug,
                   published_is_hidden = draft_is_hidden
   WHERE application_id = $1 AND deleted_at IS NULL;

  UPDATE actions SET published_config = draft_config, published_name = draft_name
   WHERE application_id = $1 AND deleted_at IS NULL;

  UPDATE action_collections SET published_body = draft_body, published_name = draft_name
   WHERE application_id = $1 AND deleted_at IS NULL;

  DELETE FROM pages   WHERE application_id = $1 AND deleted_at IS NOT NULL;
  DELETE FROM actions WHERE application_id = $1 AND deleted_at IS NOT NULL;

  UPDATE applications SET published_theme_id = draft_theme_id,
                          published_js_libs  = draft_js_libs,
                          published_app_layout = draft_app_layout,
                          last_deployed_at   = now()
   WHERE id = $1;
COMMIT;
```

**That is the whole publish flow, atomically.** Compare with [today's five unguarded write sequences](../01-current-system/05-golden-paths.md#9-publish--deploy).

---

## 4. Authorization: how `policyMap` becomes a projection

Today every document embeds `policyMap: {permission → [roleIds]}`, so an authorization check costs zero extra lookups. That property must survive.

**Each service that owns protected resources carries an `authz_grants` table**, fed by events from Identity & Access:

```sql
-- Present in application_db, datasource_db (and any future owner of protected resources)
CREATE TABLE authz_grants (
    resource_type   smallint NOT NULL,   -- 1=application 2=page 3=action 4=datasource 5=theme …
    resource_id     uuid     NOT NULL,
    permission      smallint NOT NULL,   -- enum: Read, Manage, Execute, Delete, Publish, Export …
    role_id         uuid     NOT NULL,
    PRIMARY KEY (resource_type, resource_id, permission, role_id)
);

CREATE INDEX authz_grants_lookup_idx
    ON authz_grants (resource_type, permission, role_id) INCLUDE (resource_id);
```

The read pattern stays **one indexed query, no extra round trip** — exactly today's characteristic:

```sql
SELECT a.*
  FROM applications a
  JOIN authz_grants g
    ON g.resource_type = 1
   AND g.resource_id   = a.id
   AND g.permission    = 1                    -- Read
   AND g.role_id       = ANY($userRoleIds)    -- from the session context
 WHERE a.id = $appId
   AND a.deleted_at IS NULL;
```

Returning zero rows yields a 404, never a 403 — [preserving today's no-existence-leak behaviour](../01-current-system/04-permissions-and-acl.md#2-how-a-permission-check-actually-happens).

The user's role set (`$userRoleIds`) arrives in the request context from the gateway's session validation, cached in Redis — which is exactly what `CacheableRepositoryHelper.getPermissionGroupsOfUser` already does today.

**Who writes `authz_grants`:**
- On resource creation, the owning service writes grants derived from the workspace's roles (the [policy hierarchy graph](../01-current-system/04-permissions-and-acl.md#4-the-policy-graph-how-permissions-cascade) is ported into the owning service as a pure function).
- On `PermissionGrantChanged` / `WorkspaceMemberRemoved` from Identity & Access, the service updates the affected rows as an **idempotent, resumable background projection**.

Note the write-amplification problem is *unchanged* from today (a workspace-level grant change touches every descendant) — but it becomes a tracked, restartable background job instead of an unbounded synchronous cascade.

---

## 5. Schema per service

### 5.1 `identity_db`

```sql
CREATE TABLE instances (               -- collapses Tenant + Organization into one row
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    instance_name       text NOT NULL,
    configuration       jsonb NOT NULL DEFAULT '{}'::jsonb,  -- branding, enabled auth methods
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    instance_id         uuid NOT NULL REFERENCES instances(id),
    email               citext NOT NULL,
    password_hash       text,                 -- NULL for SSO-only users
    is_enabled          boolean NOT NULL DEFAULT true,
    is_anonymous        boolean NOT NULL DEFAULT false,
    email_verified_at   timestamptz,
    last_login_at       timestamptz,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    deleted_at          timestamptz,
    CONSTRAINT users_email_uq UNIQUE (instance_id, email)
);

CREATE TABLE user_profiles (
    user_id             uuid PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    display_name        text,
    photo_url           text,                 -- object storage, not bytes
    git_author_name     text,
    git_author_email    text
);

CREATE TABLE user_preferences (           -- today's UserData grab-bag
    user_id             uuid PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    recently_used       jsonb NOT NULL DEFAULT '{}'::jsonb,
    favorite_app_ids    uuid[] NOT NULL DEFAULT '{}',
    release_notes_seen  text,
    settings            jsonb NOT NULL DEFAULT '{}'::jsonb
);

CREATE TABLE external_logins (            -- OAuth/OIDC identities
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider            text NOT NULL,        -- google | github | oidc | saml
    subject             text NOT NULL,
    CONSTRAINT external_logins_uq UNIQUE (provider, subject)
);

CREATE TABLE workspaces (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    instance_id         uuid NOT NULL REFERENCES instances(id),
    name                text NOT NULL,
    slug                text NOT NULL,
    logo_url            text,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    deleted_at          timestamptz
);

CREATE TABLE workspace_members (          -- replaces User.workspaceIds
    workspace_id        uuid NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id             uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    joined_at           timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (workspace_id, user_id)
);

CREATE TABLE roles (                      -- today's PermissionGroup
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    instance_id         uuid NOT NULL REFERENCES instances(id),
    name                text NOT NULL,
    description         text,
    scope_type          smallint NOT NULL,    -- 0=instance 1=workspace 2=application
    scope_id            uuid,                 -- the resource this role was auto-created for
    is_system           boolean NOT NULL DEFAULT false,  -- Administrator / Developer / App Viewer
    created_at          timestamptz NOT NULL DEFAULT now(),
    deleted_at          timestamptz
);

CREATE TABLE role_assignments (           -- replaces PermissionGroup.assignedToUserIds
    role_id             uuid NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    user_id             uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    assigned_at         timestamptz NOT NULL DEFAULT now(),
    assigned_by         uuid,
    PRIMARY KEY (role_id, user_id)
);

CREATE TABLE permission_grants (          -- the AUTHORITATIVE grant table
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id             uuid NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    resource_type       smallint NOT NULL,
    resource_id         uuid NOT NULL,
    permission          smallint NOT NULL,
    granted_at          timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT permission_grants_uq UNIQUE (role_id, resource_type, resource_id, permission)
);

CREATE INDEX role_assignments_user_idx ON role_assignments (user_id);
CREATE INDEX permission_grants_resource_idx ON permission_grants (resource_type, resource_id);
```

### 5.2 `application_db`

```sql
CREATE TABLE applications (
    id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id          uuid NOT NULL,          -- owned by identity_db; NOT a FK across databases
    base_id               uuid NOT NULL,
    ref_name              text NOT NULL DEFAULT 'main',
    name                  text NOT NULL,
    slug                  text,
    icon                  text,
    color                 text,
    is_public             boolean NOT NULL DEFAULT false,

    draft_theme_id        uuid,
    published_theme_id    uuid,
    draft_app_layout      jsonb,
    published_app_layout  jsonb,
    draft_js_libs         jsonb NOT NULL DEFAULT '[]'::jsonb,
    published_js_libs     jsonb NOT NULL DEFAULT '[]'::jsonb,
    draft_detail          jsonb NOT NULL DEFAULT '{}'::jsonb,
    published_detail      jsonb,

    application_version   integer NOT NULL DEFAULT 1,
    evaluation_version    integer NOT NULL DEFAULT 2,
    forked_from_id        uuid,
    last_deployed_at      timestamptz,
    last_edited_at        timestamptz,

    created_at            timestamptz NOT NULL DEFAULT now(),
    updated_at            timestamptz NOT NULL DEFAULT now(),
    deleted_at            timestamptz,
    CONSTRAINT applications_base_ref_uq UNIQUE (base_id, ref_name)
);
CREATE INDEX applications_ws_idx ON applications (workspace_id) WHERE deleted_at IS NULL;

CREATE TABLE application_pages (        -- replaces the embedded pages[] summary array
    application_id      uuid NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    page_id             uuid NOT NULL,
    position            integer NOT NULL,
    is_default          boolean NOT NULL DEFAULT false,
    is_published        boolean NOT NULL DEFAULT false,   -- draft list vs published list
    PRIMARY KEY (application_id, page_id, is_published)
);

-- pages: see §3

CREATE TABLE actions (
    id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id        uuid NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    page_id               uuid REFERENCES pages(id) ON DELETE CASCADE,
    collection_id         uuid REFERENCES action_collections(id) ON DELETE CASCADE,
    base_id               uuid NOT NULL,
    ref_name              text NOT NULL DEFAULT 'main',

    plugin_id             uuid NOT NULL,        -- owned by execution_db
    plugin_type           smallint NOT NULL,    -- DB | API | JS | SAAS | AI | REMOTE
    datasource_id         uuid,                 -- owned by datasource_db

    draft_name            text NOT NULL,
    draft_config          jsonb NOT NULL DEFAULT '{}'::jsonb,  -- ActionConfiguration
    draft_run_behaviour   smallint NOT NULL DEFAULT 0,          -- Manual | OnPageLoad | Automatic
    draft_is_valid        boolean NOT NULL DEFAULT true,

    published_name        text,
    published_config      jsonb,
    published_run_behaviour smallint,

    created_at            timestamptz NOT NULL DEFAULT now(),
    updated_at            timestamptz NOT NULL DEFAULT now(),
    deleted_at            timestamptz,
    CONSTRAINT actions_base_ref_uq UNIQUE (base_id, ref_name)
);
CREATE INDEX actions_page_idx ON actions (page_id) WHERE deleted_at IS NULL;
CREATE INDEX actions_ds_idx   ON actions (datasource_id);

CREATE TABLE action_collections (       -- JS objects
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id      uuid NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    page_id             uuid REFERENCES pages(id) ON DELETE CASCADE,
    base_id             uuid NOT NULL,
    ref_name            text NOT NULL DEFAULT 'main',
    draft_name          text NOT NULL,
    draft_body          text NOT NULL DEFAULT '',    -- the JS source
    draft_variables     jsonb NOT NULL DEFAULT '[]'::jsonb,
    published_name      text,
    published_body      text,
    published_variables jsonb,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    deleted_at          timestamptz,
    CONSTRAINT action_collections_base_ref_uq UNIQUE (base_id, ref_name)
);

CREATE TABLE themes (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id      uuid REFERENCES applications(id) ON DELETE CASCADE,  -- NULL = system theme
    workspace_id        uuid,
    name                text NOT NULL,
    display_name        text,
    is_system_theme     boolean NOT NULL DEFAULT false,
    config              jsonb NOT NULL DEFAULT '{}'::jsonb,
    properties          jsonb NOT NULL DEFAULT '{}'::jsonb,
    created_at          timestamptz NOT NULL DEFAULT now(),
    deleted_at          timestamptz
);

CREATE TABLE custom_js_libs (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    uid_string          text NOT NULL UNIQUE,     -- name + version + url
    name                text NOT NULL,
    version             text,
    url                 text NOT NULL,
    defs                jsonb,                    -- autocomplete definitions
    accessor            text[]
);

CREATE TABLE application_js_libs (
    application_id      uuid NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    js_lib_id           uuid NOT NULL REFERENCES custom_js_libs(id),
    is_published        boolean NOT NULL DEFAULT false,
    PRIMARY KEY (application_id, js_lib_id, is_published)
);

CREATE TABLE application_snapshots (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id      uuid NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    chunk_order         integer NOT NULL DEFAULT 0,
    snapshot            bytea NOT NULL,           -- compressed artifact JSON
    created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE static_urls (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id      uuid NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
    unique_slug         text NOT NULL UNIQUE,
    is_enabled          boolean NOT NULL DEFAULT true
);

-- Local replica from Datasource Service — read-only, event-fed
CREATE TABLE datasource_summaries (
    datasource_id       uuid PRIMARY KEY,
    workspace_id        uuid NOT NULL,
    name                text NOT NULL,
    plugin_id           uuid NOT NULL,
    is_configured       boolean NOT NULL DEFAULT false,
    updated_at          timestamptz NOT NULL DEFAULT now()
);
```

> **`datasource_summaries` replaces today's worst denormalisation.** Currently every `NewAction` embeds a *complete copy* of its `Datasource` object. Replacing that with a single, explicitly event-synced summary table is strictly better than what exists.

### 5.3 `datasource_db`

```sql
CREATE TABLE environments (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id        uuid NOT NULL,
    name                text NOT NULL,            -- 'production' by default in CE
    is_default          boolean NOT NULL DEFAULT true,
    CONSTRAINT environments_uq UNIQUE (workspace_id, name)
);

CREATE TABLE datasources (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id        uuid NOT NULL,
    plugin_id           uuid NOT NULL,
    name                text NOT NULL,
    is_mock             boolean NOT NULL DEFAULT false,
    is_template         boolean NOT NULL DEFAULT false,
    created_at          timestamptz NOT NULL DEFAULT now(),
    updated_at          timestamptz NOT NULL DEFAULT now(),
    deleted_at          timestamptz,
    CONSTRAINT datasources_name_uq UNIQUE (workspace_id, name)
);

CREATE TABLE datasource_storages (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    datasource_id       uuid NOT NULL REFERENCES datasources(id) ON DELETE CASCADE,
    environment_id      uuid NOT NULL REFERENCES environments(id),
    configuration       jsonb NOT NULL DEFAULT '{}'::jsonb,   -- NON-SECRET config only
    secret_ref          text,                                  -- pointer into the secrets manager
    is_configured       boolean NOT NULL DEFAULT false,
    invalids            text[] NOT NULL DEFAULT '{}',
    updated_at          timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT datasource_storages_uq UNIQUE (datasource_id, environment_id)
);

CREATE TABLE datasource_structures (    -- cached schema introspection
    datasource_id       uuid NOT NULL,
    environment_id      uuid NOT NULL,
    structure           jsonb NOT NULL,
    refreshed_at        timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (datasource_id, environment_id)
);

-- Read-only replica of the plugin catalog, owned by execution_db
CREATE TABLE plugins (
    id                  uuid PRIMARY KEY,
    package_name        text NOT NULL UNIQUE,
    name                text NOT NULL,
    type                smallint NOT NULL,
    icon_url            text,
    form_schema         jsonb,
    updated_at          timestamptz NOT NULL DEFAULT now()
);
```

**Secrets leave the database.** `configuration` holds host/port/database/SSL mode; anything credential-bearing is replaced by `secret_ref`. This removes the current situation where the datasource decryption key lives in the same process that runs untrusted connector code.

### 5.4 `execution_db`

```sql
CREATE TABLE plugins (                  -- SOURCE OF TRUTH
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    package_name        text NOT NULL UNIQUE,
    name                text NOT NULL,
    type                smallint NOT NULL,       -- DB | API | JS | SAAS | AI | REMOTE
    worker_pool         text NOT NULL,           -- sql | nosql | http | cloud | ai | js
    version             text NOT NULL,
    icon_url            text,
    documentation_url   text,
    is_enabled          boolean NOT NULL DEFAULT true
);

CREATE TABLE plugin_forms (
    plugin_id           uuid PRIMARY KEY REFERENCES plugins(id) ON DELETE CASCADE,
    datasource_form     jsonb NOT NULL,
    editor_form         jsonb NOT NULL,
    setting_form        jsonb,
    dependency_map      jsonb
);

CREATE TABLE plugin_templates (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    plugin_id           uuid NOT NULL REFERENCES plugins(id) ON DELETE CASCADE,
    key                 text NOT NULL,
    template            jsonb NOT NULL
);

-- Event-replicated cache of datasource config. NOT authoritative.
CREATE TABLE datasource_config_cache (
    datasource_id       uuid NOT NULL,
    environment_id      uuid NOT NULL,
    plugin_id           uuid NOT NULL,
    configuration       jsonb NOT NULL,
    secret_ref          text,
    version             bigint NOT NULL,          -- event sequence, for staleness detection
    updated_at          timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (datasource_id, environment_id)
);

-- NEW CAPABILITY — none of this is observable today
CREATE TABLE execution_audit (
    id                  bigserial PRIMARY KEY,
    executed_at         timestamptz NOT NULL DEFAULT now(),
    workspace_id        uuid NOT NULL,
    application_id      uuid,
    action_id           uuid,
    datasource_id       uuid,
    plugin_id           uuid NOT NULL,
    worker_pool         text NOT NULL,
    view_mode           boolean NOT NULL,
    duration_ms         integer NOT NULL,
    response_bytes      bigint,
    is_success          boolean NOT NULL,
    error_class         text,
    error_code          text
) PARTITION BY RANGE (executed_at);
```

Partition `execution_audit` by day or week with a retention policy — it is the highest-volume table in the system.

### 5.5 `git_db`

```sql
CREATE TABLE git_repositories (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    artifact_id         uuid NOT NULL,           -- base application id in application_db
    artifact_type       smallint NOT NULL DEFAULT 1,
    remote_url          text NOT NULL,
    repo_name           text NOT NULL,
    default_branch      text NOT NULL DEFAULT 'main',
    deploy_key_id       uuid REFERENCES git_deploy_keys(id),
    is_auto_commit_enabled boolean NOT NULL DEFAULT true,
    connected_at        timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT git_repositories_artifact_uq UNIQUE (artifact_id)
);

CREATE TABLE git_branches (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id       uuid NOT NULL REFERENCES git_repositories(id) ON DELETE CASCADE,
    ref_name            text NOT NULL,
    ref_type            smallint NOT NULL DEFAULT 0,   -- branch | tag
    is_protected        boolean NOT NULL DEFAULT false,
    last_commit_sha     text,
    last_committed_at   timestamptz,
    CONSTRAINT git_branches_uq UNIQUE (repository_id, ref_name)
);

CREATE TABLE git_deploy_keys (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    public_key          text NOT NULL,
    private_key_ref     text NOT NULL,           -- secrets manager reference, NOT the key
    key_type            text NOT NULL DEFAULT 'ECDSA',
    created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE git_profiles (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             uuid NOT NULL,
    repository_id       uuid REFERENCES git_repositories(id) ON DELETE CASCADE, -- NULL = default profile
    author_name         text NOT NULL,
    author_email        text NOT NULL,
    CONSTRAINT git_profiles_uq UNIQUE (user_id, repository_id)
);
```

### 5.6 `notifications_db`

```sql
CREATE TABLE notification_templates (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    key                 text NOT NULL UNIQUE,    -- welcome | invite | password_reset | verify_email
    subject_template    text NOT NULL,
    body_template       text NOT NULL,
    locale              text NOT NULL DEFAULT 'en'
);

CREATE TABLE delivery_log (             -- THE fix for today's swallowed email failures
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    channel             smallint NOT NULL DEFAULT 0,   -- email
    template_key        text NOT NULL,
    recipient           text NOT NULL,
    correlation_id      text,
    payload             jsonb NOT NULL,
    status              smallint NOT NULL DEFAULT 0,   -- Pending|Sent|Failed|DeadLettered
    attempts            integer NOT NULL DEFAULT 0,
    next_retry_at       timestamptz,
    last_error          text,
    created_at          timestamptz NOT NULL DEFAULT now(),
    sent_at             timestamptz
);
CREATE INDEX delivery_log_retry_idx ON delivery_log (next_retry_at) WHERE status = 0;

CREATE TABLE product_alerts (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id          text NOT NULL UNIQUE,
    title               text NOT NULL,
    message             text NOT NULL,
    url                 text,
    starts_at           timestamptz,
    ends_at             timestamptz
);

CREATE TABLE alert_acknowledgements (
    alert_id            uuid NOT NULL REFERENCES product_alerts(id) ON DELETE CASCADE,
    user_id             uuid NOT NULL,
    acknowledged_at     timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (alert_id, user_id)
);

CREATE TABLE usage_events (             -- today's UsagePulse
    id                  bigserial PRIMARY KEY,
    occurred_at         timestamptz NOT NULL DEFAULT now(),
    user_hash           text NOT NULL,           -- hashed, never the raw email
    instance_id         uuid NOT NULL,
    is_anonymous        boolean NOT NULL DEFAULT false,
    view_mode           boolean NOT NULL
) PARTITION BY RANGE (occurred_at);
```

---

## 6. Cross-database references

There are no foreign keys across databases. `application.workspace_id` is a `uuid` column with no `REFERENCES` clause. This is deliberate and has consequences you must handle explicitly:

| Concern | Handling |
|---|---|
| Referential integrity | Enforced by the owning service on write, and by event-driven cleanup on delete |
| Orphan detection | A periodic reconciliation job per service compares its foreign ids against the owner's event stream. Discrepancies alert; they don't auto-delete |
| Workspace deletion | A **saga**: Identity & Access publishes `WorkspaceDeleted`; Application, Datasource and Git cascade-delete their own rows and acknowledge. Tracked to completion, not fire-and-forget |
| Display names for foreign ids | Local projections (`datasource_summaries`), not synchronous lookups |

---

## 7. The transactional outbox

Every service that publishes events writes them **in the same transaction as the state change**, then a background dispatcher publishes them to RabbitMQ. This is non-negotiable: without it, a service can commit a change and crash before publishing, permanently desynchronising every consumer's projection.

```sql
CREATE TABLE outbox (
    id                  bigserial PRIMARY KEY,
    occurred_at         timestamptz NOT NULL DEFAULT now(),
    event_type          text NOT NULL,
    aggregate_type      text NOT NULL,
    aggregate_id        uuid NOT NULL,
    payload             jsonb NOT NULL,
    correlation_id      text,
    published_at        timestamptz
);
CREATE INDEX outbox_unpublished_idx ON outbox (id) WHERE published_at IS NULL;
```

And every consumer keeps an **inbox** for idempotency, since brokers deliver at-least-once:

```sql
CREATE TABLE inbox (
    message_id          uuid PRIMARY KEY,
    consumed_at         timestamptz NOT NULL DEFAULT now()
);
```

MassTransit provides both patterns with EF Core integration — use its implementation rather than hand-rolling. See [.NET 10 Standards §6](04-dotnet-10-standards.md).

---

## 8. Redis: what it is and is not for

| Use | Detail | TTL |
|---|---|---|
| Session store | Gateway; the cookie resolves to a session record | Sliding, matches session lifetime |
| User role-set cache | `{userId, workspaceId} → roleIds[]`, invalidated on `RoleAssignmentChanged` | Short (60s) with event invalidation |
| Git lock | `SETNX` keyed on `artifactId`, with a lease. Carried over unchanged | Lease-based |
| SignalR backplane | Cross-instance presence fanout | n/a |
| Rate limiting | Gateway token buckets | Window-based |
| Idempotency keys | Deduplicating retried mutating requests at the gateway | Minutes |

**Redis is never a system of record.** Everything above must be reconstructible from Postgres or safely lost.

---

## 9. Indexing and performance notes

| Query | Index |
|---|---|
| Editor boot: pages of an application | `pages (application_id) WHERE deleted_at IS NULL` |
| Editor boot: actions of a page | `actions (page_id) WHERE deleted_at IS NULL` |
| Authorization filter | `authz_grants (resource_type, permission, role_id) INCLUDE (resource_id)` |
| Git branch resolution | `applications (base_id, ref_name)` unique |
| Home screen | `applications (workspace_id) WHERE deleted_at IS NULL` |
| Published-app view by slug | `static_urls (unique_slug)` unique |
| Datasource by workspace | `datasources (workspace_id) WHERE deleted_at IS NULL` |
| Execution audit rollups | Partition by time; BRIN on `executed_at` |

**On `jsonb` and the DSL:** page layouts can reach several MB. Postgres TOASTs them automatically, which is fine for whole-document read/write — the access pattern here. Do **not** add GIN indexes on `draft_layout` unless a concrete query needs them; the write cost on a table updated on every canvas keystroke is not worth it.

**On soft deletes:** every index that supports a user-facing query should be partial (`WHERE deleted_at IS NULL`). EF Core's global query filter adds the predicate automatically, so the partial index will be used.

---

[← Target Topology](02-target-topology.md) · [Next: .NET 10 Standards →](04-dotnet-10-standards.md)
