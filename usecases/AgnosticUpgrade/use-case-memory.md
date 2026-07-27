# Use-Case Memory: Agnostic Upgrade

> Vendor-agnostic network device software upgrade orchestration, driven by a NetBox plugin catalog of per-device command sets/sequences rather than hard-coded per-vendor logic.

**Stage:** `build` (documented via `/project-to-spec`; known defects found — see Gotchas — before this can be called `delivered`)
**Status:** `active`
**Last updated:** 2026-07-22
**Platform:** https://ws-solution-design-poc-iap01.trial.itential.io

> **Resuming this use-case?** `Stage` tells you where to pick up — but verify it against what files actually exist before trusting it (a stale field is worse than no field). See AGENTS.md's "Resuming a Use-Case" table for the file-presence check per stage.

---

## Platform References

| What | Value |
|------|-------|
| Platform URL | https://ws-solution-design-poc-iap01.trial.itential.io |
| Project name | Agnostic Upgrade |
| Project ID (`_id`) | `6a60d7316556606de51220d1` |
| Project `iid` | 408 |
| Auth method | oauth (client_credentials) |
| Adapter instance(s) | `netbox-grid` (NetBox), `Email` (EmailOpensource) |
| Adapter app type(s) | `Netbox`, `EmailOpensource`, `GitHub` |
| NetBox custom plugin endpoint | `GET plugins/upgrade-catalog/upgrade-bundles` — returns per-device upgrade bundle (check/upgrade/rollback command sets, validation rules, images, diff-ignore patterns) |
| Related project: BMC-Helix | `_id 6882500f543cc2c6724d7bd7`, `iid 410` — holds `Get_CRQ_CI_List`/`Get_CRQ_Status`, called by `Fetch CRQ Details` (not wired into main flow) |
| Related project: Netbox | `_id 698e25203d6a63dd1a8b8518`, `iid 409` — unrelated content (Fetch NetBox Upgrade Bundle, Pre/Post-Flight Validation, Stage/Execute Upgrade), investigated as a red herring during this session |
| Unresolved cross-project ref | `_id 6984ff4371ab83a64cbf75c9` — holds `TAD Transfer File - IOS-XR`, called from the legacy/unwired `Retrieve from GitHub` workflow. Exists but access-restricted to this client; never resolved this session. |
| Broken cross-project ref | `_id 698e67953d6a63dd1a8b851a` — target of `pushFileWithTextContentToNexus` childJob calls. Returns "Project not found" — genuinely missing, not an ACL issue. |
| Key workflow UUIDs | `Upgrade Wrapper: d896d8e6-8a38-42da-b6cc-caf3308d83c3` (entry point) · `Agnostic Upgrade: c4a22ed6-59d4-4019-94ae-be64793c7199` (core orchestrator, 64 tasks) · `Get Netbox Device and Context: 90f1de84-6afb-49a7-ae87-e41af86b447d` · `File Transfer: d88ecc9e-05d8-4e2e-87ea-a14f48a0764d` |
| Trigger | JSON Form `Upgrade Form` (`iid 88`, ref `6a4579e34622df91e6b08fe9`) — fields `version`, `devices`, `bootMode`, `emails`; matches `Upgrade Wrapper` input schema exactly |

---

## What Was Built

| Asset | Type | ID / Name | Status |
|-------|------|-----------|--------|
| Upgrade Wrapper | workflow | d896d8e6-8a38-42da-b6cc-caf3308d83c3 | existing (production entry point) |
| Agnostic Upgrade | workflow | c4a22ed6-59d4-4019-94ae-be64793c7199 | existing (core orchestrator) |
| Get Netbox Device and Context | workflow | 90f1de84-6afb-49a7-ae87-e41af86b447d | existing |
| File Transfer | workflow | d88ecc9e-05d8-4e2e-87ea-a14f48a0764d | existing |
| Create and Run Command Template | workflow | a33634f1-5a02-4dcf-877e-f31ffded1396 | existing |
| Create and Run Verification Command Template | workflow | 4e110285-fbba-494d-aeb0-1c53adbede58 | existing |
| DynamicTemplateCreation (+ with Validations) | workflow | 74872f50-01cf-4ca0-9b91-281fa22a40fb / 73aaefed-d37b-4016-a209-5a1579ef713f | existing |
| Clean Up Template | workflow | 121c23c8-30fd-4008-bad8-b89d9482b036 | existing |
| Run Commands for Checks and Upgrades / for Validations | workflow | 3118ab8b-976b-47d0-afc4-d5fdb4b2dd77 / 6be1b700-82f5-40c2-a14a-5e57e2ef91ce | existing |
| Take Device Backup | workflow | f49e1ea2-d37d-4f6a-b712-f993498ac676 | existing (standalone) |
| Fetch CRQ Details | workflow | 798283e5-cdf0-4b78-b0c1-089c40df3e19 | existing (unwired) |
| 11 legacy GitHub-manifest workflows (Software Upgrade[TAD], Execute Upgrade Steps[TAD], Retrieve from GitHub, getGithubFile, run command loop, Run Steps with File Contents, Create Blob, Create Tree and commit, Update Version TXT, Run Transfer Scripts, Upgrade POC) | workflow | see solution-design.md §B | existing (unwired legacy subsystem) |
| 4 dev/scratch workflows (test, testEmail, test email, update ref test) | workflow | see solution-design.md §B | existing (cleanup candidates) |
| Generate template / with Validations / Fix Template / Create Email for Upgrade / HTML Email | template (jinja2) | see solution-design.md §B | existing |
| Upgrade Form | json-form | 6a4579e34622df91e6b08fe9 | existing |
| 37 transformations (JST) | transformation | see project-components.json | existing |

---

## Architecture Decisions

- **Why NetBox instead of per-vendor hard-coded workflows:** a custom NetBox plugin (`upgrade-catalog`) stores per-device/version command sets, sequences, validation rules, and images. `Get Netbox Device and Context` resolves this once at the top of the orchestrator, making the rest of the workflow vendor-agnostic by construction.
- **Why dynamic MOP templates instead of static per-vendor command templates:** `DynamicTemplateCreation` renders a Jinja2 template into a MOP command-template payload, imports it, runs it, then deletes it (`Clean Up Template`). Avoids a proliferation of static templates per platform/step.
- **Why a bounded reattempt loop instead of a fixed sleep:** after upgrade/reload, `Alive?` is checked with `Increment Reattempt` + `Delay a Job`, capped at 10 attempts — balances giving the device time to reboot against not hanging the job forever.
- **Why a manual ViewDiff gate:** lets an engineer reject a technically-successful upgrade based on the pre/post diff before the job is marked done — human judgment call baked into the automation, not fully hands-off.
- **Why HA-pair support is designed as a wrapper (`HA Upgrade Wrapper`) instead of modifying `Agnostic Upgrade`:** surfaced while evaluating a Palo Alto extension (HA firewall pairs need ordered suspend/upgrade/verify/failover, which the single-device orchestrator can't express). Decided to keep `Agnostic Upgrade` completely unmodified and put all pairing/ordering/abort logic in a new wrapper that calls it twice — lower risk than touching the proven core. Engineer confirmed two safety decisions: (1) abort before touching the Active member if the Passive leg fails/rolls back — never touch the device carrying live traffic if its peer isn't healthy; (2) keep `ViewDiff` approval per-leg inside `Agnostic Upgrade` rather than building a combined cross-member approval — zero changes to the core workflow. Full design in `solution-design.md` §J — not yet built, not testable in this sandbox (no HA-capable adapters configured).

---

## Gotchas Hit

- **Issue:** Direct `GET /automation-studio/projects/{_id}` and `/export` for project 408 returned `"Insufficient permissions to view project"`.
  **Root cause:** this OAuth client (`6a5deb10f7dbd835875b4155`) was not on the project's ACL. Itential projects use per-project ACLs only — no platform-wide admin role.
  **Fix:** engineer granted ACL access via the UI; confirmed by re-querying.

- **Issue:** `408` in the user's request is the project's `iid` (sequential integer), not its Mongo `_id`. `GET /automation-studio/projects/{iid}` works — the endpoint accepts either.
  **Root cause:** none — just worth remembering for future numeric "project id" references from users.

- **Issue:** Every childJob task inside `Agnostic Upgrade` (and several children) references a workflow via a string like `"@6994dbcc6d90cb331ac67eb3: File Transfer"` — a project `_id` that is neither this project's own `_id` nor accessible to this client.
  **Root cause:** the project appears to have been cloned/duplicated from another project without re-pointing childJob `workflow` fields to the new project's own copies. A workflow with the exact matching bare name exists locally in project 408 for all but 2 of these references.
  **Fix (not yet applied — documentation only this session):** each affected childJob task's `variables.incoming.workflow` field needs to be rewritten to `@6a60d7316556606de51220d1: <name>` to guarantee resolution. Flagged in solution-design.md §F as a build defect, not fixed in this session (project-to-spec is documentation-only).

- **Issue:** Two of those cross-project references could not be resolved even after ACL troubleshooting: `pushFileWithTextContentToNexus` → project `_id 698e67953d6a63dd1a8b851a` returns "Project not found" to this client, and `TAD Transfer File - IOS-XR` → project `_id 6984ff4371ab83a64cbf75c9` remained access-restricted throughout the session.
  **Root cause:** unresolved for the second one. For the first: **corrected after reviewing two real completed job exports (`job1.json`/`job2.json`, engineer-supplied)** — both show the Nexus push succeeding in production with live artifacts at `https://nexus.gnscet.com`. The "not found" is an access/visibility gap for this specific test client, not evidence the capability is broken.
  **Fix:** documented as an open item; revisit if/when access is granted. Do not repeat the "Nexus push is broken" claim without re-verifying — it was wrong.

- **Issue:** Both job exports are named `"@6994dbcc6d90cb331ac67eb3: Agnostic Upgrade"` and dated 2026-07-14 (8 days before this documentation pass) — a **different project `_id`** than project 408 (`6a60d7316556606de51220d1`).
  **Root cause:** project 408 is very likely a duplicate/snapshot of the live production project `6994dbcc6d90cb331ac67eb3`, not the live orchestrator itself. This reverses the earlier conclusion that childJob references to `6994dbcc6d90cb331ac67eb3` were "stale" — they were probably correct for the original, and project 408 inherited them as-is when copied.
  **Fix:** solution-design.md §F.1 and §G now carry this correction. Getting access to `6994dbcc6d90cb331ac67eb3` (previously requested, never granted) would let this be confirmed definitively — still an open item.

- **Issue:** Per-task resolved input/output values are not present in a job export — only pointers into a separate `job_data` collection (`incomingRefs`).
  **Root cause:** platform storage design — task-level values are stored by reference, not inlined into the job document.
  **Fix:** only job-scoped top-level `variables` are usable for real-example enrichment from a job export; don't expect to extract per-task resolved payloads this way.

---

## Test Log

| Date | What was tested | Result | Notes |
|------|----------------|--------|-------|
| 2026-07-14 (retrospective, from engineer-supplied job exports) | Agnostic Upgrade, device I27051, version 17.15.05, bootMode Default | pass | 20.5 min, Nexus push succeeded, no rollback/reattempt exercised |
| 2026-07-14 (retrospective, from engineer-supplied job exports) | Agnostic Upgrade, device I27051, version 17.14.01, bootMode install | pass | 31.8 min, Nexus push succeeded, no rollback/reattempt exercised |

No live test execution was performed this session — these are retrospective reads of `job1.json`/`job2.json` supplied by the engineer, not tests run during this engagement. No real example yet exists for validation-failure, rollback, or reattempt-exhaustion branches.

---

## Open Items

- [ ] **HA Pair Upgrade Wrapper is designed (solution-design.md §J, customer-spec.md §11) but not built.** Before building: confirm `ha_resume_command` behavior per vendor (automatic-on-reboot vs. explicit), and whether `Get Netbox Device and Context` should accept a pair directly vs. being called once per member.
- [ ] **Environment gaps confirmed this session (solution-design.md §I) — project 408 would not run as-is on this platform:** `netbox-grid` adapter doesn't exist (real instances: `NetBox`, `Netbox_March_2026`, `Netbox-Maanas`); `Email` adapter_id doesn't case-match real instances (`email`, `Email-2`); the 3 AGManager shell scripts `File Transfer` depends on (`agnostic_load_image.py`, `agnostic_scp.py`, `scp_cisco.py`) aren't registered among the platform's 20 IAG services; zero OM automations reference project 408 at all, so the `Upgrade Form` → `Upgrade Wrapper` trigger wiring is unconfirmed on this platform.
- [ ] Child-job-level real input/output (Get Netbox Device and Context, File Transfer, Create and Run Command Template, etc.) is still undocumented from real execution — solution-design.md §H documents their contractual schemas only (structural, not observed). If engineer can supply child-job export files or access to the platform these jobs actually ran on, upgrade §H to observed data.
- [ ] Grant this client (or resolve via engineer) access to project `_id 6984ff4371ab83a64cbf75c9` to document `TAD Transfer File - IOS-XR`
- [ ] Confirm whether project `_id 698e67953d6a63dd1a8b851a` (Nexus push target) was deleted intentionally or needs to be recreated/re-pointed
- [ ] Rewire all stale childJob `workflow` references in `Agnostic Upgrade` (and children) to the current project prefix `@6a60d7316556606de51220d1:`
- [ ] Decide whether the 11 legacy GitHub-manifest workflows should be archived/deleted or kept as a future path
- [ ] Decide whether `Fetch CRQ Details` (BMC Helix change-request gating) should be integrated into the main flow
- [ ] Confirm whether "no rollback on reconnect-timeout exhaustion" is intentional behavior
- [ ] Clean up the 4 dev/scratch workflows (test, testEmail, test email, update ref test) if confirmed unneeded

---

## Spec Deviations

N/A — `customer-spec.md` for this use case was itself reverse-engineered from the as-built project in this session, so there is no prior independent spec to compare against.
