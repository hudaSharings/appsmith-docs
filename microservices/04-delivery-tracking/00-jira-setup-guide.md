# Delivery Tracking — JIRA Setup Guide

[← Index](../README.md)

---

How to turn the [Epics, Features & User Stories](01-epics-features-stories.md) into a live JIRA project. Read this before importing [`02-jira-import.csv`](02-jira-import.csv).

This project runs **Kanban, not Sprints** — continuous flow with WIP limits, not fixed iterations. Every "sprint" reference below is a stand-in for "the first block of flow work"; nothing here assumes iteration boundaries. For how a large team pulls this backlog in parallel without idle waiting, see [Parallel Delivery & Dependency Management](03-parallel-delivery-and-dependencies.md) — read it alongside this guide, not after it.

---

## 1. Issue hierarchy

This backlog is three levels deep: **Epic → Feature → Story**. Standard JIRA (Team-Managed and most Company-Managed projects without Advanced Roadmaps) only has two native levels: **Epic → Story**. Pick based on what your instance has:

| Your JIRA plan | How to represent "Feature" |
|---|---|
| **Free / Standard** (no Advanced Roadmaps) | Feature is **not a separate issue**. Use a `Labels` value per feature (already populated in the CSV, e.g. `shared-buildinglocks-libraries`) so stories can be filtered/grouped by feature in board filters and JQL, even though they sit directly under the Epic |
| **Premium / Enterprise with Advanced Roadmaps** | Create the **Feature** issue type in your hierarchy configuration (Settings → Issue types → Hierarchy), sitting between Epic and Story. Create one Feature per feature group listed in the doc, link it to its Epic, then re-parent each imported Story under the matching Feature (the CSV can't set this directly — see §4) |
| **Data Center with a custom hierarchy plugin** | Same idea as Premium — map Feature to whatever your plugin calls its mid-tier issue type |

**Recommendation for most teams:** start with the label-based approach. It's zero-configuration and the CSV already carries the right label. Upgrade to real Feature issues later only if the backlog outgrows flat filtering.

---

## 2. Epics = services, one-to-one

Create exactly **11 Epics**, matching the service inventory:

| Epic key (suggested) | Name | Component |
|---|---|---|
| PLAT | Platform & Foundation | `Platform` |
| IAM | Identity & Access Service | `Identity` |
| APP | Application Service | `Application` |
| DS | Datasource Service | `Datasource` |
| EXEC | Query Execution Service | `Execution` |
| GIT | Git Versioning Service | `Git` |
| RT | Realtime Collaboration Service | `Realtime` |
| NOTIF | Notifications & Telemetry Service | `Notifications` |
| GW | API Gateway BFF Hardening | `Gateway` |
| NG | Angular Client | `Client` |
| MIG | Data Migration & Cutover | `Migration` |

This mirrors [Service Inventory](../02-target-architecture/01-service-inventory.md) exactly and matches the suggested [team allocation](../03-execution/01-decomposition-strategy.md#7-team-shape) — each Epic is naturally one team's backlog, which is why an Epic-per-service split (rather than per-phase or per-sprint) was chosen over alternatives.

**Do not create a 12th "cross-cutting" epic.** Every story belongs to the service that will own the resulting code, even infrastructure-flavoured ones (e.g. BuildingBlocks stories live under PLAT, not floating unassigned).

---

## 3. Components, Labels and Priority

| Field | Values | Source |
|---|---|---|
| **Components** | One per service, matching the table above | Set at the Epic level; every Story inherits it via the CSV's `Components` column |
| **Labels** | A feature slug (e.g. `publish-atomic`) + a phase label (e.g. `Phase-A`, `Phase-B`, `Parallel-C0-C7`, `Pre-cutover`); contract-stub stories additionally carry `contract-stub` + `expedite` | Lets you build a board filtered by roadmap phase, and pull an `expedite`-filtered swimlane for anything unblocking another squad — see [Parallel Delivery §3](03-parallel-delivery-and-dependencies.md#3-the-contract-stub-story) |
| **Priority** | Highest / High / Medium / Low — JIRA's default scheme, no customisation needed | `Highest` marks two different things, both worth a dedicated filter: the **eight go/no-go checkpoints** from [Roadmap §9](../03-execution/02-roadmap-and-sequencing.md#9-risk-checkpoints), and every **contract-stub** story (`priority = Highest AND labels = contract-stub` isolates the latter) |
| **Tier** | `0` / `1` / `2`, one CSV column per Story | Not a native JIRA field — either map it to a custom Select List field on import, or skip mapping it and use the `contract-stub` / phase labels as the practical filter. See [Parallel Delivery & Dependency Management](03-parallel-delivery-and-dependencies.md) for what each tier means |

---

## 4. Importing the CSV

1. **Project Settings → System → External System Import → CSV.**
2. Upload [`02-jira-import.csv`](02-jira-import.csv).
3. Map columns:

   | CSV column | Maps to |
   |---|---|
   | `Issue Type` | Issue Type |
   | `Summary` | Summary |
   | `Description` | Description |
   | `Epic Name` | Epic Name (only populated on Epic rows) |
   | `Epic Link` | Epic Link (only populated on Story rows — value matches the referenced Epic's `Epic Name`) |
   | `Story Points` | Story Points (or your instance's equivalent custom field) |
   | `Priority` | Priority |
   | `Components` | Components |
   | `Labels` | Labels (comma-separated — JIRA splits automatically) |
   | `Tier` | A custom Select List field (optional — see §3) |

4. **Import Epics first is not required** — JIRA's importer resolves `Epic Link → Epic Name` regardless of row order within one import batch, since the whole file is processed as one import.
5. After import, spot-check: filter `issuetype = Epic`, confirm all 11 exist with the right Components; filter `issuetype = Story`, confirm the count is 263 and every story has a non-empty Epic Link.
6. **If using Advanced Roadmaps Features (§1 option 2):** the CSV importer cannot create the Feature layer or re-parent Stories under it in one pass. After the Epic/Story import, bulk-select stories by their feature Label and use **Bulk Change → Edit → Parent** to re-parent them under manually created Feature issues, one feature group at a time.

---

## 5. Story points and estimation

Points use the Fibonacci scale (1, 2, 3, 5, 8, 13, 21) already assigned in the CSV — **263 stories, 1,507 points total** (this includes 26 contract-stub stories at 2 points each — see [Parallel Delivery & Dependency Management](03-parallel-delivery-and-dependencies.md)). Treat these as a **seeded starting estimate, not a committed throughput plan.** They were sized by relative complexity against the architecture (a schema-and-CRUD story is a 3–5, a saga or a golden-file-gated connector port is an 8–13, the widget-tail catch-all is a deliberately large 21), not against any team's actual cycle time.

**Re-baseline once Platform & Foundation stories start flowing through Done.** They're infrastructure-shaped work every .NET team has done some version of before, so their early cycle time is a reasonable predictor for the rest — track cycle time and throughput per Epic (Kanban's native metrics), not velocity.

---

## 6. Suggested workflow (Kanban board, per Epic)

Each Epic runs its own board — one team, one board, one WIP limit:

```
Backlog → Blocked → Ready → In Progress → In Review → Done
```

`Backlog` and `Ready` are the two columns that carry this programme's dependency logic — see [Parallel Delivery & Dependency Management §1](03-parallel-delivery-and-dependencies.md#1-the-tier-model) for exactly when a story is allowed to cross from one to the other:

- A **Tier 0** story sits in `Ready` from the start — nothing gates it.
- A **Tier 1** story sits in `Blocked` until its contract-stub story (see §3 there) is Done, then moves to `Ready` — the mock, not the real upstream feature, is what unblocks it.
- A **Tier 2** story stays in `Blocked` until the real upstream work it names is Done.

Set a **WIP limit on `In Progress`** per board (roughly 1.5× the squad's headcount) — this is the control that keeps a large team from starting more than it can finish. Add one cross-cutting board too: every story labelled `contract-stub`, sorted by age — this is what a large team watches to see which contracts are the current bottleneck, since a stale one blocks every squad waiting on it, not just its own.

A **"Needs ADR" flag**, for any story that turns out to require a fourth saga, a new cross-service sync call, or anything else the [Risks & ADRs](../03-execution/04-risks-and-adrs.md) doc treats as a closed decision, is worth adding too — route these to a tech lead before work starts, not after.

---

## 7. Kanban board setup checklist

Before any Story leaves `Backlog`, confirm:

- [ ] The 11 Epics exist with correct Components
- [ ] CSV imported cleanly — 263 Stories, all Epic Links resolved
- [ ] Board filters built per Epic (one board per team, per §2's allocation), one cross-team board filtered on `priority = Highest` for checkpoint visibility, and one filtered on `labels = contract-stub` for dependency-bottleneck visibility
- [ ] Every Tier 0 story starts in `Ready`, not `Backlog` — there's no reason to hold it
- [ ] `PLAT`'s `BuildingBlocks.*` stories are pulled first by every squad's own developers if idle, since every other Epic's code depends on that package existing — this is a compile-time library dependency, not a Tier 1/2 runtime one, and it's the one true "everyone waits" case in the whole backlog (see [Parallel Delivery §4](03-parallel-delivery-and-dependencies.md#4-the-one-real-everyone-waits-dependency))
- [ ] The Angular (`NG`) team pulls from `Ready` from day one, not after the backend — per [Angular Frontend §5](../02-target-architecture/08-angular-frontend.md#5-sequencing), `NG · Evaluation Engine Port (C1)` is Tier 0 and should not wait on backend phases

---

[← Index](../README.md) · [Next: Epics, Features & User Stories →](01-epics-features-stories.md)
