# Wireframe guide: `mdm-admin`

How to navigate `mockup/mdm-prototype.html` as an `mdm-admin`, screen by screen, with every action and its path. The mockup holds all state in memory. Reload the page to reset it.

## 1. Set up

1. Serve the repo root: `python3 -m http.server 8000`.
2. Open `http://localhost:8000/mockup/mdm-prototype.html`.
3. In the app bar, set **Acting as** to `a.chen · mdm-admin`. The page opens as `r.dhanawade` (approver) by default.
4. In the app bar, set **Scenario**:
   - **Pilot · 1 tenant**: EA's internal tenant, Lakeside ISD, a handful of targets.
   - **Minnesota · 512 LEAs**: one state team, one Ed-Fi tenant per LEA, MDM as the only master-data writer.

Switching the scenario returns you to Home. Changes you make in one scenario stay until you reload.

## 2. What `mdm-admin` can and cannot do

`mdm-admin` holds every privilege: `team.mdm:read`, `edit`, `approve`, `publish`, `configure`. Every button in the wireframe is enabled for you, except where separation of duties applies:

| Rule | Where you see it | What to do |
|---|---|---|
| You cannot approve a change request you contributed to. | Change request page: **Approve** is disabled with "you contributed to this batch". | Switch **Acting as** to `r.dhanawade` to approve it. |
| You cannot give the second approval to an in-year manifest you built. | Manifest page: **Second approval** is disabled with "you built this manifest". | Switch to `r.dhanawade` (approver) or `j.okafor` (publisher). |
| Team admins are not exempt. | Both alerts say so. | — |

Editing any record adds you as a contributor to your draft change request.

## 3. Layout

| Area | Contents |
|---|---|
| App bar | Logo (Home), team name, **Scenario**, **Acting as**, help, avatar (Account). |
| Left nav | Home, Account, Change requests, Manifests, Runs, Drift, Upstream sources, Settings. |
| Left nav · Year sets | One node per year set. The caret opens its resource families. |
| Breadcrumbs | Above every page title. Every crumb is a link. |
| Page actions | Buttons at the top right of the page card. |

## 4. Paths

| Path | Screen |
|---|---|
| `#/` | Home (inbox) |
| `#/account` | Account and roles |
| `#/cr` | Change requests list |
| `#/cr/<id>` | One change request |
| `#/manifests` | Manifests list |
| `#/manifests/<id>` | One manifest |
| `#/push` | Build manifest wizard |
| `#/runs` | Runs list |
| `#/runs/<id>` | One run |
| `#/drift` | Drift |
| `#/sources` | Upstream sources |
| `#/settings` | Settings |
| `#/rollforward` | Roll forward the latest year set |
| `#/year/<year>` | Year set |
| `#/year/<year>/<family>` | Records of one resource family |

Family keys: `descriptors`, `edorgs`, `programs`, `assessments`, `chartOfAccounts`, `certifications`, `descriptorMappings`, `courses`. Courses appear only in year sets with a `courses` scope.

Seeded ids worth knowing:

| | Pilot | Minnesota |
|---|---|---|
| Year sets | 2024, 2025, 2026 | 2026 (in-year), 2027 (pre-collection setup) |
| Change requests | `#/cr/12` submitted · `#/cr/11` stale · `#/cr/10` approved · `#/cr/9` rejected | `#/cr/41` catalog refresh (2027) · `#/cr/40` in-year code (2026) · `#/cr/39` approved |
| Manifests | `#/manifests/31` approved · 30 done · 29 superseded | `#/manifests/213` setup, approved · `#/manifests/212` in-year, done |
| Runs | `#/runs/7` partial · `#/runs/6` success | `#/runs/88` suspended by the circuit breaker |

## 5. Screens and actions

### Home · `#/`

| Section | Action | Result |
|---|---|---|
| Needs review | **Review** on a row | Opens `#/cr/<id>`. Rows you contributed to say "cannot approve". |
| Your drafts | Row link | Opens your draft change request. |
| Manifests awaiting second approval (Minnesota, when present) | **Review** | Opens `#/manifests/<id>`. |
| Approved manifests ready to apply | **Apply** | Opens `#/manifests/<id>`. |
| Runs needing attention | Row link | Opens `#/runs/<id>`. |
| Drift alerts | **Fix** | Opens `#/push`. |
| Year sets table | Click a row | Opens `#/year/<year>`. |

Lists show five rows, then **View all N →**.

### Year set · `#/year/<year>`

| Action | Result |
|---|---|
| **Build manifest** | Opens `#/push` with this year set selected. |
| **Roll forward** | Opens `#/rollforward`. It always rolls the latest year set forward. |
| **Scopes** | Opens `#/settings`. |
| Click a family row | Opens `#/year/<year>/<family>`. |
| Click a change request row | Opens `#/cr/<id>`. |

The page shows status, pinned standard, published revision, descriptor namespaces, managed scopes, and (Minnesota) the collection-opens date with "pre-collection setup" or "in-year adjustments only". The targets table is paged at 25.

### Records · `#/year/<year>/<family>`

| Action | Result |
|---|---|
| **Import from ODS** | Creates a draft change request with one imported row and opens it. |
| **Import file** | Toast only. |
| **Pull from upstream** | Opens `#/sources`. |
| **Create** (Descriptors) | Prompts for a code value. Adds a `create` item to your draft. |
| Search box | Filters rows on every family. |
| Row hover · pencil | Prompts for the editable field. Adds an `update` item with the row's base revision. |
| Row hover · trash | Adds a `tombstone` item. Only approved tombstones can become deletes. |

Rows outside managed scope are grey with a **read-only** tag and no edit buttons. A blue banner links to your draft once it has items. Descriptor mappings have no editable field; the pencil explains that a change needs tombstone + create.

### Change requests · `#/cr` and `#/cr/<id>`

| Status | Actions |
|---|---|
| `draft` | **Submit for review** (freezes the batch, stores a hash) · **Edit** (opens Descriptors) · **Discard** (toast) |
| `submitted` | **Approve** (compare-and-swap on every base revision) · **Reject** · **Reopen** |
| `approved` | **Build manifest** |
| `rejected` | **Reopen as draft** |

Approving a batch whose records moved rejects it as `stale` and leaves it submitted. Approving publishes the batch and supersedes open manifests for that year set.

### Build manifest · `#/push`

| Step | Controls | Next |
|---|---|---|
| 1 Targets | Year set · `Environment_Type` · `EA_Tenant` (Pilot: select; Minnesota: comma-separated `lea-<id>` list, empty = all LEAs) · **Include approved tombstones as deletes** | **Read targets & preview** |
| 2 Manifest preview | Read-only diff of one sample target. Minnesota adds "same operations on N other targets" and "N targets differ". | **Approve manifest** (setup) or **Approve (first of two)** (in-year) · **← Targets** |
| 3–4 | — | Opens the new manifest at `#/manifests/<id>`. |

Step 1 notes:

- A blue banner marks **Pre-collection setup**. A yellow banner marks **In-year adjustment**: deletes are disabled, a second approval is required, and the run waits for the quiet window.
- Minnesota shows tag coverage, MDM application readiness, and the SEA check, with **Ensure applications** (toast: batch `ensure` under the concurrency cap).
- Blocked targets list first, with the reason. Pilot blocks a DS 5.1 target and a profiled target; Minnesota blocks none.

### Manifest · `#/manifests/<id>`

| Action | Enabled when | Result |
|---|---|---|
| **Second approval** | In-year manifest, awaiting approval, built by someone else | Status → `approved`; records you as second approver. |
| **Apply** (setup) | Status `approved`, published revision current, no active run | Runs immediately; opens `#/runs/<id>`. |
| **Apply in next window** (in-year) | Same, after the second approval | Creates a run waiting for the quiet window. |
| **Discard** | Status `approved` or `awaiting approval` | Toast only. |

The page shows the workflow tag, approvals, targets (paged), blocked targets, operations, and (Minnesota) how the operations are stored.

### Runs · `#/runs` and `#/runs/<id>`

| Action | Enabled when | Result |
|---|---|---|
| **Simulate window opening** | In-year run waiting for its window | Executes the run. |
| **Re-run N remaining (on K targets)** | Failed, skipped, or uncertain operations remain (plus pending after a breaker suspension) | Same manifest, never re-diffs. Uncertain is reconciled first; conflicts stay and need a new manifest. |

A run shows outcome counts. Minnesota runs add a breaker alert (run #88), **Triage by cause**, a **By target** table worst first (paged), and per-operation outcomes for one sample target.

### Drift · `#/drift`

| Action | Result |
|---|---|
| **Scan now** | Toast: read-only manifest build queued. Minnesota scans on demand under the concurrency cap. |
| Schedule checkbox | Toast. Pilot: nightly scan on. Minnesota: off by default. |
| **new manifest** on an owned-path row | Opens `#/push`. |
| **adopt** on an owned-path row | Toast: becomes a change request item. |

### Upstream sources · `#/sources`

| Action | Result |
|---|---|
| **Create** | Toast: HTTP API only, allow-listed origin, team-bound secret. |
| **Pull now** | Adds the source's sample item to your draft (contributor `svc-pipeline`) and opens it. |
| Pencil | Toast: edit mapping. |

### Settings · `#/settings`

| Section | Action | Result |
|---|---|---|
| Apply and quiet windows | **Edit** | Toast: `PUT /teams/:teamId/mdm-settings`. Shows `applyConcurrency`, `applyBreakerPct`, quiet window, `collectionOpensOn` per year set, and (Minnesota) "Master-data writers: MDM only". |
| Partner namespaces | **Edit** | Toast: namespace update on the MDM vendor. |
| Managed scopes by year set | **Add scope** · trash per row | Toast. |
| Field ownership policy | — | Pilot: `default`. Minnesota: `sole-writer`, every path owned. |

### Roll forward · `#/rollforward`

1. **Descriptor namespaces for <next year>**: a text box pre-filled with the latest year set's namespace prefixes, one per line. Each prefix is checked as you type:
   - Unreserved characters only, 64 characters at most, and no descriptor domain such as `/GradeLevelDescriptor`.
   - Minnesota (state team): the host ends in `.gov` or `.us` and names the state, e.g. `uri://education.mn.gov`, `uri://education.mn.state.us`.
2. **Year set and records**: source, new year, pinned standard, and the per-family roll-forward rules.
3. **Create <next> from <latest>**: enabled when every prefix passes. Creates the year set with those prefixes as `namespace` scopes, adds a draft roll-forward change request, and opens `#/year/<next>`.

### Account · `#/account`

Shows your name, team, role, effective privileges, and the role table. No actions.

## 6. Walkthroughs

### Pilot

**A. Approve a change request.**
1. Open `#/cr/12`. **Approve**. Status → `approved`; published revision advances; manifest 31 is superseded.
2. Open `#/cr/11`. **Approve**. It is rejected as stale and stays submitted.

**B. Edit, submit, and hand off approval.**
1. Open `#/year/2026/descriptors`. Hover a `uri://lakeside.org` row; pencil; enter a new description.
2. Follow the banner to your draft. **Submit for review**.
3. **Approve** is disabled: you contributed. Switch **Acting as** to `r.dhanawade` and approve.

**C. Build, apply, and re-run.** Do this before walkthrough A, or the approval supersedes manifest 31.
1. Open `#/manifests/31`. **Apply**. The run shows a `conflict` row (412) and the "no rollback" alert.
2. **Re-run**: uncertain operations are reconciled; the conflict remains.
3. To build a new one: `#/push` → `Environment_Type = dev` → **Read targets & preview** → **Approve manifest** → **Apply**.

**D. Roll forward.** `#/rollforward` → keep `uri://ea.startingblocks.org` and `uri://lakeside.org` → **Create 2027 from 2026**.

### Minnesota

**E. Pre-collection setup apply.**
1. Open `#/manifests/213`. **Apply**. The run fans out to 512 targets, 25 at a time, without tripping the breaker.
2. Read **Triage by cause** and the **By target** table. The sample target shows a `conflict` row from an edit made outside MDM.

**F. In-year adjustment with two approvals.**
1. `#/push` → Year set `2026`. The yellow banner appears; deletes are disabled.
2. **Read targets & preview** → **Approve (first of two)**. The manifest is `awaiting approval`.
3. **Second approval** is disabled: you built it. Switch to `r.dhanawade` → **Second approval**.
4. Switch back to `a.chen` → **Apply in next window**. The run waits for Sat 01:00–05:00.
5. **Simulate window opening**. The run executes.

**G. Resume a breaker suspension.** Open `#/runs/88`. Read the breaker alert: 6 of 52 finished targets failed. **Re-run 926 remaining on 466 targets**. The run finishes as `success`.

**H. Edit the state course catalog.** `#/year/2027/courses` → pencil on `ALG1` → new title → your draft gains an `update` item. Every row is managed: MDM is the only writer.

**I. Check provisioning.** `#/push` (2027) → read "coverage: 512 of 512", "MDM applications: 512 ready", "SEA 990000 present" → **Ensure applications**.

**J. Scan for drift.** `#/drift` → **Scan now**. Owned-path drift rows offer **new manifest** or **adopt**.

**K. Roll forward with namespace checks.**
1. `#/rollforward`. The box shows `uri://education.mn.gov`.
2. Try `uri://mn-education.example.com`: refused, not a state domain.
3. Try `uri://education.mn.gov/GradeLevelDescriptor`: refused, prefix only.
4. Enter `uri://education.mn.gov` → **Create 2028 from 2027**.

## 7. Stand-ins

- Forms are `prompt()` dialogs until Stage 1 picks a form pattern.
- Buttons that only show a toast have no wireframe behaviour yet: Import file, Discard, Create source, Edit mapping, Add scope, Edit settings, Edit namespaces, adopt, scheduled-scan toggle.
- Fonts and icons load from CDNs; offline they fall back.
