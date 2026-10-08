# Engineering Brief: MDM for SBAA

Technical summary of the Master Data Management (MDM) addition to the StartingBlocks Admin App (SBAA). Sources of record: `ProjectScope.md` (decisions Q1–Q26), `ProjectPlan.md` (stages), `ImplementationPlan.md` (76 steps). This brief does not override them.

## 1. What ships

- MDM stores partner master data per **year set**, reviews changes as **change requests**, and pushes approved state to Ed-Fi ODSs as an immutable **manifest**.
- The ODS/API is the only write path. MDM never writes ODS SQL (Q6).
- SBAA stays the authority for users, roles, topology, tags, and ODS credentials (Q10, Q11, Q18, Q19).
- MDM is two new deployables in the SBAA monorepo, with their own database and pipeline (Q2).

## 2. Architecture

```
Browser ──cookie mdm.sid──▶ mdm-fe (React, :4201)
                              │
                              ▼
                         mdm-api (NestJS BFF, :3334) ──▶ MDM Postgres (+ pgboss_mdm)
                              │  M2M client-credentials
                              ├──────────────▶ SBAA API  /api/mdm/*, /api/teams/:teamId/mdm/*
                              │  per-ODS OAuth (key/secret from SBAA)
                              └──────────────▶ Ed-Fi ODS/API (6.x / 7.x)
```

|Component|Path|Notes|
|---|---|---|
|`mdm-api`|`packages/mdm-api`|NestJS. Own TypeORM datasource, `synchronize: false`. Own `PgBossModule` on schema `pgboss_mdm`.|
|`mdm-fe`|`packages/mdm-fe`|React + Vite. Reuses `@edanalytics/common-ui`. Copies SBAA `Layout/*` chrome. Home is a workflow inbox.|
|MDM DB|`mdm-db` (local port 3307)|PostgreSQL 14. Separate from the SBAA DB.|
|Shared libs|`models`, `utils`, `common-ui`|Shared DTOs and UI. No shared DB.|
|Client|`sbaa_api_client`|Python wrappers for every MDM operation. Machine identities cannot approve.|

## 3. Auth and identity (Q10, Q18, Q22)

- `mdm-api` is a cookie-session BFF on the SBAA IdP: a sibling OIDC client, host-scoped `mdm.sid`, `SameSite=Lax`. SSO runs through the IdP. Cookies are never shared.
- On login, `mdm-api` calls SBAA `POST /api/mdm/identity { iss, sub }`. Machine callers resolve with `{ clientId: azp }`. The session caches `{ userId, teams[] }` and refreshes every 60 s or on 403.
- Unknown identity fails closed. There is no email lookup and no MDM user store.
- CSRF: reject state-changing requests whose `Origin` ≠ `FE_URL`.

|Role (single SBAA team role)|Privileges|
|---|---|
|`mdm-editor`|`team.mdm:read`, `team.mdm:edit`, `team.sb-environment:read-tag`|
|`mdm-approver`|editor + `team.mdm:approve`|
|`mdm-publisher`|editor + `team.mdm:publish`|
|`mdm-admin`|all + `team.mdm:configure`|
|MDM service identity|`mdm:resolve-identity`, `mdm:read-topology`, `mdm:read-credentials` only|

Separation of duties is an `mdm-api` rule, not a role property. A contributor to a change request cannot approve it. Team admins are not exempt. Machine identities never approve.

## 4. SBAA changes

New module `packages/api/src/mdm/`. All paths sit under `/api`.

|Endpoint|Caller|Purpose|
|---|---|---|
|`POST /mdm/identity`|service|Resolve `{ iss, sub }` or `{ clientId }` → `{ userId, isMachine, teams[{ teamId, privileges }] }`.|
|`GET /teams/:teamId/mdm/targets?School_Year=&EA_Tenant=&Environment_Type=`|human, service|List team-owned ODSs matching tags, with URLs, versions, tags, and `fingerprint`.|
|`POST /teams/:teamId/mdm/applications/ensure { odsId }`|`configure`|Idempotent. One MDM application per ODS. Vendor `MDM`, claimset `mdm-v<n>`. Returns status only, never the secret.|
|`GET /mdm/applications/:odsId/credentials`|service|Return the key, secret, and previous secret during the rotation overlap.|
|`POST /mdm/applications/:odsId/rotate`|service|Issue a new secret. The previous one stays valid for 24 h. Audited with `X-MDM-Actor`.|
|`PUT /teams/:teamId/mdm/namespaces { prefixes }`|`configure`|Set the MDM vendor's `namespacePrefixes`.|

- New table `mdm_integration_credential`: `odsId` unique, encrypted key and secret, rotation fields, `claimsetVersion`. It uses SBAA's existing at-rest encryption.
- Versioned claimsets live in `packages/models/src/mdm/claimset.v<n>.json`. `ensure` reconciles each ODS to the current version before MDM pushes a new resource type.
- Roles are seeded by data migration. All changes are additive and backward compatible.

## 5. MDM data model (Q3, Q4, Q20, Q21, Q23, Q24, Q25)

|Table|Key fields|
|---|---|
|`mdm_year_set`|`teamId`, `schoolYear`, pinned `dataStandard`, `keySchemaVersion`, `status`; unique (team, year)|
|`mdm_record`|`yearSetId`, `resourceKey`, `canonicalKey` (jsonb), `canonicalKeyHash`, `data` (write-shape jsonb), `revision`|
|`mdm_record_version`|append-only `before`/`after`, `changeRequestId`, `actorUserId`|
|`mdm_managed_scope`|`yearSetId`, `resourceKey`, `scopeKind` `namespace \| edorgSubtree \| edorg \| all`, `scopeValue`|
|`mdm_change_request` (+ `_contributor`)|`status` `draft → submitted → approved \| rejected`, `submittedHash`, `approverUserId`|
|`mdm_change_item`|`op` `create \| update \| tombstone`, `baseRevision` or `expectAbsent`, `after`|
|`mdm_push_manifest`|immutable: `publishedRevision`, `targets` + fingerprints, ordered `operations` with payloads and `observedEtag`, `ownershipPolicyVersion`, `includeDeletes`|
|`mdm_push_run`, `mdm_push_run_record`|run `running \| success \| partial \| suspended`; operation `pending → in_flight → ok \| failed \| skipped \| conflict \| uncertain`|
|`mdm_upstream_source`, `mdm_drift_settings`, `mdm_drift_report`|Stage 3–4|

- Record identity is (year set, resource key, canonical natural key, key-schema version). The hash is an index only; on a hash match, compare `canonicalKey`.
- Imported rows are normalised to write shape: strip `id`, `_etag`, `link`, and `*Reference.link`.
- Natural keys are immutable in v1. A key change is a reviewed tombstone + create.

## 6. Resource registry (Q1, Q3, Q23, Q26)

- Entries are keyed by concrete endpoint (`gradeLevelDescriptors`), not by family.
- At boot, the registry is generated from the pinned standard's OpenAPI document plus a hand-written overlay: `{ resourceKey, naturalKeyFields, keySchemaVersion, ownedPaths, referenceExtractors, rollForward, optIn }`.
- A new resource type is a config change, not a schema change.

|Family|Endpoints|Natural key|Owned paths|UI|Stage|
|---|---|---|---|---|---|
|Descriptors|`*Descriptors`|`namespace`, `codeValue`|all|purpose-built|1|
|EdOrgs|`localEducationAgencies`, `schools`, `educationServiceCenters`, …|per subtype|`nameOfInstitution`, `shortNameOfInstitution`, parent refs, `educationOrganizationCategories`|tree|2|
|Programs|`programs`|from OpenAPI|all|schema form|3|
|Assessments|`assessments`, `objectiveAssessments`|from OpenAPI|all|schema form|3|
|Chart of Accounts|`chartOfAccounts` + dimension endpoints|from OpenAPI|all|schema form|3|
|Certifications|`tpdm/certifications` (needs TPDM)|from OpenAPI|all|schema form|3|
|Descriptor Mappings|`descriptorMappings`|from OpenAPI|all|schema form|3|
|Courses (opt-in)|`courses`|`courseCode`, `educationOrganizationId`|all, inside `edorg` scope|schema form|3|

Courses behaviour by deployment:

|Deployment|Configuration|Result|
|---|---|---|
|State-only catalog|`courses` scope of kind `edorg` = the SEA id|MDM owns the full catalog.|
|State + local SIS courses (e.g. South Carolina)|same scope|SEA courses are managed. LEA and school courses import as `external`, read-only, and are never written.|
|Non-state (SIS-owned courses)|no `courses` scope|The family is off: no import, no nav, no manifest operations.|

`courseOfferings`, `sections`, and `courseTranscripts` are never in the registry or the claimset.

## 7. Workflow

**Intake (Q8, Q8a).** Import-from-ODS, Ed-Fi JSON file (CSV for descriptors and flat EdOrg attributes), UI edits, and HTTP-API upstream pull. Every path writes a draft change request. Rows outside managed scope are stored `external`, for display only.

**Review (Q13a, Q21).**
- Submit freezes the items and stores `submittedHash`.
- Approve runs in one transaction:
  - Assert the hash is unchanged.
  - CAS every item's `baseRevision`, or absence for creates.
  - Reject the whole batch as `stale` on any mismatch.
  - Otherwise apply the items, bump revisions, and write versions.

**Manifest build (Q5, Q7, Q14, Q20, Q24).**
- Resolve targets through `TargetProvider` (v1: `SbaaTagTargetProvider`).
- Block incompatible targets with a reason. A target is compatible only when all of these hold:
  - Its data standard matches the year set's pinned standard.
  - It exposes every registry resource.
  - The MDM application is unprofiled.
- Read target state with `totalCount=true`. Abort as `incomplete` if `Total-Count` changes between pages.
- Compute operations:
  - `create`: full stored payload.
  - `update`: owned paths replaced in the target's current representation.
  - `delete`: only from approved tombstones, and only when `includeDeletes` is set. Set difference never produces a delete.
- Order operations by `/metadata/data/v3/dependencies`; deletes run in reverse.

**Apply (Q12, Q21, Q25).**
- Preconditions:
  - The manifest is `approved`.
  - `publishedRevision` is still current.
  - No other run is active.
- Per target, re-check the fingerprint. On change, suspend that target.
- Per operation:
  - Write with `If-Match`. 412 → `conflict`. Other 4xx/5xx → `failed` with the body. Unknown outcome → `uncertain`.
  - Dependents of a failed record → `skipped`.
- A startup sweep marks stale `in_flight` rows `uncertain`.
- Re-run uses the same manifest and never re-diffs. It retries only `failed`, `skipped`, and `uncertain` operations; each `uncertain` one is reconciled by a natural-key GET first. A `conflict` requires a new manifest.
- There is no rollback; the ODS/API is non-transactional.

**Roll-forward (Q4, Q9, Q24).** Create year set N+1, then clone published records into a draft change request using per-resource rules. Year-less resources clone unchanged. Crossing a standard runs the hand-written `ds4-to-ds5` mapping; unmapped required fields become item errors.

## 8. Stages

|Stage|Scope|Exit criterion|
|---|---|---|
|0 Foundations|SBAA roles + endpoints, packages, schema, auth, infra|`mdm-api` authenticates an SBAA user and resolves the pilot's tagged ODSs; credentials survive a restart without rotation.|
|1 Descriptors|Full vertical slice, EA internal pilot (ODS/API 7.3, DS 5.2, TPDM)|Import → approve → manifest → apply to a dev ODS; conflict and uncertain cases demonstrated; visible in UI and `sbaa_api_client`.|
|2 EdOrgs|Tree UI, mixed ownership, record-level reference graph|Ordered LEA + school creates; a SIS-owned field survives; a failed child delete blocks its parent.|
|3 Remaining + years|Six families incl. Courses, roll-forward, DS4 → DS5, file + upstream intake|All endpoints pushable; a 6.x/DS4 sandbox year set rolls into DS5 and pushes; Courses scope behaviour verified.|
|4 Drift (stretch)|Scheduled read-only manifest build|The pilot receives a drift report; drifted records flagged.|

## 9. Release order

1. SBAA: migration (roles, `mdm_integration_credential`), privileges, `/mdm` endpoints.
2. Run `ensure` per pilot ODS. Verify the claimset fingerprint with one read.
3. MDM: migrations, `pgboss_mdm`, workers. Queue payloads carry `schemaVersion`.
4. Enable per team through an SBAA team flag.
5. Each claimset bump deploys before the MDM release that needs it.

## 10. Operational constraints and risks

- **Tags are the routing.** An untagged ODS is invisible to MDM.
- **Quiescence is operational.** Bulk reads assume idle SIS writers. Count drift is the only technical guard.
- **Partial runs are normal.** Outcomes must stay legible per operation.
- **Version mappings are hand-written** for the version pairs StartingBlocks runs.
- **The upstream connector carries SSRF and secret risk.** It needs team-bound secrets, allow-listed HTTPS origins, private-IP rejection, and no cross-origin redirects.
- **Courses need their owning EdOrg in the target.** A missing SEA fails every Course create.

## 11. Out of scope (v1)

- Student, staff, and transactional data.
- Course offerings, sections, and transcripts.
- ODS SQL writes; scheduled pushes; rollback.
- Topology management.
- Direct-DB upstream sources; write-back to upstream sources.
- An MDM user store.
- Change-query snapshots.
- Embedding `mdm-fe` in the SBAA shell.

## 12. Seams for standalone mode (Scope §7.1)

- `TargetProvider` interface: `resolve`, `credentials`, `fingerprint`. v1 ships only the SBAA implementation.
- `Target` identity is URLs + fingerprint. SBAA ids live only under `provider`.
- The authorization guard reads `req.user.teams` only.
- Manifests and runs take `Target[]`, never ODS ids.
