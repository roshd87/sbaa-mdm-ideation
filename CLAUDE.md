# CLAUDE.md — MDM Ideation

Context for AI sessions working in this repo. Read fully before editing anything.

## What this repo is

Planning record for adding Master Data Management (MDM) to the StartingBlocks Admin App (SBAA). No application code lives here. Every decision is closed as of 2026-10-06; the documents are the source of truth, in this precedence for *intent* → *shape* → *mechanics*:

| File | Role | Edit when |
|---|---|---|
| `ProjectScope.md` | 25 numbered decisions (Q1–Q25, plus Pilot and Inventory), each with rationale; §4 In Scope, §5 Out, §6 Risks, §7 Future | a decision changes or a new one is made |
| `ProjectPlan.md` | One page, five stages, exit criteria, work by workstream, risks | stage content or sequencing changes |
| `ImplementationPlan.md` | 76 ordered steps against the real SBAA monorepo, each sub-stage ends in a check; release order; seams for future standalone mode | mechanics, names, tables, endpoints change |
| `ProjectPlan.guide.html` | 15-slide walkthrough for a non-engineering audience | any of the above changes |
| `mockup/mdm-prototype.html` | Clickable UI mockup, SBAA chrome, in-memory state, role switcher | UI decisions change |
| `README.md` | Index | files are added or renamed |

Sibling repos (not in this repo; clone next to it if needed): `../startingblocks_admin_app` (Nx monorepo: NestJS `packages/api`, React `packages/fe`, TypeORM `packages/models-server`, `packages/common-ui`), `../sbaa_api_client` (Python). `ImplementationPlan.md` cites real paths in them — verify against the checkout before changing a step.

## Writing style

The plan documents use a controlled technical style. Keep it when editing `ProjectPlan.md`, `ImplementationPlan.md`, the decisions block of `ProjectScope.md`, and the stage slides of the guide. Do not name the standard it derives from in any document; the style speaks for itself.

Rules:
- One instruction or one fact per sentence. Sentences of 20 words or fewer.
- Active voice, present tense, imperative for work items ("Add the roles…", "Create the table…").
- Keep articles (*the*, *a*). Drop filler, hedging, and marketing language.
- One term per concept, always the same term (see glossary). Never a synonym for variety.
- Technical names exact and verbatim in backticks: table names, endpoints, roles, Ed-Fi resource names, HTTP headers.
- Lists over paragraphs. Tables for anything with more than two columns of fact.
- Every work item in `ProjectPlan.md` and `ImplementationPlan.md` ends with its decision reference `(Qn, …)`. Only Q1–Q25 exist.

Where narrative prose is allowed: the ✎ Summary in `ProjectPlan.md`, the *Rationale* lines in `ProjectScope.md`, italic `.prose` blocks in the guide. About 20% of the plan by volume, no more.

Audience-facing HTML (`ProjectPlan.guide.html`): **no** Q-numbers, revision markers, review-item numbers, or references to internal review files. Plain decisions and plain next steps. The engineering docs keep their traceability; the deck does not.

`ProjectPlan.md` must stay one rendered page (≈1000–1100 words by `wc -w`, table pipes included).

## Glossary — use these exact terms

- **year set** — (team, school year) with a pinned data standard, key-schema version, status, published revision. Not "year", not "partition".
- **resource family** — UI grouping (Descriptors, Ed Orgs, …). **resource key** — concrete Ed-Fi endpoint (`gradeLevelDescriptors`). The registry is keyed by resource key.
- **managed scope** — explicit record-level authority: namespace prefix, EdOrg subtree, or all. **owned paths** — field-level authority for updates only.
- **change request** — batch of items in one year set; `draft → submitted → approved | rejected`. Items carry a **base revision**. Submit freezes the batch and stores its hash. Approve is compare-and-swap. **Contributors** cannot approve. **tombstone** — the only kind of delete item.
- **target** — an ODS with URLs, **fingerprint** (API release, data standard, extensions, profiles), tags, compatibility verdict. Resolved by a `TargetProvider`; v1 has one (`SbaaTagTargetProvider`).
- **manifest** — immutable, approved set of operations against named targets at a published revision, with observed ETags. **apply** — one active run per manifest, writes with `If-Match`. Never say "diff job" or "push job".
- **run** — per-operation states `pending → in_flight → ok | failed | skipped | conflict | uncertain`; run status `success | partial | suspended`. **re-run** finishes the same manifest; it never re-diffs.
- **roles** — four single SBAA team roles: `mdm-editor`, `mdm-approver`, `mdm-publisher`, `mdm-admin`. Machine identities never approve.
- **upstream source** — HTTP-API only in v1. **drift scan** — the manifest build, read-only, on a schedule (stretch).

## Decision state (closed 2026-10-06)

Pilot: EA's internal StartingBlocks tenant, ODS/API 7.3 / Data Standard 5.2, TPDM installed, MDM application unprofiled, current school year; external partner joins at Stage 2. DS4 proof: EA-hosted 6.x sandbox seeded from fixtures. Bulk reads in a writer-quiescence window; `Total-Count` drift aborts. Upstream: HTTP API only. UI: SBAA chrome exactly, inbox Home. Four contracts gate Stage 0: identity endpoint in SBAA, credential custody in SBAA, managed scope + tombstone-only deletes, immutable manifest + conditional writes. Full detail: `ProjectScope.md` Q18–Q25.

Do not reopen closed decisions in passing. If a change is needed, add a dated decision to `ProjectScope.md`, then propagate to plan → implementation → guide → mockup, in that order.

## How things were produced (so you can repeat them)

- Decisions came from a one-question-at-a-time interview. Scope changes should go the same way: ask, record, propagate.
- Reviews: a friendly consistency review (adopted), an adversarial review by a second model (19 findings; 4 blockers, all adopted as Q18–Q25), and a final consistency pass. Review files were deleted once folded in; the `[AR-n]` tags in `ImplementationPlan.md` and "AR-n" notes in `ProjectScope.md` are the only trace (n = adversarial finding number).
- The guide and mockup are plain HTML with inline JS. Smoke-test by serving the folder (`python -m http.server` or `Bun.serve`) and loading in a headless browser: check zero console errors, that every hash route renders, and — for the mockup — that an approver cannot approve a change request they contributed to, that approving a stale change request is rejected, and that apply produces a `conflict` row.
- Fonts and icons (IBM Plex Sans, Bootstrap Icons) load from CDNs. Offline renders fall back; that is accepted.

## Working agreements

- Commit locally freely; **ask before `git push`**. The repo is public (`roshd87/sbaa-mdm-ideation`); scan for secrets, emails, and partner data before pushing anything new.
- No LICENSE file yet. Default is all-rights-reserved; Apache 2.0 would match SBAA if one is wanted.
- Keep `mockup/logo-sb.svg` as copied from `startingblocks_admin_app` (Apache 2.0); note the origin in README if the file moves.
- Line endings: LF, enforced by `.gitattributes`.

## Open follow-ups

- Add collaborators or transfer the repo to the `edanalytics` org when the team is ready.
- Decide on a LICENSE.
- Mockup forms are `prompt()` stand-ins; replace when a real form pattern is chosen in Stage 1.
- `ProjectScope.md` Q25 says re-run executes "remaining operations"; the plans interpret that as failed + skipped + uncertain. If operations on a suspended target should need a new manifest instead, say so in Q25.
- Confirm `ProjectPlan.md` renders on one page in the team's Markdown viewer; trim Q-reference parentheses first if it spills.
