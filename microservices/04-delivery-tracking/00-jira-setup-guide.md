# Delivery Tracking — JIRA Setup Guide

[← Index](../README.md)

---

How to turn the [Epics, Features & User Stories](01-epics-features-stories.md) into a live JIRA project. Read this before importing [`02-jira-import.csv`](02-jira-import.csv).

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
| **Labels** | A feature slug (e.g. `publish-atomic`) + a phase label (e.g. `Phase-A`, `Phase-B`, `Parallel-C0-C7`, `Pre-cutover`) | Lets you build a board filtered by roadmap phase without waiting for real Sprints to be scheduled |
| **Priority** | Highest / High / Medium / Low — JIRA's default scheme, no customisation needed | `Highest` is reserved for the **eight go/no-go checkpoints** from [Roadmap §9](../03-execution/02-roadmap-and-sequencing.md#9-risk-checkpoints) — search `priority = Highest` after import to see all eight in one view |

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

4. **Import Epics first is not required** — JIRA's importer resolves `Epic Link → Epic Name` regardless of row order within one import batch, since the whole file is processed as one import.
5. After import, spot-check: filter `issuetype = Epic`, confirm all 11 exist with the right Components; filter `issuetype = Story`, confirm the count is 217 and every story has a non-empty Epic Link.
6. **If using Advanced Roadmaps Features (§1 option 2):** the CSV importer cannot create the Feature layer or re-parent Stories under it in one pass. After the Epic/Story import, bulk-select stories by their feature Label and use **Bulk Change → Edit → Parent** to re-parent them under manually created Feature issues, one feature group at a time.

---

## 5. Story points and estimation

Points use the Fibonacci scale (1, 2, 3, 5, 8, 13, 21) already assigned in the CSV — **217 stories, 1,328 points total**. Treat these as a **seeded starting estimate for sprint-planning conversations, not a committed velocity plan.** They were sized by relative complexity against the architecture (a schema-and-CRUD story is a 3–5, a saga or a golden-file-gated connector port is an 8–13, the widget-tail catch-all is a deliberately large 21), not against any team's actual throughput.

**Re-baseline after the team's first two sprints on Phase A.** Platform & Foundation stories are the best calibration set — they're infrastructure-shaped work every .NET team has done some version of before, so early velocity there is a reasonable predictor for the rest.

---

## 6. Suggested workflow

Nothing exotic — a standard three/four-column workflow is enough for this backlog:

```
To Do → In Progress → In Review → Done
```

Two additions worth making given this specific programme:

- **A "Blocked — awaiting checkpoint" status or flag**, used on any story downstream of a go/no-go checkpoint (see §3). For example, no `EXEC` connector-port story should leave "To Do" until `EXEC · Build the golden-file replay tooling` is Done — enforce this with a workflow condition or just discipline at planning time.
- **A "Needs ADR" flag**, for any story that turns out to require a fourth saga, a new cross-service sync call, or anything else the [Risks & ADRs](../03-execution/04-risks-and-adrs.md) doc treats as a closed decision. Route these to a tech lead before estimation, not after.

---

## 7. Sprint 0 checklist

Before any Story leaves "To Do," confirm:

- [ ] The 11 Epics exist with correct Components
- [ ] CSV imported cleanly — 217 Stories, all Epic Links resolved
- [ ] Board filters built per Epic (one board per team, per §2's allocation) and one cross-team board filtered on `priority = Highest` for checkpoint visibility
- [ ] `PLAT` epic stories are the Sprint 1 focus — nothing in `IAM`, `APP` or `DS` should start before the BuildingBlocks packages they depend on exist (see [Roadmap §2](../03-execution/02-roadmap-and-sequencing.md#2-phase-a--foundation))
- [ ] The Angular (`NG`) team starts in parallel from Sprint 1, not after the backend — per [Angular Frontend §5](../02-target-architecture/08-angular-frontend.md#5-sequencing), `NG · Evaluation Engine Port (C1)` is the single highest-priority early checkpoint in the whole programme and should not wait on backend phases

---

[← Index](../README.md) · [Next: Epics, Features & User Stories →](01-epics-features-stories.md)
