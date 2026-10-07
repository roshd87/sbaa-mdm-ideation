# Project Scope: Master Data Management (MDM) for StartingBlocks

Grounding document. Section 1 is the verbatim initiating prompt. Later sections are filled in as scope is dialed in through the grilling session.

## 1. Initial Prompt (verbatim, 2026-10-06)

> Scope a large new feature to be added to StartingBlocks Admin App and it's associated tooling. This feature is the addition of a Master Data Management application that can either be an addition to SBAA or a standalone application with deep integration with SBAA. Explore what would be necessary to add in the ability to manage master data AND also push that data as seed data to StartingBlocks instances. Note that StartingBlocks deploys partner StartingBlocks Environments (SBE) and also Ed-Fi tenants within that SBE. MDM would be administered at a partner level and have the ability to manage multiple years worth of master data. That means storing that historical data along side current state and being able to smartly route changes at a user's prompt to the right environment. All Ed-Fi data management rules apply here, notably changes that have downstream data implications are prevented by the API. Record this initial prompt in a ProjectScope.md file as a grounding document. Initate to dial in scope.

## 2. Context Discovered in Codebase

Repos in this workspace (both on branch `Rosh-MDMIdeation`):

- `startingblocks_admin_app/` — Nx monorepo. NestJS API (`packages/api`), React + TanStack frontend (`packages/fe`), TypeORM entities on PostgreSQL (`packages/models-server`), shared DTOs/interfaces (`packages/models`). Background jobs via `pg-boss` (`packages/api/src/sb-sync`, `al-sync`). Talks to Ed-Fi Admin API over HTTPS and to ODS databases via SBE Lambda functions.
- `sbaa_api_client/` — Python client for the SBAA API (`SBAAClient`, `SBAASession`). Supports team / SBE / integration-provider context, tenant/ODS listing, vendor and integration-app management, resource tagging (`School_Year`, `EA_Tenant`, `Environment_Type`).

Existing SBAA resource hierarchy: `Team → SB Environment (SBE) → Ed-Fi Tenant → ODS → Edorg`. SBAA already syncs the EdOrg tree out of each ODS into its own DB (`Edorg` entity, closure table, keyed by `edfiTenantId + odsId + educationOrganizationId`). SBAA explicitly does **not** ingest, transform, or query Ed-Fi data today.

## 3. Decisions

_All closed 2026-10-06. Each entry: question → decision → rationale._

**Q1. What counts as master data (v1)?**
→ EdOrgs, Descriptors, Programs, Assessments (assessment metadata, not StudentAssessments), Chart of Accounts, Certifications, Descriptor Mappings. App must have expansion capacity for future Ed-Fi resources.
→ Rationale: these are partner-owned reference/configuration resources, not transactional student data. "Expansion capacity" implies resource handling should be metadata-driven rather than hand-coded per resource.

**Q2. Where does MDM live?**
→ Separate, separately-deployable Nx packages inside the SBAA monorepo (working names `packages/mdm-api`, `packages/mdm-fe`), with their own PostgreSQL database and deploy pipeline. Shares `models`, `utils`, `common-ui` libs with SBAA.
→ Rationale: reuses SBAA conventions and shared libs without growing SBAA's own scope or DB; keeps the option to split into its own repo later.

**Q3. How are resources modeled in the MDM DB?**
→ Hybrid. Generic JSONB core store for all resource types (`mdm_record`: year set, concrete resource key, canonical natural key, `data` JSONB; identity per Q23), validated against the target ODS/API's OpenAPI spec. Typed projections / purpose-built forms only for EdOrgs and Descriptors (highest-traffic, tree- and set-shaped). All other v1 resources (Programs, Assessments, Chart of Accounts, Certifications, Descriptor Mappings) use the generic schema-driven UI.
→ Rationale: generic core satisfies "expansion capacity" (new resource = config); typed projections buy good UX where users spend most time. Projections are read-models derived from the JSONB core, never a second source of truth.

**Q4. How are multiple years and history represented?**
→ School year is a first-class dimension on every record via its year set (one per team and school year, Q24); identity = (year set, resource, natural key) per Q23. A "roll forward" operation clones year N into N+1 as a starting point; years then diverge independently. Within a year, edit history is an append-only `mdm_record_version` table.
→ Rationale: mirrors how StartingBlocks actually deploys (one ODS per school year, tagged `School_Year`), so routing a record to its target ODS is a direct lookup. Rejected year-less-master-with-overrides (computed merges complicate validation) and bitemporal (no stated need for mid-year effective dating).

**Q5. How are push targets resolved ("smart routing")?**
→ MDM partner = SBAA Team. Targets for a year set = all ODSs the team owns whose SBAA tags match `School_Year = <year>`, optionally narrowed by `EA_Tenant` and `Environment_Type` (e.g. dev before prod); incompatible targets are blocked (Q24). MDM resolves and displays the concrete target list for confirmation before any push. No MDM-owned mapping tables.
→ Rationale: SBAA already owns environment topology and tagging; duplicating it would drift. Consequence: consistent tagging of SBEs/tenants/ODSs becomes a hard operational dependency — untagged ODSs are invisible to MDM.

**Q6. How does MDM write into a target ODS?**
→ ODS/API only. SBAA provisions an MDM-owned integration application **per ODS** (vendor `MDM` with the partner's configured namespace prefixes + versioned claimset `mdm-v<n>` scoped to the Q1 resources enabled so far, bound to that ODS instance). SBAA is the durable custodian of each application's key/secret (Q19). Pushes execute as pg-boss jobs issuing POST/PUT/DELETE against the ODS/API; Ed-Fi's own validation and dependency rules (e.g. 409 on delete with dependent data, reference integrity) are surfaced per record and are never bypassed with direct SQL. Claimset version must be reconciled per ODS before MDM pushes a newly added resource type. *(Revised after adversarial review, 2026-10-06: AR-5.)*
→ Rationale: the prompt's constraint ("changes with downstream implications are prevented by the API") is only true if the API is the sole write path. Rejected SQL bulk-seed path (skips Ed-Fi validation) and file hand-off (no live routing or per-record feedback).

**Q7. What does a push do to the target ODS?**
→ Diff-and-approve, bound to an immutable manifest (Q21). For each compatible target (Q24), MDM GETs current state of the selected resource types (through the ODS's MDM application), computes create/update sets against the published year set, includes deletes **only** for approved tombstones (Q20) and only when opted in, shows a per-target preview, and persists the result as a manifest that a publisher applies. Every apply produces a run record with per-record outcome states (Q25).
→ Rationale: SIS vendors and other integrations also write EdOrgs/descriptors in many deployments; MDM must coexist, not fight them. Preview is also the natural place to surface Ed-Fi dependency refusals before they happen.

**Q7a. Push ordering (resolved from Ed-Fi, not asked).**
→ Records are applied in Ed-Fi dependency order using the ODS/API's `/metadata/data/v3/dependencies` endpoint (available in 6.x and 7.x); deletes in reverse order. No hand-maintained ordering table.

**Q8. How does data get into MDM?**
→ Four intake paths: (1) import-from-ODS bootstrap — pull existing resources from a chosen ODS into a year set using the Q7 GET path; (2) file import — Ed-Fi-shaped JSON per resource (same shape the API accepts) plus CSV for flat resources, validated against the OpenAPI schema on upload; (3) UI entry/edit; (4) pull from a partner's upstream master data source (Q8a). Roll-forward (Q4) then carries years forward.
→ Rationale: import-from-ODS covers migration of existing partners on day one; file import covers partners who maintain master data in spreadsheets. Caveat: CSV schemas per resource/version are a maintenance surface — v1 should accept Ed-Fi JSON for everything and CSV only for Descriptors and flat EdOrg attributes.

**Q8a. Upstream master data source (added 2026-10-06).**
→ Where a partner already holds master data in an upstream system (own MDM, SIS, data warehouse), MDM can connect to it as a read-only *source* via HTTP API: configurable base URL + auth, response mapped to the Ed-Fi resource shape by a per-source mapping. Pulls run as pg-boss jobs, on demand; results land as a change request (Q13a), never directly into published state. Connection secrets stored in AWS Secrets Manager, as SBAA does for IdP credentials. Write direction stays ODS/API only (Q6) — upstream connectors never write back.
→ **Decision 2026-10-06:** direct database connections to partner systems are *deferred* out of v1 (see §5). Partner-side network access (VPN/peering/allow-listing) is a per-partner negotiation, not a product feature; revisit when a partner without an API is onboarded.
→ Rationale: keeps MDM from becoming a second place to hand-maintain data that already has a system of record. Kept light: one mapping config per source, no scheduler, no generic ETL.

**Q9. How are differing Ed-Fi versions across years handled?**
→ Each year set pins the Ed-Fi data standard its targets must run (Q24); targets on another standard are blocked, not merely flagged. Records are stored in exactly the shape that version's API accepts, so push is passthrough and OpenAPI validation is exact. Roll-forward across a version boundary runs an explicit, hand-written per-resource field mapping, maintained only for version pairs StartingBlocks actually runs (today 6.x/DS4 → 7.x/DS5).
→ Rationale: avoids MDM re-implementing the data standard as a canonical model. Version mapping is a one-time per-boundary cost, paid only at roll-forward, not on every push.

**Q10. Authentication and authorization.**
→ `mdm-api` is a cookie-session BFF like SBAA (same IdP, sibling OIDC client, host-scoped `mdm.sid`, SSO via the IdP — never shared cookies). Identity and effective MDM privileges come from one SBAA service endpoint (Q18). Roles are four single SBAA team roles (Q22). No MDM user store, no duplicate RBAC. *(Revised after adversarial review, 2026-10-06: AR-1, AR-16.)*
→ Rationale: SBAA already owns users, teams, roles, and audit; a second RBAC drifts. Distinct roles exist only for separate duties — approve (changes published state), publish (mutates live ODSs), configure.

**Q11. SBAA ↔ MDM contract and tooling.**
→ `mdm-api` holds a client-credentials identity on the shared IdP (same pattern `SBAAClient` uses) and calls the SBAA API for teams, SBEs, tenants, ODSs, tags, and to provision/read MDM integration-app credentials. SBAA API gains a small MDM-facing surface: identity resolution (Q18), owned-ODS targets by tags with fingerprints, per-ODS application ensure, service-only credential read/rotate (Q19), namespace setting (Q20). `sbaa_api_client` gains matching methods plus wrappers for MDM year sets, records, change requests, import, manifests, apply, and runs so automation can drive MDM headlessly.
→ Rationale: keeps SBAA as the single authority on topology and credentials, goes through its authorization and audit. Rejected direct DB reads (bypasses both) and event mirroring (eventual consistency on the data that selects push targets).

**Q12. Failure semantics during a push.**
→ Per-record continue. Independent records proceed; records that reference a failed record (per the Q7a dependency graph) are auto-skipped with reason. Run status is `partial` with per-record outcomes including the Ed-Fi error body. Re-running retries only the failed, skipped, and `uncertain` operations of the same manifest (Q25). No rollback is attempted — the ODS/API is non-transactional and already rejected what it should.
→ Rationale: the API is the gatekeeper; MDM's job is to report precisely and let the user fix data. Stop-at-first-failure leaves the ODS equally half-applied while blocking unrelated work; local pre-validation cannot predict dependency refusals that depend on ODS data MDM doesn't hold.

**Q13. Edit workflow.**
→ Full draft → review → publish lifecycle with approvers. Editors propose changes; approvers accept or reject; only published state is pushable. Version history (Q4) records both the proposal and the approval. Push manifest approval (Q7, Q21) remains as a second, environment-level gate.
→ Rationale: partners require separation of duties on master data beyond a single publisher role. Cost acknowledged: adds change-request state, review UI, and notification surface — the largest single UI investment in v1.

**Q13a. Unit of review and approver role.**
→ Change request = named batch of record edits (any resource types) within one year set. Editor opens, edits, submits (batch frozen, hash stored); approver reviews a single diff and approves/rejects the batch atomically with compare-and-swap on every item's base revision (Q21); approved batch becomes published state. Separation of duties is a server rule, not a role property: any contributor to a change request may not approve it; team admins and machine identities are not exempt; machine identities cannot approve at all (Q22). Imports (Q8) also land as change requests, never directly into published state. *(Revised after adversarial review, 2026-10-06: AR-6, AR-7.)*
→ Rationale: mirrors a pull request — related cross-type edits are reviewed together and approvals stay few.

**Q14. Drift from other writers.**
→ Field-ownership model governs **updates only**: MDM-owned paths (natural key plus a per-resource list of designated JSON paths) are replaced in the target's current representation; non-owned paths (e.g. descriptive EdOrg attributes maintained by SIS vendors) are preserved and shown in the preview for visibility only. **Creates** send the full stored payload — a record imported with non-owned fields seeds a new target completely. Record-level authority (which rows MDM may create, update, or tombstone at all) is a separate concept: managed scope (Q20). Natural keys are immutable in v1; a key change is a reviewed tombstone + create. Detection happens at manifest time on the critical path; background drift scanning is an in-scope stretch feature (Q14a). Ownership lists are global defaults in v1 with a hook for per-partner override later. *(Revised after adversarial review, 2026-10-06: AR-3, AR-11.)*
→ Where a partner wants hard enforcement, SBAA claimsets for other applications can drop Create/Update/Delete on MDM-owned resources. Caveat: Ed-Fi claimsets are resource-level, not field-level — this locks the whole resource, so it only fits resources where MDM owns every field (Descriptors, Descriptor Mappings, Chart of Accounts), not mixed-ownership EdOrgs.
→ Rationale: coexistence with vendor writers (Q7) while guaranteeing identity/structure integrity.

**Q14a. Background drift scanning (stretch, added 2026-10-06).**
→ Opt-in per partner. A scheduled pg-boss job reuses the Q7 manifest build, read-only, against every compatible target of a year set, persists the result as a drift report, and flags drifted records in the UI (and optionally notifies approvers). Read-only: scanning never pushes and never adopts; acting on drift still goes through a change request or a push. Schedule and scope (which years/resource types) configurable per partner.
→ Rationale: proactive visibility for partners with many vendor writers. Stretch, not critical path: it is purely the existing manifest build on a timer, so it can land any time after Stage 1 without architectural change. Cost is N× ODS reads per scan — hence opt-in and scheduled off-peak.

**Q15. Phasing.**
→ Stage 0 (foundations: SBAA endpoints and roles, packages, schema, auth, infra) precedes the phases; Stage N delivers Phase N.
→ Stage 1: Descriptors only, one pilot partner, single Ed-Fi version. Full vertical slice: per-ODS application provisioning → `mdm-api` JSONB store + OpenAPI validation → import-from-ODS → change request → approve → tag-based target resolution → manifest preview → apply with per-record outcomes → `sbaa_api_client` wrappers.
→ Stage 2: EdOrgs (tree projection/UI, mixed field ownership, record-level dependency graph). Stage 3: Programs, Assessments, Chart of Accounts, Certifications, Descriptor Mappings via generic schema-driven UI; roll-forward; cross-version mapping; file import; upstream pull (Q8a). Stage 4 (stretch): drift scanning (Q14a).
→ Rationale: Descriptors are flat, fully MDM-owned, version-stable, and dependency-light, so they exercise every architectural layer with the least UI, and surface workflow (Q13) feedback earliest.

**Q16. Explicit exclusions.** → See §5. Embedding MDM UI inside SBAA's frontend was deliberately *not* excluded: `mdm-fe` ships as its own app in v1, but embedding/deep integration into the SBAA shell remains a future option.

**Q17. UI shell (decided from the clickable mockup, 2026-10-06).**
→ `mdm-fe` reproduces SBAA's chrome exactly (AppBar, resizable Nav with `NavButton` tree, StandardLayout content box, Breadcrumbs, `PageTemplate` with attached actions, `SbaaTable`). The Home page is a workflow inbox: needs-your-review change requests, your drafts, approved manifests ready to apply, partial/suspended runs, drift alerts. Master data is browsed from the nav tree (year set → resource family).
→ Rationale: zero learning curve for SBAA users and 1:1 reuse of `common-ui`; the inbox home fixes the SBAA-style shell's weakness at surfacing what needs attention (Q13 workflow). Rejected year-tab workspace (visual divergence from SBAA) and inbox-only shell (poor for browsing data).

**Pilot (Q15, decided 2026-10-06):** Stage 1 runs against EA's own internal StartingBlocks team/tenant on ODS/API 7.3 / Data Standard 5.2 with TPDM installed, MDM application unprofiled, current school year. An external partner joins at Stage 2. Cross-version (Stage 3) proof uses an EA-hosted 6.x/DS4 sandbox ODS seeded from recorded fixtures, not a partner's live legacy ODS.

**Inventory consistency (AR-13, decided 2026-10-06):** bulk imports and manifest reads run inside a writer-quiescence window agreed with the pilot (SIS integrations idle); any `Total-Count` change between pages aborts the read as `incomplete`. Change-query snapshots and retry-until-stable are not v1.

### Adversarial review outcomes (2026-10-06)

From the adversarial review (2026-10-06); finding *n* is cited as `AR-n` here and in `ImplementationPlan.md`. These contracts gate Stage 0.

**Q18. Identity contract.**
→ SBAA exposes one service-only endpoint, `POST /api/mdm/identity`, callable by the MDM service identity (privilege `mdm:resolve-identity`). Input is a verified `{ iss, sub }` from an id-token `mdm-api` has itself validated, or `{ clientId }` for machine callers — never a request-supplied email. Output: SBAA `userId`, `isMachine`, and per-team effective `team.mdm:*` privileges computed by SBAA's existing authorization logic. Unknown identity fails closed. The MDM service identity holds only `mdm:resolve-identity`, `mdm:read-topology`, `mdm:read-credentials`; it never carries user-level authority. The acting user id travels on every job and audit row.
→ Rationale: SBAA has no email lookup, its machine-user path resolves `azp` to a client, and its privilege model is global role + membership + ownership — none of which the earlier Q10 wording could reuse. One narrow endpoint keeps SBAA the only RBAC.

**Q19. Credential custody.**
→ SBAA stores each MDM application's ODS/API key/secret encrypted at rest (same mechanism as Admin API secrets) in `mdm_integration_credential`, one row per ODS. Retrieval is service-only; rotation keeps the previous secret valid for a bounded overlap; human-facing endpoints return status, never secrets. `mdm-api` holds credentials in memory only.
→ Rationale: SBAA's integration-app model does not persist recoverable ODS credentials — it resets to reveal. Per-run retrieval needs an explicit custodian or every restart would force a rotation race.

**Q20. Managed scope and deletes.**
→ Record-level authority is explicit: `mdm_managed_scope` rows per (year set, resource, scope) where scope is a descriptor namespace prefix, an EdOrg subtree, or all. The MDM vendor's namespace prefixes are set to the partner-owned namespaces; Alliance- and other-vendor-namespaced descriptors are imported read-only for visibility. Deletes exist only as approved tombstone items in a change request; set difference between target and MDM never generates a delete.
→ Rationale: Ed-Fi namespace authorization is separate from resource CRUD, and "all fields owned" says nothing about whether MDM owns an unrelated row. A 409 protects referenced rows, not valid-but-unused ones.

**Q21. Approval binds an immutable manifest; writes are conditional.**
→ Change-request approval is compare-and-swap on each item's base revision (or expected absence) and on the submitted batch hash; any staleness rejects the whole batch for rebase. Push approval produces an immutable `mdm_push_manifest`: published revision, target identities + fingerprints (version, standard, URLs, tags), ordered operations with full write payloads, ownership-policy version, observed `_etag` per existing resource. Apply consumes exactly one manifest atomically (single active run), re-validates each target's fingerprint before touching it (suspending that target on change), and writes with `If-Match`; a 412 is recorded as `conflict` and requires a new manifest. Manifests are superseded when the year set's published revision advances.
→ Rationale: a diff with a TTL is a preview, not an authorization. Without base revisions two drafts silently clobber; without ETags a merged PUT erases a SIS change made after preview; without atomic consumption a manifest replays.

**Q22. Roles.**
→ SBAA memberships hold exactly one role, so MDM ships four single roles: `mdm-editor`, `mdm-approver` (editor + approve), `mdm-publisher` (editor + publish), `mdm-admin` (all + configure). Separation of duties is enforced in `mdm-api` per Q13a, independent of which role a user holds. Replaces the earlier "one user may hold both unless partner configuration forbids it".

**Q23. Resource identity.**
→ The registry is keyed by concrete Ed-Fi endpoint (`gradeLevelDescriptors`, not "Descriptors"); families are a UI grouping. Record identity = (year set, resource key, canonical natural key as ordered JSON, key-schema version); the hash is an index, with canonical-key comparison on collision. Imported rows are normalised to write shape (`id`, `_etag`, `link` stripped). Natural keys are immutable in v1.

**Q24. Year set and target compatibility.**
→ A year set (`mdm_year_set`: team, school year, pinned data standard, key-schema version, status) exists before any target and owns the schema choice. A target is compatible when its fingerprint (API release, data standard, installed extensions, profile constraints from `/metadata`) matches the year set's standard, exposes every registry resource, and the MDM application is unprofiled; incompatible targets are blocked with a reason, not merely flagged. Roll-forward is a per-resource rule in the registry (which embedded year fields change, which year-less resources clone unchanged, how fiscal-year keys are treated).
→ Rationale: data-standard agreement does not imply TPDM is installed or that profiles permit the write representation; descriptors and programs are year-less in Ed-Fi identity, so "the year" is MDM's snapshot/routing dimension, not a field to rewrite everywhere.

**Q25. Run execution states.**
→ Per-operation states: `pending → in_flight → ok | failed | skipped | conflict | uncertain`. The manifest is persisted before enqueue; one active apply per manifest; a startup sweep marks stale `in_flight` rows `uncertain`; `uncertain` is reconciled by reading the record by natural key before any retry. Re-run executes the remaining operations of the **same** manifest — it never re-diffs. Run status: `success | partial | suspended`.
→ Rationale: a pg-boss job and a remote write are not one transaction; Ed-Fi POST is an upsert, so blind retry can replace a concurrent writer's data.

**Deferred to stage acceptance criteria (not Stage 0 gates):** record-level reference graph and reverse-direction delete blocking (Stage 2); schema-form library commitment and nested upstream mappings (Stage 3); upstream connector SSRF/secret-ownership controls (Stage 3, mandatory before the connector ships).

## 4. In Scope

**Resources (v1 total):** EdOrgs, Descriptors, Programs, Assessments (metadata only), Chart of Accounts, Certifications, Descriptor Mappings. Resource set is config-driven so new Ed-Fi resources can be added without a schema change.

**Architecture**
- `packages/mdm-api` (NestJS, cookie-session BFF) and `packages/mdm-fe` (React) in the SBAA monorepo; own PostgreSQL DB; own deploy pipeline; share `models`, `utils`, `common-ui`.
- `mdm-fe` reproduces SBAA's chrome; Home is a workflow inbox; data browsed by year set → resource family (Q17).
- Year sets pin a data standard; generic JSONB record store keyed by (year set, concrete Ed-Fi resource, canonical natural key, key-schema version); append-only versions; managed scopes bound record-level authority; typed projections for EdOrgs and Descriptors.
- Records stored in write shape for the year set's pinned standard; validated against each compatible target's OpenAPI fingerprint.
- Same OIDC IdP as SBAA; identity and `team.mdm:*` privileges resolved through SBAA's service endpoint (Q18); four SBAA roles `mdm-editor`, `mdm-approver`, `mdm-publisher`, `mdm-admin` (Q22).
- `mdm-api` → SBAA API via a least-privilege service identity. SBAA API gains: identity resolution, owned-ODS targets with fingerprints, per-ODS application ensure, service-only credential read/rotate, namespace setting. `sbaa_api_client` gains matching methods plus MDM year-set/record/change-request/manifest/run wrappers.

**Workflow**
- Intake: import-from-ODS, Ed-Fi JSON file import (CSV for Descriptors and flat EdOrg attributes), UI entry, and on-demand pull from a partner's upstream master data source via HTTP API with a per-source mapping to Ed-Fi shape (Q8a). All intake lands as a change request.
- Change request = batch of edits within one year set; submit freezes the batch; approver reviews single diff; approve/reject atomically with CAS on base revisions → published state. No contributor approves their own change request; machine identities never approve (Q13a, Q21).
- Roll-forward clones year set N → N+1 by per-resource registry rules (Q24), running explicit per-resource field mapping when crossing an Ed-Fi version boundary (maintained only for version pairs StartingBlocks runs).

**Push**
- Targets resolved from SBAA tags (`School_Year`, optional `EA_Tenant`, `Environment_Type`) over ODSs the team owns; each target carries a fingerprint; incompatible targets blocked with a reason; list shown before push.
- ODS/API is the only write path, via an MDM-owned integration app **per ODS**; SBAA is credential custodian.
- Manifest-and-apply: GET current → create/update (+ approved tombstones when opted in) → immutable manifest → publisher applies with `If-Match`; `conflict`/`uncertain` states; re-run never re-diffs. Dependency order from `/metadata/data/v3/dependencies`.
- Bulk imports and manifest reads run in a writer-quiescence window; `Total-Count` drift between pages aborts as `incomplete`.
- Field ownership governs updates; creates send full payloads; managed scopes bound which records MDM may touch at all. Global ownership defaults with per-partner override hook.
- Per-operation states `ok | failed | skipped | conflict | uncertain`; dependents of failed records skipped; run `success`, `partial`, or `suspended`; re-run completes the same manifest. Outcomes persisted with the acting user.
- Stretch: opt-in scheduled background drift scan per partner, reusing the manifest build; read-only, produces drift reports and UI flags (Q14a).

**Stages** (Q15)
0. Foundations: SBAA endpoints and roles, `mdm-api`/`mdm-fe` packages, schema, auth, local dev and deploy.
1. Descriptors, pilot (EA internal tenant, ODS/API 7.3 / DS 5.2 / TPDM), single Ed-Fi version, full vertical slice.
2. EdOrgs: tree UI, mixed field ownership, record-level dependency graph; first external partner.
3. Remaining resources via generic UI; roll-forward; cross-version mapping (EA-hosted 6.x/DS4 sandbox); file import; upstream pull.
4. Stretch (any time after Stage 1): background drift scanning (Q14a).

## 5. Out of Scope

- Student, staff, or any transactional data (StudentAssessments, enrollments, etc.).
- Creating, deleting, or configuring SBEs, tenants, or ODSs — topology remains SBAA's job; MDM only targets existing, tagged ODSs.
- Direct ODS SQL writes via Lambda or any path other than the ODS/API.
- Scheduled or automatic pushes — every push is user-initiated and approved (headless pushes only via `sbaa_api_client` by an authorized caller).
- Field-level Ed-Fi permission enforcement — Ed-Fi claimsets are resource-level; MDM cannot lock individual fields on mixed-ownership resources.
- Direct database connections to partner upstream systems — deferred from v1; HTTP-API sources only (Q8a decision).
- Write-back to upstream sources, scheduled upstream pulls, generic ETL (Q8a).
- An MDM-owned user store or RBAC (Q10).
- Automatic rollback of partial pushes (Q12); deletes inferred from set difference (Q20).
- Natural-key edits — a key change is a reviewed tombstone + create (Q14, Q23).
- Change-query snapshots and retry-until-stable inventory reads (inventory decision).
- Embedding `mdm-fe` in the SBAA shell, per-partner ownership overrides, bitemporal records, standalone mode — future (§7).

## 6. Known Risks / Dependencies

- Tagging discipline on SBEs/tenants/ODSs is a hard dependency for routing; untagged ODSs are invisible to MDM.
- Change-request/review UI is the largest single UI investment and the riskiest UX decision; Stage 1 exists to validate it early.
- Cross-version field mappings are hand-maintained per version pair.
- ODS/API is non-transactional; partial pushes are expected and must be legible to users.
- Identity resolution, roles, and credential custody live in SBAA; SBAA releases must precede MDM releases (release order in `ImplementationPlan.md`).
- Pilot capability matrix is closed (7.3 / DS 5.2 / TPDM / unprofiled; EA-hosted 6.x sandbox for DS4). Risk remains that the first external partner (Stage 2) runs a different release or profiles — the fingerprint check must block, not warn.
- Writer-quiescence is an operational agreement, not a technical guarantee; count-drift abort is the only safety net.
- The upstream connector adds network and secret authority; its SSRF and secret-ownership controls are mandatory Stage 3 acceptance criteria.

## 7. Future Scope (not v1)

**7.1 Standalone MDM for data outside StartingBlocks.** A deployment mode in which MDM manages master data for targets that are *not* StartingBlocks-hosted: a district's self-hosted Ed-Fi ODS/API, another vendor's hosted Ed-Fi instance, or a state/collaborative ODS. Added 2026-10-06 at the user's request; recorded now so v1 keeps the seams open.

What changes relative to v1:

- **Target registry becomes MDM-owned.** v1 resolves targets from SBAA tags (Q5) and never stores topology. Standalone mode needs an `mdm_target` table: base URL, ODS/API version, data standard, OAuth credentials (by secret reference), school year, environment type, optional grouping. Q5's tag resolver becomes one *target provider* among two: `SbaaTagTargetProvider` and `RegisteredTargetProvider`.
- **Credentials become MDM-owned.** v1 asks SBAA to ensure an integration app per ODS and SBAA holds its secret (Q6, Q19). Standalone mode stores an ODS/API key/secret per target in Secrets Manager under an MDM-owned reference, entered by the partner's admin for an application they created in their own Admin API/Admin App with a claimset equivalent to the current `mdm-v<n>`.
- **Identity becomes MDM-owned or federated.** v1 delegates users, teams, roles to SBAA (Q10, Q11). Standalone mode needs either (a) its own tenancy model (`mdm_org`, memberships, the five `team.mdm:*` privileges mapped onto local roles) with the same OIDC login, or (b) continued delegation to SBAA for partners who are also SBAA teams but have non-StartingBlocks ODSs. Both can coexist; (b) is the smaller step and the likelier first increment.
- **Topology drift replaces tag drift.** The v1 risk "untagged ODSs are invisible" becomes "registered target is stale/unreachable". Add a target health check (token + `/metadata` ping) before manifest build.
- **Everything else is unchanged.** Record store, change requests, manifest/apply/run semantics, dependency ordering, field ownership, roll-forward, version mapping, upstream sources, drift scanning all operate on "a list of targets with credentials" and are target-provider agnostic by construction.

Seams v1 must keep to make 7.1 cheap:

1. `TargetProvider` interface in `mdm-api` (`resolve(teamId, filters) → Target[]`, `credentials(target) → { key, secret }`, `fingerprint(target)`); v1 ships only the SBAA implementation.
2. `Target` carries `dataUrl`, `oauthUrl`, `metadataUrl`, `dataStandard`, `odsApiVersion`, `fingerprint` — never an SBAA-specific id as its only identity; credentials come only through `TargetProvider.credentials`. SBAA ids live in `provider: { kind: 'sbaa', sbEnvironmentId, edfiTenantId, odsId, odsInstanceId }`.
3. Authorization guard reads a privilege map from `req.user`; the SBAA resolver is the only thing that knows where that map came from.
4. Manifest and apply jobs take `Target[]`, not ODS ids.

Explicitly *not* in 7.1: non-Ed-Fi targets (SIS write-back, data warehouses). Ed-Fi's API is the validation layer the whole design depends on (Q6); other targets would need a different safety model.

**7.2 Other deferred items** (from decisions above): direct-DB upstream sources (Q8a); embedding `mdm-fe` inside the SBAA shell (Q16); per-partner field-ownership overrides (Q14); bitemporal/effective-dated records (Q4).
