# Project Plan: MDM for StartingBlocks

*Format: short, direct sentences throughout — one instruction each — except the ✎ Summary (narrative). Revision 5, 2026-10-06: one page; includes the adversarial-review contracts Q18–Q25.*

## ✎ Summary

This plan delivers `ProjectScope.md` §4 in five stages: Stage 0 lays the foundations; Stages 1–4 are the phases decided in Q15. The plan builds the whole pipeline — intake, change request, review, manifest, push — once, for Descriptors and one pilot partner, then widens it stage by stage without rewiring it. Descriptors go first: flat, fully MDM-owned, and dependency-light, so they exercise every layer with the least UI and give the earliest review feedback. Drift scanning (Stage 4) can start any time after Stage 1.

| Stage | Exit criterion | Size |
|---|---|---|
| 0 Foundations | `mdm-api` runs in dev. It authenticates an SBAA user. It resolves the pilot team's tagged ODSs through SBAA. | M |
| 1 Descriptors slice | The pilot imports Descriptors from an ODS, approves a change request, and pushes to a dev ODS. Outcomes show in the UI and through `sbaa_api_client`. | L |
| 2 EdOrgs | MDM pushes the EdOrg hierarchy in dependency order. The manifest preview shows vendor-owned fields. MDM does not overwrite them. | M |
| 3 Remaining + years | All seven resources are pushable. One year rolls from 6.x/DS4 to 7.x/DS5. File intake and upstream intake land as change requests. | L |
| 4 Stretch: drift | The pilot receives a scheduled drift report. The UI flags the drifted records. | S |

**Decisions.** Pilot: EA internal tenant on ODS/API 7.3 / DS 5.2 with TPDM, MDM app unprofiled, current school year; external partner joins at Stage 2 (Q15). DS4 proof: EA-hosted 6.x sandbox from fixtures (Q9). Bulk reads run in a writer-quiescence window; count drift aborts (Q7). Upstream sources: HTTP API only; direct database access deferred (Q8a).

## Stage 0 — Foundations

- **SBAA API:** Add roles `mdm-editor`, `mdm-approver`, `mdm-publisher`, `mdm-admin`. Add the identity endpoint, the per-ODS application `ensure` with SBAA-held credentials, the service-only credential read and rotation, the namespace setting, and owned-ODS target listing with fingerprints (Q5, Q6, Q18, Q19, Q20, Q22).
- **mdm-api:** Create the NestJS package with its own TypeORM datasource and pg-boss schema. Use a host-scoped cookie-session BFF; resolve identity through SBAA; use bearer tokens only for service calls (Q2, Q10, Q18).
- **mdm-api:** Create `mdm_year_set`, `mdm_record`, `mdm_record_version`, `mdm_managed_scope`, `mdm_change_request` (+ contributors), `mdm_change_item`, `mdm_push_manifest`, `mdm_push_run`, `mdm_push_run_record`. Build the registry from the pinned OpenAPI document, one entry per concrete descriptor endpoint (Q3, Q4, Q20, Q21, Q23, Q24).
- **mdm-fe:** Create the React package on `common-ui`. Reproduce the SBAA chrome. Wire the login (Q2, Q17).
- **Infra:** Add the MDM Postgres, the Keycloak `mdm` client, and the Nx serve targets to local dev. Create the deployed database, pipeline, and Secrets Manager access (Q2).

## Stage 1 — Descriptors slice

- **mdm-api:** Add year sets. Fingerprint each target; block incompatible targets. Add the import job with count-drift detection and write-shape normalisation; write the result as a change request inside managed scope. Add the change-request lifecycle with base revisions, frozen batches, CAS approval, and contributor-based separation of duties (Q3, Q8, Q9, Q13, Q13a, Q20, Q21, Q22, Q24).
- **mdm-api:** Build immutable manifests: create/update by owned paths, deletes from approved tombstones only, observed ETags, target fingerprints. Apply one manifest at a time with `If-Match`; suspend a target whose fingerprint changed; order by `/metadata/data/v3/dependencies` (Q5, Q7, Q7a, Q21).
- **mdm-api:** Record `ok | failed | skipped | conflict | uncertain` per operation. Mark the run `partial` or `suspended`. Re-run only failed, skipped, and `uncertain` operations of the same manifest; reconcile `uncertain` by natural key first. Make every descriptor path MDM-owned (Q12, Q14, Q25).
- **mdm-fe:** Build the inbox Home, the Descriptor editor, the review screen with one batch diff, the target confirmation, the manifest preview, and the run results (Q3, Q5, Q7, Q12, Q13, Q17).
- **sbaa_api_client:** Add `mdm_api_base_url`. Add year-set, target, record, change-request, import, manifest, apply, run, and re-run wrappers (Q11, Q18).
- **Validation:** Record HTTP fixtures for unit tests. Run one nightly end-to-end test against the pilot dev ODS. Do not run an ODS/API in CI. Run the review-UX UAT (Q6, Q12, Q13).

## Stage 2 — EdOrgs

- **SBAA API:** Extend the claimset to the EdOrg resources (Q6).
- **mdm-api:** Add the EdOrg registry entries with one natural key for each subtype (`schoolId`, `localEducationAgencyId`, …). Build the tree read-model. Add the mixed ownership lists; show non-owned fields only for visibility. Rely on `/dependencies` to place LEA before School. Add record-level reference edges; skip dependents of failures; block a parent delete when a child delete fails (Q3, Q7a, Q12, Q14).
- **mdm-fe:** Build the tree editor. Mark owned and non-owned fields in the manifest preview (Q3, Q14).
- **Docs:** Write the claimset hard-enforcement guidance for fully-owned resources (Q14).

## Stage 3 — Remaining resources, years, intake

- **SBAA API:** Extend the claimset to Programs, Assessments, Chart of Accounts, Certifications, and Descriptor Mappings (Q1, Q6).
- **mdm-api:** Add the five registry entries. Add the roll-forward job N → N+1. Write the 6.x/DS4 → 7.x/DS5 mapping. Add file import: Ed-Fi JSON for all; CSV for Descriptors and flat EdOrg attributes. Add the HTTP-API upstream connector with SSRF and secret-ownership controls and one mapping for each source; pull on demand; write a change request (Q1, Q3, Q4, Q8, Q8a, Q9).
- **mdm-fe:** Build the schema-driven list and form from OpenAPI with a schema-form library. Build the roll-forward, file upload, and upstream-source screens (Q3, Q4, Q8, Q8a).
- **sbaa_api_client / Infra:** Add roll-forward, file-import, and upstream-pull wrappers. Store upstream credentials in Secrets Manager (Q8a, Q11).

## Stage 4 — Stretch: drift scanning

- **mdm-api:** Add a scheduled per-partner scan that reuses the manifest build, read-only. Persist drift reports. Add opt-in, schedule, and scope settings (Q14a).
- **mdm-fe / Docs:** Flag drifted records. Build the report screen. Write the off-peak guidance. Measure the ODS read load (Q14a).

## Sequence and risks

- Stage 0 blocks all `mdm-api` auth and push work. The change-request lifecycle blocks every intake path. The manifest builder blocks push and drift. Each claimset extension ships before MDM pushes that resource (Q5, Q6, Q7, Q10, Q11, Q13a, Q14a).
- **Untagged ODSs are invisible.** Audit tags before each stage. Show the target list before each push (Q5).
- **The review UI is the riskiest UX.** Validate it with the pilot in Stage 1 (Q13, Q15).
- **Partial pushes occur.** Persist per-record outcomes. Re-run the same manifest only (Q12, Q25).
- **Version mappings are hand-written.** Write them only for version pairs StartingBlocks runs. Apply them only at roll-forward (Q9).
