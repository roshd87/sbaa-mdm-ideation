# Implementation Plan: MDM in the SBAA monorepo

Step-by-step engineering plan for `ProjectScope.md` / `ProjectPlan.md` against the real repo layout. Revision 2 (2026-10-06) folds in the adversarial review (2026-10-06); steps it changed are tagged **[AR-n]** = adversarial review finding *n* (cited the same way in `ProjectScope.md`). Paths are relative to `startingblocks_admin_app/` unless noted. Steps are ordered; each sub-stage ends with the check that proves it done. Conventions: squash-merge into `develop` with a semantic prefix (`feature:`), migrations via `npm run migrations:generate -- <Name>`, privileges declared in `packages/models/src/types/privileges-source.ts`, controllers guarded with `@Authorize({ privilege, subject })`.

---

## Stage 0 — Foundations

### 0.1 SBAA roles and privileges
1. In `packages/models/src/types/privileges-source.ts` add an `mdmPrivileges` block, spread into `privileges`:
   - team-scoped: `team.mdm:read`, `team.mdm:edit`, `team.mdm:approve`, `team.mdm:publish`, `team.mdm:configure`
   - service-scoped (machine identity only): `mdm:resolve-identity`, `mdm:read-topology`, `mdm:read-credentials` **[AR-1]**
2. **[AR-6, AR-15]** Add a data migration `packages/api/src/database/migrations/<ts>-mdm-roles.ts` (not `demo-populate.ts`, which is absent) seeding four single team roles — SBAA memberships hold exactly one role (`user-tenant-membership.entity.ts`):
   - `mdm-editor`: `read`, `edit`, `team.sb-environment:read-tag`
   - `mdm-approver`: editor + `approve`
   - `mdm-publisher`: editor + `publish`
   - `mdm-admin`: all five + `configure`
   Team admin role gains the same five. Separation of duties is **not** a role property — it is enforced in step 40 (machine identities also blocked by step 22).
3. Check: Roles page lists four roles with the stated privileges; a user with `mdm-editor` can call step 5.

### 0.2 SBAA API — MDM-facing endpoints
4. New module `packages/api/src/mdm/` mounted in `app/routes.ts` as `/mdm` (service routes) and `/teams/:teamId/mdm` (team routes). Register in `app/app.module.ts`. Swagger tag `Mdm`. All paths below sit under SBAA's global `/api` prefix (e.g. `POST /api/mdm/identity`).
5. **[AR-5, AR-14]** `GET /teams/:teamId/mdm/targets?School_Year=&EA_Tenant=&Environment_Type=` — ODSs the **team owns** (apply the same ownership filter SBAA's ODS listing uses), then tag-match via `TagsTeamService.getDerivedTags(sbEnvironmentId, edfiTenantId, odsId)`. Return per ODS: `{ odsId, odsInstanceId, edfiTenantId, sbEnvironmentId, label, odsApiVersion, dataStandard, dataUrl, oauthUrl, metadataUrl, tags, tagSources, fingerprint }` where `fingerprint = sha256(version|standard|urls|tags)`. 6.x/7.x route construction stays in SBAA's existing v1/v2 adapters. Guard `team.mdm:read` for humans, `mdm:read-topology` for the service identity.
6. **[AR-2, AR-5]** `POST /teams/:teamId/mdm/applications/ensure { odsId }` — idempotent, **one application per ODS**: find-or-create vendor `MDM` with `namespacePrefixes` = the partner's configured namespaces (step 8), claimset `mdm-v<n>` (step 9), application `mdm-<tenant>-<ods>` bound to `odsInstanceIds: [odsInstanceId]` via the v2 service (v1 equivalent via ed-org scope). On create or rotate, persist the key/secret in new table `mdm_integration_credential` (`odsId` unique, `encryptedKey`, `encryptedSecret`, `rotatedAt`, `previousEncryptedSecret`, `previousValidUntil`) using the same at-rest encryption SBAA uses for Admin API secrets. Returns status + claimset fingerprint only — **never the secret**. Guard `team.mdm:configure`.
7. **[AR-2]** `GET /mdm/applications/:odsId/credentials` — service-only (`mdm:read-credentials`), returns `{ key, secret, previousSecret?, previousValidUntil? }`. `POST /mdm/applications/:odsId/rotate` — service-only; new secret, previous valid for 24 h. Audit both with the acting MDM user id passed in header `X-MDM-Actor`.
8. **[AR-3]** `PUT /teams/:teamId/mdm/namespaces { prefixes: string[] }` — partner-owned descriptor namespaces MDM may write; updates the MDM vendor's `namespacePrefixes` (existing vendor-update path). Guard `team.mdm:configure`.
9. Claimset definitions `packages/models/src/mdm/claimset.v1.json` … versioned; `ensure` reconciles the per-ODS claimset to the current version and records `claimsetVersion` on `mdm_integration_credential`. Descriptors endpoints only at v1.
10. **[AR-1]** `POST /mdm/identity` — service-only (`mdm:resolve-identity`). Body `{ iss, sub }` (from an id-token mdm-api has already validated) **or** `{ clientId }` (machine caller). Resolves the active SBAA user by stored OIDC subject (humans) or machine `clientId` (machines) and returns `{ userId, isMachine, teams: [{ teamId, privileges: string[] }] }` computed by the existing `AuthService` privilege logic restricted to `team.mdm:*`. No email lookup. Unknown identity → 404, fail closed.
11. Check: ensure twice → second call returns existing app without rotation; credentials endpoint works after an API restart; identity endpoint returns the pilot approver's privileges; `targets` excludes an ODS the team does not own.

### 0.3 `mdm-api` package
12. `nx g @nx/nest:application mdm-api --directory=packages/mdm-api` (targets as in `packages/api/project.json`: `serve`, `build`, `build-migration-config`, `migrations:*`). Port 3334. npm scripts `start:mdm-api:dev`, `build:mdm-api`, `mdm:migrations:*`.
13. `packages/mdm-api/config/` copied from `packages/api/config/`. Keys: `MDM_DB_CONNECTION_STRING`, `SBAA_API_URL`, `SBAA_M2M_CLIENT_ID/SECRET/AUDIENCE`, `OIDC_ISSUER/CLIENT_ID/CLIENT_SECRET`, `MY_URL` (3334), `FE_URL` (4201), `SESSION_SECRET`.
14. **[AR-15]** `packages/mdm-api/src/database/` with its own `typeorm.config.ts`, `migrations.datasource.ts`, `migrations/`. `synchronize: false`. Own `PgBossModule` using `MDM_DB_CONNECTION_STRING` and schema `pgboss_mdm` — written fresh, not copied (SBAA's copy hard-codes `pgboss12` and registers SBAA queue views).
15. **[AR-8, AR-17, AR-2, AR-4, AR-12]** Entities in `packages/mdm-api/src/entities/`:
    - `mdm_year_set` — `teamId`, `schoolYear`, `dataStandard` (pinned), `keySchemaVersion`, `status` (`draft|active|archived`); unique (team, year). Exists before any target.
    - `mdm_record` — `yearSetId`, `resourceKey` (concrete endpoint, e.g. `gradeLevelDescriptors`), `canonicalKey` (jsonb, canonical field order), `canonicalKeyHash`, `data` (jsonb, write-shape: no `id`, `_etag`, `link`), `revision` (int), timestamps; unique (yearSetId, resourceKey, canonicalKeyHash); on hash match compare `canonicalKey` before treating as same record.
    - `mdm_record_version` — append-only: `recordId`, `revision`, `changeRequestId`, `op`, `before`, `after`, `actorUserId`, `createdAt`.
    - `mdm_managed_scope` — `yearSetId`, `resourceKey`, `scopeKind` (`namespace|edorgSubtree|all`), `scopeValue`. Only records inside a scope may be created/updated/tombstoned.
    - `mdm_change_request` — `yearSetId`, `title`, `status` (`draft|submitted|approved|rejected`), `authorUserId`, `submittedHash`, `approverUserId`, timestamps. `mdm_change_request_contributor` — (`changeRequestId`, `userId`).
    - `mdm_change_item` — `changeRequestId`, `resourceKey`, `canonicalKeyHash`, `op` (`create|update|tombstone`), `baseRevision` (null + `expectAbsent` for create), `after`.
    - `mdm_push_manifest` — immutable: `yearSetId`, `publishedRevision` (max record revision at approval), `targets` (jsonb: identity + fingerprint + tags), `operations` (jsonb: ordered `{ resourceKey, canonicalKey, op, payload, observedEtag? }`), `ownershipPolicyVersion`, `includeDeletes`, `createdByUserId`, `status` (`approved|running|done|superseded`).
    - `mdm_push_run` — `manifestId`, `publisherUserId`, `status` (`running|success|partial|suspended`), counts, timestamps; partial unique index: one `running` per manifest.
    - `mdm_push_run_record` — `runId`, `targetId`, `operationIndex`, `state` (`pending|in_flight|ok|failed|skipped|conflict|uncertain`), `httpStatus`, `errorBody`, `observedEtag`, timestamps.
    - `mdm_upstream_source`, `mdm_drift_report`, `mdm_drift_settings` — created now, used in Stages 3–4.
16. Initial migration generated and run.
17. **[AR-8]** Resource registry `packages/mdm-api/src/registry/`: entries per **concrete endpoint**, generated at boot from the pinned data standard's OpenAPI document plus a hand-written overlay: `{ resourceKey, schema, path, naturalKeyFields | perSubtype, keySchemaVersion, ownedPaths: 'all' | string[], referenceExtractors, rollForward: { yearFields[], cloneUnchanged: boolean } }`. Stage 0 overlay covers the descriptor family only (`*Descriptors`, key `[namespace, codeValue]`).
18. Check: `nx run mdm-api:serve` boots; registry lists every descriptor endpoint for DS5.

### 0.4 `mdm-api` auth — BFF mirroring SBAA, identity from SBAA
19. **[AR-16]** Reuse SBAA's OIDC *protocol* wiring only (`openid-client` `Strategy`, `init-openid-client.ts`), not its user-provisioning callback. Session: `express-session` + `connect-pg-simple` on the **MDM** DB, cookie `mdm.sid`, host-scoped (no `Domain`), `SameSite=Lax`, `Secure` behind the proxy (`trust proxy`). Callback `${MY_URL}/api/auth/callback`; logout revokes the MDM session and redirects to the IdP end-session URL. SSO between SBAA and MDM is via the shared IdP, never by sharing cookies.
20. **[AR-1]** On login, call SBAA `POST /mdm/identity { iss, sub }` with the M2M token; store `{ userId, teams[] }` in the session; refresh every 60 s or on 403. Machine callers to `mdm-api` present a bearer token for the same IdP audience; `mdm-api` validates it and calls `POST /mdm/identity { clientId: azp }`.
21. `packages/mdm-api/src/sbaa-client/SbaaClientService` — client-credentials token (`SBAA_M2M_*`), wrapping steps 5, 7, 10; retries with backoff; fails closed.
22. `@MdmAuthorize(privilege)` guard: reads `req.user.teams` for `:teamId`; machine identities are denied `approve` unconditionally.
23. **[AR-16]** CSRF: reject state-changing requests whose `Origin` is not `FE_URL`; credentialed CORS for `FE_URL` only.
24. Check: browser login via Keycloak → `GET /api/me` shows teams + privileges; a bearer machine token resolves; logout clears `mdm.sid` and ends the IdP session.

### 0.5 `mdm-fe` package
25. `nx g @nx/react:application mdm-fe --directory=packages/mdm-fe --bundler=vite`; `.copyme.env.local` → `VITE_API_URL=http://localhost:3334`. Port 4201.
26. Reuse `@edanalytics/common-ui` (`theme`, `PageTemplate`, `PageActions`, `SbaaTableAllInOne`, `Icons`, `confirmAction`). Copy `packages/fe/src/app/Layout/{AppBar,Nav,NavButton,Breadcrumbs,StandardLayout}.tsx` into `packages/mdm-fe/src/app/Layout/`; nav tree = year sets → resource families (per `mockup/mdm-prototype.html`).
27. `routes/paths.ts` mirrors `as/:asId`: `/as/:asId`, `/as/:asId/years/:year`, `/as/:asId/years/:year/:resourceKey`, `/as/:asId/change-requests/:id`, `/as/:asId/push`, `/as/:asId/runs/:id`, `/as/:asId/drift`, `/as/:asId/sources`, `/as/:asId/settings` (namespaces, managed scopes).
28. Check: login → Home shell renders with team selector and empty inbox.

### 0.6 Local dev and deploy
29. **[AR-16]** `docker-compose.yml`: add `mdm-db` (postgres:14, port 3307, db `mdm`). `sbaa-keycloak-config.local.yml`: client `mdm`, `rootUrl http://localhost:3334`, redirect URIs `http://localhost:3334/api/auth/callback` **and** `http://localhost:4201/*`, post-logout `http://localhost:4201/`.
30. `package.json` `start`: `nx run-many -p api fe mdm-api mdm-fe -t serve`.
31. Deploy: `packages/mdm-api/buildspec.yml`, `Dockerfile.mdm-api`, `Dockerrun.mdm.aws.template.json`; CloudFormation `mdm-beanstalk.yml`, `mdm-rds-postgres.yml`, `mdm-code-pipeline.yml`, `mdm-s3-cloudfront.yml` (hosts `mdm.<env>.startingblocks.org`, `api.mdm.<env>…`). Secrets Manager: OIDC client, M2M client, session secret.
32. `.github/workflows/{build,lint,test}.yml`: add both packages.
33. **Exit Stage 0:** `mdm-api` in dev authenticates an SBAA user through step 10; `mdm-api` calls SBAA `GET /api/teams/:teamId/mdm/targets?School_Year=2026` (step 5) through `SbaaClientService` and gets the pilot's owned, tagged ODSs with fingerprints; credentials for one ODS are retrievable after a restart without rotation.

---

## Stage 1 — Descriptors vertical slice

### 1.1 Ed-Fi connectivity
34. **[AR-2, AR-9]** `edfi/ods-api.client.ts` per target: OAuth2 client-credentials at `target.oauthUrl` using credentials from step 7 (in-memory only, refreshed on 401, falls back to `previousSecret` during rotation overlap). Methods: `getPage(path, { offset, limit, totalCount: true })` returning rows + `Total-Count`, `post`, `put(path, id, body, { ifMatch })`, `delete(path, id, { ifMatch })`, `dependencies()`, `openApi()`, `fingerprint()` = `{ apiVersion, dataStandard, extensions[], profiles[] }` from `/metadata`.
35. **[AR-9]** `edfi/openapi.service.ts`: cache the resources document per **target fingerprint**, not per version; `validate(resourceKey, payload)` with `ajv`. Target compatibility = fingerprint's `dataStandard` equals the year set's pinned standard **and** every registry resource exists in the target document **and** the MDM application is unprofiled (profiles → block with reason).
36. Check: unit tests on recorded fixtures from the pilot (`__fixtures__/ds5.2-tpdm/{openapi,dependencies,descriptors-page-*}.json`) plus a hand-edited `ds5.2-core` variant without TPDM; the pair proves compatibility blocking.

### 1.2 Records, scopes, change requests
37. `year-sets/` module: `POST /teams/:teamId/year-sets { schoolYear, dataStandard }`, `GET`, `PATCH` status. **[AR-17]** Records cannot exist without a year set.
38. **[AR-3]** `managed-scopes/`: CRUD on `mdm_managed_scope`; `namespaces` setting proxies step 8. Import and change items outside scope are rejected with the offending key; imported rows outside scope are stored read-only (`external: true`) for diff visibility only.
39. `records/`: `GET /teams/:teamId/year-sets/:id/:resourceKey` (paged, filtered), `GET …/:hash`, `GET …/:hash/versions`. Guard `team.mdm:read`.
40. **[AR-7, AR-6]** `change-requests/`: `POST` (draft), `POST /:id/items` (validates `after` with OpenAPI; records `baseRevision` from current record or `expectAbsent`; adds caller to contributors), `DELETE /:id/items/:itemId`, `POST /:id/submit` (freezes items, stores `submittedHash`; further edits require `reopen`), `POST /:id/approve`, `POST /:id/reject`, `POST /:id/reopen`. **Approve** = one transaction: assert `submittedHash` unchanged, CAS every item's `baseRevision` against `mdm_record.revision` (and absence for creates), reject the whole batch as `stale` on any mismatch, apply items, bump revisions, write versions. SoD: approver ∉ contributors, machine identities cannot approve, team admins not exempt. Tombstones are the **only** way a delete can reach a manifest.
41. **[AR-8, AR-13]** Import job `jobs/import-from-ods.job.ts`: page every registry descriptor endpoint with `totalCount=true`; abort as `incomplete` if `Total-Count` changes between pages or any page fails; normalise each row to write-shape (strip `id`, `_etag`, `link`, `*Reference.link`), derive `canonicalKey`, dedupe by canonical key; rows outside managed scope → `external` records; rows inside → change items vs. published. Endpoint `POST /teams/:teamId/year-sets/:id/import { targetId, resourceKeys[] }` → `{ changeRequestId, jobId }`.
42. Check: pilot imports → draft CR → submit → approve as a second user → published; approving a second CR with a stale `baseRevision` is rejected; an item outside the namespace scope is refused.

### 1.3 Targets, manifests, push
43. **[AR-5, AR-9, AR-14]** `targets/` behind `TargetProvider` (`resolve(teamId, filters) → Target[]`, `credentials(target)`, `fingerprint(target)`). `Target = { id, label, dataUrl, oauthUrl, metadataUrl, odsApiVersion, dataStandard, schoolYear, environmentType, tags, fingerprint, provider: { kind: 'sbaa', sbEnvironmentId, edfiTenantId, odsId, odsInstanceId } }`. v1 implementation `SbaaTagTargetProvider` wraps step 5. `GET /teams/:teamId/targets?...` returns targets with `compatible: boolean, reason?` per step 35.
44. **[AR-4, AR-3]** `manifests/` — `POST /teams/:teamId/manifests { yearSetId, targetIds[], includeDeletes }`: for each compatible target read current state (step 41's paging rules), compute per operation: `create` (in MDM, absent in target, inside scope), `update` (owned paths differ; payload = target current representation with owned paths replaced; record `observedEtag`), `tombstone → delete` (only if an approved tombstone exists **and** `includeDeletes`), `external` (non-owned differences, display only). Persist as immutable `mdm_push_manifest` with `publishedRevision` and target fingerprints. Guard `team.mdm:publish`.
45. **[AR-10]** Ordering: `/dependencies` resource rank for creates/updates; deletes in reverse rank. Stage 1 has no intra-set references (descriptor mappings arrive in Stage 3), so Stage 1 claims **409 reporting only**; record-level reference edges and reverse-direction delete blocking are specified in step 60.
46. **[AR-4, AR-12, AR-14]** `POST /teams/:teamId/manifests/:id/apply` → run. Preconditions (CAS, single transaction): manifest `status = approved`, `publishedRevision` = current max revision of the year set, no other `running` run for the manifest. Then per target: re-fetch fingerprint, **suspend** the target's remainder if it changed; per operation set `in_flight` → write with `If-Match: observedEtag` (updates/deletes) → `ok` on 2xx, `conflict` on 412, `failed` on other 4xx/5xx with body, `uncertain` if the response is unknown (timeout/crash: a startup sweep marks stale `in_flight` rows `uncertain`). Run status: `success` | `partial` (any failed/skipped/conflict/uncertain) | `suspended`. Manifest → `done` or `superseded` when the year set's revision advances.
47. **[AR-12]** `POST /teams/:teamId/runs/:id/rerun` — same manifest, only `failed|skipped|uncertain` operations: `uncertain` first GETs the record by natural key and marks `ok` if the intended state is present; no re-diff; `conflict` requires a new manifest.
48. Ownership policy `registry/ownership.v1.ts`: descriptor family `ownedPaths: 'all'`; `ownershipPolicyVersion = 1`.
49. Check: push to pilot dev ODS; an intentional tombstone of a referenced descriptor yields a 409 `failed` row; a SIS edit between manifest and apply yields `conflict` on that record and nothing is overwritten; killing the worker mid-run leaves `uncertain` rows that rerun resolves without duplicates.

### 1.4 `mdm-fe` screens
50. Home inbox: needs-your-review, your drafts, ready-to-push (approved manifests), suspended/partial runs, drift alerts (Stage 4).
51. Year-set page; descriptor tables per endpoint grouped as one "Descriptors" family; edit/delete → draft CR; create modal; out-of-scope rows shown read-only with a scope badge.
52. Change request page: items with base revision, Submit / Approve / Reject; `stale` rejection shows which items moved.
53. Push: Targets (compatibility column, untagged list) → Manifest preview (owned vs external, tombstones only when opted in, ETag observed) → Run (states incl. `conflict`/`uncertain`, Rerun, suspended banner with reason).
54. Settings: namespaces, managed scopes.
55. Check: UAT with pilot approver + publisher.

### 1.5 `sbaa_api_client`
56. `session.py`: `mdm_api_base_url` per environment. `client.py`: `list_year_sets`, `list_targets`, `list_mdm_records`, `create_change_request`, `add_change_items`, `submit_change_request`, `approve_change_request` (fails for machine identities by design), `import_from_target`, `create_manifest`, `apply_manifest`, `get_run`, `rerun`. Machine callers are resolved by `mdm-api` via step 20.
57. README + `examples/mdm_push.py`.
58. **Exit Stage 1:** pilot import → approve → manifest → apply to dev ODS; outcomes visible in UI and via `get_run`; a conflict and an uncertain case are demonstrated.

### 1.6 Validation
59. Nightly `mdm-e2e.yml` against the EA internal pilot ODS (7.3 / DS 5.2 / TPDM): import → approve (test approver ≠ test editor) → manifest → apply fixed fixtures → assert `success`; a second job injects a concurrent edit and asserts `conflict`. **[AR-13, decided]** Bulk imports and manifests run in a writer-quiescence window (pilot's SIS integrations idle); `Total-Count` drift between pages aborts as `incomplete`. No snapshot/change-query path in v1.

---

## Stage 2 — EdOrgs
60. **[AR-10]** Record graph: registry `referenceExtractors` per resource (reference objects *and* descriptor URI fields); manifest builds record-level edges; creates/updates skip dependents of failures; **deletes block parents when a child delete fails** (reverse direction); unmanaged blockers (409 from data outside MDM) reported as `failed` with the API body.
61. **[AR-11]** Ownership at JSON paths; **create = full stored payload** (everything imported, including non-owned fields, seeds a new target); **update = owned paths replaced in the target's current representation**; natural keys immutable in v1 (key change = tombstone + create, explicitly reviewed). Document key-unification constraints for the EdOrg/assessment pairs in scope.
62. Registry overlay: EdOrg subtypes with per-subtype keys; `ownedPaths`: `nameOfInstitution`, `shortNameOfInstitution`, parent references, `educationOrganizationCategories`; managed scope kind `edorgSubtree`.
63. Claimset `mdm-v2` adds EdOrg endpoints; `ensure` reconciles.
64. Tree read-model view `mdm_edorg_tree`; `GET …/edorgs/tree`.
65. `mdm-fe`: tree editor; owned/external marking in preview.
66. Check: LEA + schools created in order into an empty dev ODS from imported full payloads; renaming a school pushes; a SIS website change survives; a failed child delete leaves its parent.

## Stage 3 — Remaining resources, years, intake
67. **[AR-18]** Enumerate concrete endpoints per family (programs; assessments + objectiveAssessments; chartOfAccounts + dimension endpoints; certifications only where TPDM is in the target fingerprint; descriptorMappings). Claimset `mdm-v3`.
68. **[AR-18]** Generic UI uses a schema-form library (`@rjsf/chakra-ui` or agreed equivalent) with nested objects, arrays, and reference pickers; round-trip preservation of untouched fields. Flat-only fallback removed from scope.
69. **[AR-17]** Roll-forward `POST /teams/:teamId/year-sets/:from/roll-forward { to }`: creates the target year set (pinned standard chosen explicitly), clones published records into a draft CR applying each registry entry's `rollForward` rule (school-year fields rewritten; fiscal-year fields left or rewritten per overlay; year-less resources cloned unchanged); cross-standard → `mapping/ds4-to-ds5.ts` with unmapped required fields surfaced as item errors.
70. File import (`multipart`): Ed-Fi JSON arrays per endpoint; CSV for descriptors and flat EdOrg attributes; per-line validation; → draft CR within managed scope.
71. **[AR-19]** HTTP-API upstream sources — acceptance criteria before shipping: `mdm_upstream_source` rows are team-bound; `secretRef` must be an MDM-owned Secrets Manager entry tagged with the team id (validated on save and on use); destinations limited to an allow-list of HTTPS origins per team; DNS/IP resolved and checked against private/link-local ranges before connect; redirects not followed cross-origin; credentials never forwarded on redirect; response size and time bounded; mappings support nested object/array construction; pulls record `partial` with per-row errors. Pull → draft CR.
72. `sbaa_api_client`: `roll_forward`, `import_file`, `pull_upstream`.
73. Check: all enumerated endpoints pushable to the pilot (Certifications via TPDM); a 2024 year set imported from the **EA-hosted 6.x/DS4 sandbox ODS** (stood up from recorded fixtures) rolled into DS5 with a mapping report and pushed; a nested assessment created through the form; connector rejects a private-IP destination.

## Stage 4 — Drift scanning (stretch)
74. `mdm_drift_settings` per team; scheduled read-only manifest build (no `approved` status, no apply path) persisted as `mdm_drift_report`.
75. Endpoints + UI; optional approver notification.
76. Check: pilot nightly scan surfaces a SIS-side rename of an owned field.

---

## Release order **[AR-15]**
1. SBAA release: migration (roles, `mdm_integration_credential`), privileges, `/mdm` endpoints. Backward compatible; no MDM dependency.
2. Per-ODS provisioning: `ensure` for each pilot ODS; verify claimset fingerprint and effective access (`GET` one descriptor endpoint with the new app).
3. MDM release: schema migrations, `pgboss_mdm` on the MDM connection, workers. Queue payloads carry `schemaVersion`; workers reject newer payloads and drain older ones.
4. Feature enablement per team (`mdm_year_set` creation requires a team flag held in SBAA team settings).
5. Rollback: MDM workers can be stopped independently; SBAA endpoints are additive. Claimset version bumps are deployed by step 2 before the MDM release that needs them.

## Cross-cutting
- **Tests:** Jest per package; fixtures over live calls; nightly e2e (step 59) including conflict and uncertain cases.
- **Observability:** audit rows for approve, manifest, apply, rotate, scope and namespace changes, with the acting SBAA `userId`.
- **Security:** ODS credentials custody in SBAA (step 6), retrieved service-to-service, in memory only; `mdm-api` service identity holds only the three `mdm:*` privileges; human endpoints never expose secrets; host-scoped sessions + Origin checks (steps 19, 23).
- **Seams for Scope §7.1 (standalone mode):** `TargetProvider` (step 43); `Target` identity is URLs + fingerprint, credentials only via `TargetProvider.credentials`, SBAA ids only under `provider`; the authorization guard consumes `req.user.teams` and nothing else knows it came from SBAA (step 20); manifests and runs take `Target[]`, never ODS ids.
- **Docs:** `docs/mdm/` — architecture, tagging checklist, namespace/scope setup, release order runbook, pilot capability matrix (closed: EA internal tenant, ODS/API 7.3, DS 5.2, TPDM installed, MDM app unprofiled; DS4 via EA-hosted 6.x sandbox; quiescence window for bulk reads).
