# Solution Design: Agnostic Upgrade

> **As-Built** — produced by reading project `Agnostic Upgrade` (platform `iid` 408, `_id` `6a60d7316556606de51220d1`) on `https://ws-solution-design-poc-iap01.trial.itential.io`.

## A. Environment Summary

- **Platform:** `ws-solution-design-poc-iap01.trial.itential.io`
- **Project:** `Agnostic Upgrade` — `_id` `6a60d7316556606de51220d1`, `iid` 408
- **Components:** 30 workflows, 37 transformations, 5 templates, 1 JSON form (73 total)
- **Adapters/apps used:** GitHub, Netbox (instance `netbox-grid`), EmailOpensource, ConfigurationManager (`isAlive`, `backUpDevice`, `getDevice`), TemplateBuilder (Jinja2 render), MOP (`RunCommandTemplate`, `MOP.importTemplate`, `MOP.deleteTemplate`), AGManager (Automation Gateway shell scripts: `agnostic_load_image.py`, `agnostic_scp.py`, `scp_cisco.py`)
- **Cross-project dependencies (see §F):**
  - `pushFileWithTextContentToNexus` → target project `_id 698e67953d6a63dd1a8b851a` — returns "Project not found" to this client, **but two real completed job runs (see §G) prove this step succeeds in production**, pushing to `https://nexus.gnscet.com`. The API-level "not found" is most likely an access/visibility gap for this specific OAuth client, not evidence the capability is broken.
  - `TAD Transfer File - IOS-XR` → target project `_id 6984ff4371ab83a64cbf75c9` — exists, access-restricted to this client, not resolved
  - `Get_CRQ_CI_List` / `Get_CRQ_Status` → project `BMC-Helix` (`iid` 410, `_id` `6882500f543cc2c6724d7bd7`) — resolved, confirmed to exist
  - **Correction from job evidence:** two completed production job runs (§G) are both named `"@6994dbcc6d90cb331ac67eb3: Agnostic Upgrade"` — i.e. the workflow that actually executes in production lives in project `_id 6994dbcc6d90cb331ac67eb3`, dated as recently as **2026-07-14**. This means project 408 (`_id 6a60d7316556606de51220d1`) is very likely a **duplicate/snapshot of the live production project**, not the live orchestrator itself. The childJob prefix pointing to `6994dbcc6d90cb331ac67eb3` is therefore probably correct/intentional for the *source* project, not a stale reference — see the revised §F.

## B. Component Inventory

### Workflows (30)

| iid | Name | uuid | Role |
|---|---|---|---|
| 55 | Upgrade Wrapper | d896d8e6-8a38-42da-b6cc-caf3308d83c3 | **Top-level entry point** — wraps `Agnostic Upgrade`, sends result email |
| 22 | Agnostic Upgrade | c4a22ed6-59d4-4019-94ae-be64793c7199 | **Core orchestrator** (64 tasks) — see §D |
| 42 | Get Netbox Device and Context | 90f1de84-6afb-49a7-ae87-e41af86b447d | Resolves per-device upgrade bundle from NetBox |
| 44 | File Transfer | d88ecc9e-05d8-4e2e-87ea-a14f48a0764d | Stages image to device via AGManager shell scripts |
| 40 | Create and Run Command Template | a33634f1-5a02-4dcf-877e-f31ffded1396 | Builds + runs a MOP command template dynamically |
| 68 | Create and Run Verification Command Template | 4e110285-fbba-494d-aeb0-1c53adbede58 | Same, for validation command sets |
| 21 | DynamicTemplateCreation | 74872f50-01cf-4ca0-9b91-281fa22a40fb | Renders Jinja2 → imports as MOP template |
| 66 | DynamicTemplateCreation with Validations | 73aaefed-d37b-4016-a209-5a1579ef713f | Same, validation variant |
| 73 | Clean Up Template | 121c23c8-30fd-4008-bad8-b89d9482b036 | Deletes the dynamically-created MOP template |
| 49 | Run Commands for Checks and Upgrades | 3118ab8b-976b-47d0-afc4-d5fdb4b2dd77 | Wraps Create/Run Command Template for checks/upgrade/rollback/post-check |
| 67 | Run Commands for Validations | 6be1b700-82f5-40c2-a14a-5e57e2ef91ce | Wraps Create/Run Verification Command Template |
| 64 | Take Device Backup | f49e1ea2-d37d-4f6a-b712-f993498ac676 | Standalone backup wrapper (`ConfigurationManager.backUpDevice`) |
| 87 | Fetch CRQ Details | 798283e5-cdf0-4b78-b0c1-089c40df3e19 | Resolves device list from a BMC Helix change request — **not wired into main flow** |
| 29 | Software Upgrade | 695e06de-982e-456c-9c1e-1c4d2b627303 | Legacy/unwired: GitHub-manifest-driven upgrade entry |
| 37 | Software Upgrade TAD | d8cc7b53-f8e7-4ec3-8d1f-10c1e4b956fc | Legacy/unwired: test variant of above |
| 30 | Execute Upgrade Steps | 45611fcc-6d45-43e9-b520-0144abc667d2 | Legacy/unwired |
| 38 | Execute Upgrade Steps TAD | 96ae2cce-a1c7-4286-ac26-c2e8479f9ca8 | Legacy/unwired |
| 31 | Run Steps with File Contents | a1809855-bf71-4744-86a5-d7a78dac31e2 | Legacy/unwired |
| 28 | getGithubFile | 7f47aac3-2f1b-48d5-968b-c5d8a0ee4814 | Legacy/unwired: generic GitHub file fetch |
| 4 | Retrieve from GitHub | 258d0f07-f6f4-4d7b-a0d1-56a3afd8ae27 | Legacy/unwired |
| 5 | run command loop | cec567b8-eb9e-455a-8d1a-a9c81cebe5e2 | Legacy/unwired: MOP `RunCommand` looped over a command list |
| 3 | Update Version TXT | 9e310a06-fb92-4644-b391-4c757d8f921c | Legacy/unwired: GitHub version-tracking file update |
| 10 | Create Blob | a4ad079d-8d1d-471a-855d-f8084a194431 | Legacy/unwired: GitHub git-blob creation (audit trail) |
| 14 | Create Tree and commit | 9cc97957-d816-425d-a45a-85cfa1c520be | Legacy/unwired: GitHub git-tree/commit creation |
| 83 | Run Transfer Scripts | c7be8778-bb22-44a4-9c4e-f671553f3b26 | Legacy/unwired: AGManager script runner |
| 8 | Upgrade POC | 8af2da4b-372c-4662-9876-5599cbf22dcc | Standalone POC/demo workflow (git-based pre/post check tracking + email) |
| 16 | update ref test | 46057b5d-2387-46f4-95e5-df2e8ded59b2 | Dev/scratch |
| 19 | testEmail | 3178c5ff-9bd2-48fd-9bb9-1ea613b1c5ed | Dev/scratch |
| 54 | test | 3577ef51-fe55-40ca-bd60-8fa58f1f9c37 | Dev/scratch |
| 75 | test email | de2af333-1d84-4878-8e87-05c90c4193c6 | Dev/scratch |

### Templates (5, all Jinja2)

| iid | Name | Purpose |
|---|---|---|
| 41 | Generate template | Renders a MOP command-template JSON body from a check-command list |
| 77 | Generate template with Validations | Same, for the validation command template variant |
| 59 | Fix Template | Renders upgrade command sequence (e.g. `bundle_commands`: boot system change, write mem, reload) |
| 57 | Create Email for Upgrade | Renders plain-text result email body |
| 78 | HTML Email | Renders HTML result email body |

### JSON Forms (1)

| iid | Name | Fields |
|---|---|---|
| 88 | Upgrade Form | `version` (string), `devices` (array), `bootMode` (enum: Install/Bundle/Default), `emails` (space-separated string) — matches `Upgrade Wrapper`'s input schema exactly; this is the trigger UI |

### Transformations (37)

Notable ones (full list in `project-components.json`): `getNetboxConfigContext`/`getNetboxConfigContextConfigManager` (shape NetBox response), `Pick Version from Images`, `Make Checks Object`, `Choose Upgrade Commands`, `Filter Validations`, `Validations Success`, `Parse Pre-Post Checks`, `Create Rollback Image`, `Increment Reattempt`, `Parse Emails` (splits the space-separated `emails` string into an array before the email adapter call — correct per data-type handling), `Create Transfer Objects`, `Get Image Name and Path`. The remainder support the legacy GitHub-manifest subsystem (`parse github commands`, `getStepsFromManifest`, `Create Blob Array`, `Get all trees`, etc.).

## C. Adapter Mappings

| Integration | Adapter instance | Tasks used |
|---|---|---|
| NetBox | `netbox-grid` | `genericAdapterRequest` → `GET plugins/upgrade-catalog/upgrade-bundles` (custom NetBox plugin, per-device/version upgrade bundle) |
| GitHub | (GitHub adapter, instance not confirmed) | `getReposOwnerRepoContentsPath`, `putReposOwnerRepoContentsPath`, `postReposOwnerRepoGitTrees`, `postReposOwnerRepoGitCommits`, `genericAdapterRequest` — used only by the legacy/unwired subsystem |
| Email | `Email` (`EmailOpensource`) | `mailWithOptions` — result notifications |
| Configuration Manager | (native platform app) | `isAlive`, `backUpDevice`, `getDevice` |
| MOP | (native platform app) | `RunCommandTemplate`, `MOP.importTemplate`, `MOP.deleteTemplate`, `RunCommand` |
| Automation Gateway | `AGManager` | Shell scripts: `agnostic_load_image.py`, `agnostic_scp.py`, `scp_cisco.py` |
| Template Builder | (native platform app) | `renderJinja2TemplateWithCast`, `renderJinjaTemplate` |

## D. Workflow Structure — Core Path

### Upgrade Wrapper (entry point)
- **Inputs:** `version`, `bootMode`, `devices`, `emails`
- **Flow:** `childJob → Agnostic Upgrade` → transform result into HTML/plain email context → `mailWithOptions` (to = parsed `emails` array)
- **Outputs:** pass-through of inputs; no additional job-level output beyond the email send

### Agnostic Upgrade (core orchestrator — 64 tasks, 73 transitions)
- **Inputs:** `version`, `devices`, `bootMode`
- **Outputs:** `success`, `reattempt`, `emailMessage`, `preCheckCommit`, `postCheckCommit`
- **Observed real inputs/outputs (from two completed production job runs — see §G for full detail):**

  | Field | Job 1 | Job 2 |
  |---|---|---|
  | `devices` | `"I27051"` | `"I27051"` |
  | `version` | `"17.15.05"` | `"17.14.01"` |
  | `bootMode` | `"Default"` | `"install"` |
  | `reattempt` | `1` | `1` |
  | `success` | `"success"` | `"success"` |
  | `emailMessage` | `"Successfully completed upgrade"` | `"Successfully completed upgrade"` |
  | `preCheckCommit.fileName` | `"I27051_pre-checks.cfg"` | `"I27051_pre-checks.cfg"` |
  | `preCheckCommit.html_url` | `https://nexus.gnscet.com/#browse/browse:gsa_netauto:configs/I27051_pre-checks.cfg` | (same pattern) |
  | `postCheckCommit.fileName` | `"I27051_post-checks.cfg"` | `"I27051_post-checks.cfg"` |
  | Job duration (start→end) | 20.5 min | 31.8 min |

  `devices` is confirmed to be a **single device string** (a NetBox device name/ID, e.g. `"I27051"`), not an array, in both observed runs — despite the trigger-level `Upgrade Form` schema typing it as an `array`. Confirm with the engineer whether `Upgrade Wrapper`/`Agnostic Upgrade` is invoked once per device (looped by a caller) or whether array-vs-string handling varies by call site.
  `preCheckCommit`/`postCheckCommit` are both device-backup-style artifacts (`rawFileContent` = concatenated `show` command output, e.g. `sh clock`, `dir /all`) pushed to Nexus repository `gsa_netauto` under path `configs/`, with a `responseObject` of shape `{success, inputVars, outputVars, reason, response}`.
- **Sequence:**
  1. `Merge Data` → **childJob: Get Netbox Device and Context** → `Pick Version from Images` → evaluate `imageFound`
  2. **childJob: File Transfer** (stage image) → evaluate `job_details`
  3. Build checks object → **childJob: Create and Run Command Template** (pre-check command set) → parse results → filter validations → **childJob: Run Commands for Validations** (pre-check) → `Validations Success` evaluation
     - **fail → jump to Rollback (step 7)**
  4. `ConfigurationManager.backUpDevice` → **childJob: pushFileWithTextContentToNexus** ("Push to Nexus" — ⚠ broken, see §F) → add commit URL
  5. Query upgrade commands → `Choose Upgrade Commands` → render Jinja2 → **childJob: Create and Run Command Template** (upgrade command set, respects `bootMode`) → `ConfigurationManager.isAlive`
     - `error` on isAlive call → terminal failure email ("Failed to reconnect to device after upgrade")
     - `Alive?` false → reattempt loop: `Increment Reattempt` → `Delay a Job` → re-check, capped at **10 attempts** → exhausted → terminal failure email, **no rollback attempted**
  6. `Alive?` true → **childJob: Create and Run Command Template** (post-check command set) → parse/filter → **childJob: Run Commands for Validations** (post-check) → `Validations Success` evaluation
     - **fail → jump to Rollback (step 7)**
     - success → **childJob: pushFileWithTextContentToNexus** ("Push to Nexus" — ⚠ same broken dependency) → add commit URL → **manual task: `ViewDiff`**
       - approved → set `success = "success"` → set `emailMessage = "Successfully completed upgrade"` → end
       - rejected → jump to Rollback (step 7)
  7. **Rollback:** `Create Rollback Image` → `Pick Version from Images` (prior image) → **childJob: File Transfer** (re-stage) → evaluate `job_details`
     - success → **childJob: Create and Run Command Template** (rollback command set) → set `emailMessage = "Post Check validations failed and rollback commands have been run"` → end (`success = "failed"`)
     - failure (rollback image transfer itself failed) → set `emailMessage = "...transfer failed. No rollback commands run"` → end (`success = "failed"`)
- **Error handling:** every distinct failure branch sets its own descriptive `emailMessage` before reaching a terminal state-setter task (all of these terminal tasks are named "Success" regardless of outcome — they set the `success` variable to `"success"` or `"failed"` accordingly; the task *name* is misleading but the *behavior* is correct).

### Supporting child workflows
- **Create and Run Command Template** → childJob `DynamicTemplateCreation` (Jinja2 render → `MOP.importTemplate`) → `RunCommandTemplate` → childJob `Clean Up Template` (`MOP.deleteTemplate`)
- **Create and Run Verification Command Template** → same pattern via `DynamicTemplateCreation with Validations`
- **Get Netbox Device and Context** → `genericAdapterRequest` (NetBox) + `ConfigurationManager.getDevice` → returns the full upgrade bundle
- **File Transfer** → `AGManager` shell scripts (`agnostic_load_image.py`, `agnostic_scp.py`) → childJob `Create and Run Command Template` (twice: skip-transfer check, verify-presence check)

## E. Data Flow

Key variables threaded through the core path:
- `version`, `devices`, `bootMode`, `emails` — trigger-level inputs, passed through `Upgrade Wrapper` → `Agnostic Upgrade`
- NetBox bundle fields (`check_command_sets`, `upgrade_sequence`, `rollback_sequence`, `validation_rules`, `images`, `diff_ignore_patterns`, `plugin`) — resolved once at the top of `Agnostic Upgrade`, then referenced by every downstream command-template-building step
- `job_details` — childJob completion signal, evaluated after every childJob call to branch success/failure
- `reattempt` — counter variable, incremented and compared against a static cap of 10
- `success`, `emailMessage` — final job-level outputs, set on every terminal branch, consumed by `Upgrade Wrapper`'s email templates

## F. Known Gaps

1. **Project 408 is very likely a duplicate/snapshot of a live production project, not the live orchestrator itself.** Both completed job runs analyzed in §G are named `"@6994dbcc6d90cb331ac67eb3: Agnostic Upgrade"` and are dated 2026-07-14 — 8 days before this documentation pass. Project 408's own `_id` is `6a60d7316556606de51220d1`, a different project entirely. **Revised interpretation:** the childJob project-prefix references throughout project 408 (e.g. `"@6994dbcc6d90cb331ac67eb3: File Transfer"`) are likely *correct for the original production project* — project 408 is a copy that inherited them, and its own local same-named workflows (File Transfer, DynamicTemplateCreation, etc.) are probably an unused, possibly-diverged duplicate set. This reverses the earlier read of these references as "stale." **Recommendation:** confirm with the engineer whether project 408 is meant to be a working replica (in which case all childJob references need re-pointing to `6a60d7316556606de51220d1`) or a documentation/review snapshot of `6994dbcc6d90cb331ac67eb3` (in which case no rewiring is needed, but this solution-design should be treated as describing the *shape* of the production system, not confirmed to be independently executable as project 408 stands).

2. **Nexus push works in production — earlier "broken" finding was a client-access artifact, not a functional defect.** `pushFileWithTextContentToNexus` (tasks `13db`, `28e2`) targets project `_id 698e67953d6a63dd1a8b851a`, which this API client cannot see ("Project not found"). However, §G's job evidence shows this step completing successfully in real runs, with live artifacts at `https://nexus.gnscet.com/#browse/browse:gsa_netauto:configs/...`. Do not treat this as broken without further confirmation — most likely this client simply lacks visibility into whichever project currently hosts that workflow (possibly inside `6994dbcc6d90cb331ac67eb3` itself, not a separate project as project 408's stale local copy implies).

3. **One childJob target still fully unresolved:** `TAD Transfer File - IOS-XR` (called from the legacy, unwired `Retrieve from GitHub` workflow) → project `_id 6984ff4371ab83a64cbf75c9`, exists but remained access-restricted to this client throughout this engagement. This workflow is only used by the disconnected legacy subsystem (see #6 below), so it does not block understanding the live production path.

4. **No rollback on reconnect-timeout.** If the device never comes back alive after the upgrade reload (reattempt counter exhausts at 10), the workflow ends in failure **without** attempting a rollback. Neither observed job run exercised this branch (both reconnected on `reattempt: 1`). Confirm with the engineer whether the no-rollback behavior is intentional.

5. **Change-management gate not integrated.** `Fetch CRQ Details` (resolves device list + status from a BMC Helix change request, project `iid` 410) is a complete, standalone workflow but is not called from `Upgrade Wrapper` or `Agnostic Upgrade`. If CRQ/change-window enforcement is required before executing an upgrade, it needs to be wired in explicitly.

6. **Legacy GitHub-manifest subsystem is present but fully disconnected.** 11 workflows (`Software Upgrade`[TAD], `Execute Upgrade Steps`[TAD], `Retrieve from GitHub`, `getGithubFile`, `run command loop`, `Run Steps with File Contents`, `Create Blob`, `Create Tree and commit`, `Update Version TXT`, `Run Transfer Scripts`, `Upgrade POC`) implement an alternate design where upgrade steps come from a GitHub-stored manifest/step file with git-blob/tree/commit audit trail, rather than NetBox. None are referenced by the production entry point. Recommend confirming with the engineer whether to archive/delete these or whether they represent a planned future capability.

7. **Dev/scratch workflows in the project.** `test`, `testEmail`, `test email`, `update ref test` appear to be development artifacts with no inputs/meaningful outputs — candidates for cleanup.

8. **Confusing terminal-task naming.** All terminal state-setter tasks in `Agnostic Upgrade` are named "Success" (canvas name), including ones that set `success = "failed"`. The behavior is correct; the naming would confuse anyone auditing the canvas visually.

## G. Observed Production Job Runs

Two completed job records (`job1.json`, `job2.json`, supplied by the engineer) were analyzed to validate this design against real execution. Both are full `Agnostic Upgrade` job exports (not project 408's copy — see §F.1), status `complete`, invoked as a childJob from an `Upgrade Wrapper` run (`parent.task: "039d"`, matching the childJob wiring documented in §D).

| | Job 1 | Job 2 |
|---|---|---|
| Job `_id` | `88c9ea8de09348b3bcb50387` | `6df18228db0a47c489313cd8` |
| Parent job `_id` | `d71454012f9044828e2b8fb8` | `6eaf10277e70438496cbd933` |
| `last_updated` | 2026-07-14T18:24:59.685Z | 2026-07-14T15:58:42.029Z |
| Duration | 20.5 min | 31.8 min |
| `devices` (input) | `"I27051"` | `"I27051"` |
| `version` (input) | `"17.15.05"` | `"17.14.01"` |
| `bootMode` (input) | `"Default"` | `"install"` |
| Outcome | `success` / "Successfully completed upgrade" | `success` / "Successfully completed upgrade" |
| Reattempts used | 1 (reconnected on first check) | 1 (reconnected on first check) |

**Confirmed from real data:**
- Target device (`I27051`) is a Cisco Catalyst 9000 running IOS-XE (`cat9k_iosxe.17.15.05.SPA.bin` observed in `dir /all` output) — consistent with the customer-spec's inference that the concrete implementation is Cisco-IOS/IOS-XE-oriented today, even though the design intends to generalize via the NetBox catalog.
- `devices` is passed as a **plain string** (single device), not an array, at the `Agnostic Upgrade` level in both runs — despite the `Upgrade Form` JSON Form typing `devices` as an array. **Resolved by checking the actual workflow schema (§H):** `Agnostic Upgrade`'s own `inputSchema` types `devices` as `['array','boolean','null','number','object','string']` — i.e. effectively untyped/any — so a single string is schema-valid at this level. Most likely `Upgrade Wrapper` (or something upstream) fans an array out into one `Agnostic Upgrade` childJob call per device; this is consistent with both observed runs targeting the same single device. Still worth confirming the fan-out point directly with the engineer, since it isn't visible in the pulled workflow detail.
- Both `preCheckCommit` and `postCheckCommit` are Nexus-artifact records of shape `{status, _id, initiator, fileName, pathToFile, repositoryName, rawFileContent, responseObject: {success, inputVars, outputVars, reason, response}, html_url}`. `rawFileContent` is concatenated `show`-command text output (`sh clock`, `dir /all`, etc.) — i.e. this is functioning as a lightweight pre/post device-state snapshot, distinct from the `ConfigurationManager.backUpDevice` call elsewhere in the flow.
- Neither observed run exercised a failure or rollback branch — both took the full happy path through to `ViewDiff` approval and `success`. No real example data exists yet for the rollback, reattempt-exhaustion, or validation-failure paths documented in §D.
- Task-level per-run resolved values (individual task inputs/outputs) are **not** embedded in the job export — the platform stores them in a separate `job_data` collection referenced by pointer/`_id` (visible under each task's `incomingRefs`), not inline. Only job-scoped variables (the top-level `variables` object) are directly readable from a job export. Any future job-log analysis should be aware of this before assuming per-task detail is exportable this way.

## H. Child Job Reference — Input/Output Schemas

> **Structural, not observed.** Every job export supplied for this engagement (`job1.json`, `job2.json`) only inlines job-scoped variables at the `Agnostic Upgrade` level (§G) — individual childJob executions (Get Netbox Device and Context, File Transfer, Create and Run Command Template, etc.) are stored by reference in a separate `job_data` collection not included in those exports, and the parent job IDs don't exist on the platform this session is authenticated against, so they couldn't be queried directly either. The tables below are derived from each workflow's own `inputSchema`/`outputSchema` (pulled from the platform in §B/§D) — they describe what each child job is *contractually able to receive and return*, not a captured real execution. Treat "any type" entries (`['array','boolean','null','number','object','string']`) as the platform's way of expressing an untyped/unenforced field, not a real ambiguity in the data.

Ordered by position in the call graph, root first.

### Agnostic Upgrade (core orchestrator)
- uuid: `c4a22ed6-59d4-4019-94ae-be64793c7199`
- **Inputs:** `version` (string, required) · `devices` (any type, required — see note above on why a single string is valid) · `bootMode` (string, required)
- **Outputs:** `version`, `devices`, `bootMode` (echo of inputs) · `_id` (string) · `initiator` (string) · `emailMessage` (string) · `reattempt` (number) · `success` (string) · `postCheckCommit` (object) · `preCheckCommit` (object)

### Upgrade Wrapper (entry point)
- uuid: `d896d8e6-8a38-42da-b6cc-caf3308d83c3`
- **Inputs:** `version` (string, required) · `bootMode` (any type, required) · `devices` (array, required) · `emails` (string, required)
- **Outputs:** echo of inputs · `_id` · `initiator`

### Get Netbox Device and Context
- uuid: `90f1de84-6afb-49a7-ae87-e41af86b447d`
- **Inputs:** `version` (string, required) · `name` (string, required — the device name/ID)
- **Outputs:** echo of inputs · `_id` · `initiator` · `success` (string) · `check_command_sets` (array) · `diff_ignore_patterns` (array) · `images` (array) · `upgrade_sequence` (array) · `rollback_sequence` (array) · `image_transfer` (array) · `validation_rules` (array) · `plugin` (object) — this is the full NetBox `upgrade-catalog` bundle described in §C

### File Transfer
- uuid: `d88ecc9e-05d8-4e2e-87ea-a14f48a0764d`
- **Inputs:** `image` (object, required) · `version` (any type, required) · `devices` (any type, required) · `transferObject` (object, required)
- **Outputs:** echo of inputs · `_id` · `initiator` · `skipTransferResults` (any type) · `transferResults` (any type) · `transferSuccess` (**boolean**)

### Run Commands for Checks and Upgrades
- uuid: `3118ab8b-976b-47d0-afc4-d5fdb4b2dd77`
- **Inputs:** `image` (object, required) · `version` (any type, required) · `step` (any type, required — which command phase: checks/upgrade/rollback/post-check) · `devices` (any type, required) · `object` (object, required)
- **Outputs:** echo of inputs · `_id` · `initiator` · `commandResults` (any type)

### Run Commands for Validations
- uuid: `6be1b700-82f5-40c2-a14a-5e57e2ef91ce`
- **Inputs:** `obj` (array, required) · `version` (any type, required) · `step` (any type, required) · `devices` (any type, required) · `image` (object, required)
- **Outputs:** echo of inputs · `_id` · `initiator` · `commandResults` (any type)

### Create and Run Command Template
- uuid: `a33634f1-5a02-4dcf-877e-f31ffded1396`
- **Inputs:** `obj` (any type, required) · `version` (any type, required) · `step` (any type, required) · `devices` (**array<string>**, required)
- **Outputs:** echo of inputs · `_id` · `initiator` · `templateName` (untyped) · `templateResults` (object) · `success` (**boolean**)

### Create and Run Verification Command Template
- uuid: `4e110285-fbba-494d-aeb0-1c53adbede58`
- **Inputs:** `obj` (any type, required) · `version` (any type, required) · `step` (any type, required) · `devices` (**array<string>**, required)
- **Outputs:** echo of inputs · `_id` · `initiator` · `templateName` (untyped) · `templateResults` (object) · `success` (**boolean**)

### DynamicTemplateCreation
- uuid: `74872f50-01cf-4ca0-9b91-281fa22a40fb`
- **Inputs:** `obj` (any type, required) · `version` (any type, required) · `step` (any type, required)
- **Outputs:** echo of inputs · `_id` · `initiator` · `templateName` (any type) · `success` (**boolean**)

### DynamicTemplateCreation with Validations
- uuid: `73aaefed-d37b-4016-a209-5a1579ef713f`
- **Inputs:** `obj` (any type, required) · `version` (any type, required) · `step` (any type, required)
- **Outputs:** echo of inputs · `_id` · `initiator` · `templateName` (any type) · `success` (**boolean**)

### Clean Up Template
- uuid: `121c23c8-30fd-4008-bad8-b89d9482b036`
- **Inputs:** `id` (**string**, required — the MOP template ID to delete)
- **Outputs:** echo of input · `_id` · `initiator` · `success` (**boolean**)

### Take Device Backup (standalone — not called by Agnostic Upgrade; it calls `ConfigurationManager.backUpDevice` inline instead)
- uuid: `f49e1ea2-d37d-4f6a-b712-f993498ac676`
- **Inputs:** `name` (string, required)
- **Outputs:** echo of input · `_id` · `initiator`

### Fetch CRQ Details (not wired into main flow — see §F.5)
- uuid: `798283e5-cdf0-4b78-b0c1-089c40df3e19`
- **Inputs:** `Infrastructure_Change_Id` (any type, required)
- **Outputs:** echo of input · `_id` · `initiator` · `devices` (untyped) · `success` (**boolean**)

### Legacy GitHub-manifest subsystem (unwired — see §F.6)

| Workflow | uuid | Inputs | Outputs (beyond echo/`_id`/`initiator`) |
|---|---|---|---|
| Software Upgrade | 695e06de-982e-456c-9c1e-1c4d2b627303 | `formData` (object, required) | `pathToFile` (untyped) |
| Software Upgrade TAD | d8cc7b53-f8e7-4ec3-8d1f-10c1e4b956fc | `version` (string, required) | `upgradeSteps`, `manifest` (untyped) |
| Execute Upgrade Steps | 45611fcc-6d45-43e9-b520-0144abc667d2 | `pathTofile`, `fileName` (any type, required) | — |
| Execute Upgrade Steps TAD | 96ae2cce-a1c7-4286-ac26-c2e8479f9ca8 | `pathTofile`, `fileName` (any type, required) | `fileContents` (untyped) |
| Run Steps with File Contents | a1809855-bf71-4744-86a5-d7a78dac31e2 | (none) | — |
| Retrieve from GitHub | 258d0f07-f6f4-4d7b-a0d1-56a3afd8ae27 | (none) | — |
| getGithubFile | 7f47aac3-2f1b-48d5-968b-c5d8a0ee4814 | `fileName`, `pathToFile` (any type, required) | `fileContents` (any type) |
| run command loop | cec567b8-eb9e-455a-8d1a-a9c81cebe5e2 | `command` (**string**, required) | `result` (object) |
| Run Transfer Scripts | c7be8778-bb22-44a4-9c4e-f671553f3b26 | (none) | — |
| Upgrade POC | 8af2da4b-372c-4662-9876-5599cbf22dcc | (none) | — |
| Create Blob | a4ad079d-8d1d-471a-855d-f8084a194431 | `command` (string, required) · `content` (any type, required) | `filePath` (**string**), `sha` (untyped) |
| Create Tree and commit | 9cc97957-d816-425d-a45a-85cfa1c520be | `blobArray` (array, required) | `commitResult` (object) |
| Update Version TXT | 9e310a06-fb92-4644-b391-4c757d8f921c | `content` (any type, required) | — |

**Pattern worth noting:** almost every workflow in this project types its non-trivial fields (`version`, `devices`, `step`, `obj`, etc.) as "any type" rather than a specific JSON Schema type, with only a handful of fields getting real type enforcement (`success`/`transferSuccess` as boolean, `devices` as `array<string>` specifically inside the two command-template builders, `id`/`command`/`filePath` as string). This is consistent with a workflow set built for flexibility during rapid iteration rather than strict contract enforcement — worth flagging if the requirement is for strict input validation at each layer.

## I. Environment Gaps — Things Referenced by the Project but Not Present/Confirmed on the Platform

Everything in §F was found by reading the project's own components. This section is the result of cross-checking those components against the **live platform environment** (`adapters.json`, `iag-services.json`, `om-automations.json` — saved alongside this doc) — i.e. things the workflows assume exist, that a direct check shows are missing, mismatched, or unconfirmed.

1. **`netbox-grid` adapter instance does not exist.** `Get Netbox Device and Context` (task `328c`, `genericAdapterRequest`) hard-codes `adapter_id: "netbox-grid"`. The platform's actual NetBox adapter instances are `NetBox` (type `NetboxV33`, active), `Netbox_March_2026` (type `Netbox`, active), and `Netbox-Maanas` (type `Netbox`, inactive) — none named `netbox-grid`. Per the platform's adapter resolution rules, this would fail at runtime with `"No config found for Adapter: netbox-grid"` unless `netbox-grid` exists in whichever environment the two observed job runs (§G) actually executed against (see §F.1 — likely not this platform).

2. **`Email` adapter instance name doesn't exactly match either configured Email adapter.** `Upgrade Wrapper`'s `mailWithOptions` task (`01ca`) hard-codes `adapter_id: "Email"`. The platform has `email` (lowercase) and `Email-2`, both type `EmailOpensource` — neither is an exact-case match for `"Email"`. Confirm with the engineer whether adapter ID resolution here is case-sensitive; if so, this is a second live mismatch.

3. **The three AGManager shell scripts invoked by `File Transfer` are not registered as IAG services on this platform.** `agnostic_load_image.py`, `agnostic_scp.py` (tasks `9ca1`/`a0d9`), and `scp_cisco.py` (used by the legacy `Run Transfer Scripts`) don't appear among the platform's 20 registered Gateway Manager services (`cfg-b2b`, `eos-snapshot`, `get-config`, `hello-*`, `is-alive`, `run-command`, `send-command`, `send-config`, `set-config`, plus 3 `my-*-service` samples). `File Transfer` is in the live production path — this would block image staging on this platform as configured.

4. **No Operations Manager trigger/automation references project 408 at all.** Checked all 59 platform-wide OM automations for any reference to project 408's `_id` — zero matches. The `Upgrade Form` → `Upgrade Wrapper` connection documented in §D is inferred purely from matching input schemas; there is no confirmed API endpoint, manual trigger, or schedule that actually starts this workflow from the form submission on this platform. (Two OM automations, `IOS Upgrade POC` and `IOS Upgrade POC-Final`, do exist and point at a workflow `"Upgrade POC With Template Creation"` — but that's a different project (`6994dbcc6d90cb331ac67eb3`) and a different, unrecognized workflow name, not any workflow in project 408.)

5. **`Github-TAD` adapter (used by the legacy `Create Blob`/`Create Tree and commit` GitHub-manifest subsystem) is inactive.** Consistent with §F.6's conclusion that this subsystem is dead/legacy — it's not just logically unwired, its adapter dependency is disabled.

6. **The NetBox `upgrade-catalog` plugin's own data model was never independently verified.** Everything documented about it (§C, §D, §H) comes from the *workflow's* `outputSchema`, not from querying NetBox's plugin API/schema directly or confirming which devices/versions actually have a bundle defined. If NetBox access is available, worth confirming the plugin exists and is populated for the target device fleet.

7. **Device inventory not checked.** Whether `I27051` (or other target devices) are registered in Configuration Manager / Inventory Manager with correct credentials and device-type mapping was not verified — `backUpDevice`/`isAlive`/`getDevice` all depend on this.

8. **Project 408 membership/ACL, while resolvable, isn't in solution-design.md's inventory:** owner `ankush.vasishta@itential.com`, editor `maanas.manjunath@itential.com`, group `admins` (editor). Noted here for completeness; add to §A if ACL matters for the delivery record.

**Bottom line:** items 1–4 are concrete, verified blockers *on this specific platform* — even setting aside §F's cross-project wiring questions, `Agnostic Upgrade` as configured in project 408 would fail at the NetBox lookup, the file transfer, and (separately) has no confirmed way to be started, all before ever reaching a device. This reinforces §F.1's read that project 408 is a snapshot/copy: the two successful job runs in §G almost certainly executed against a *different, fully-configured* environment, not this one.

## J. Proposed Extension — HA Pair Support (design only, not yet built)

### Why

Evaluating whether this architecture extends to Palo Alto HA firewall pairs surfaced a structural gap: `Agnostic Upgrade` has no concept of a device *pair* with ordered roles. Palo Alto's documented process (and HA-paired platforms on other vendors generally) requires: detect Active/Passive → suspend Passive → upgrade+verify Passive → suspend Active (failover) → upgrade+verify Active → optionally restore original roles. None of this exists today. This section is the proposed fix, designed to be vendor-agnostic (data-driven via NetBox, like everything else in this project) rather than a Palo-Alto-specific bolt-on.

### Design constraint: `Agnostic Upgrade` is not modified

All new logic lives in a **new wrapper workflow** — tentatively named **`HA Upgrade Wrapper`** — that calls the existing, unmodified `Agnostic Upgrade` as a childJob **twice**, once per pair member. This preserves the proven single-device orchestrator (backup → checks → upgrade → reconnect-loop → post-checks → `ViewDiff` approval → rollback) exactly as-is; `HA Upgrade Wrapper` only adds what's needed to sequence two calls to it safely.

### Proposed new trigger

**`HA Upgrade Form`** (new JSON form, alongside the existing `Upgrade Form`):
- `version` (string, required)
- `bootMode` (string, required)
- `haPair` (array of exactly 2 device identifiers, required) — replaces the single-device `devices` field for this entry point
- `emails` (string, required)
- `restoreOriginalRoles` (boolean, optional, default `false`)

### Proposed NetBox `upgrade-catalog` schema extension

Four new fields on the per-device bundle (alongside the existing `check_command_sets`, `upgrade_sequence`, `rollback_sequence`, `validation_rules`, `images`, `image_transfer`, `diff_ignore_patterns`, `plugin`):

| New field | Purpose | Example (Palo Alto) |
|---|---|---|
| `ha_role_query_command` | Determine Active/Passive | `show high-availability state` |
| `ha_suspend_command` | Take this member out of active HA duty | `request high-availability state suspend` |
| `ha_resume_command` | Restore this member to HA duty (if not automatic on reboot) | `request high-availability state functional` |
| `ha_peer_verify_command` | Confirm the peer is still passing traffic before proceeding | `show session all` / `show interface all` |

This keeps the same principle as every other command set in this project: vendor differences are catalog data, not workflow logic. **Not needed for the current Cisco use case** — the devices upgraded by `Agnostic Upgrade` today are standalone, not HA pairs, so these four fields would simply stay unpopulated for Cisco catalog entries. This extension exists specifically to support HA-paired platforms (Palo Alto firewalls being the immediate driver); it's additive and doesn't change how Cisco devices are handled.

### Proposed `HA Upgrade Wrapper` structure

```
workflow_start
   │
   ▼
1. Resolve HA pair context
   childJob → Get Netbox Device and Context (called once per member, or extended to accept a pair)
   → confirms haPair[0]/haPair[1] are a valid pair, returns each member's bundle incl. the 4 new ha_* commands
   │
   ▼
2. Query role for both members (dynamic command template, reuse Create and Run Command Template)
   → evaluate ha_role_query_command output → set passiveDevice / activeDevice
   │
   ▼
3. Send "upgrade started" notification (NEW — today's Upgrade Wrapper only notifies at the end)
   │
   ▼
4. LEG 1 — Passive member
   a. Run ha_suspend_command against passiveDevice (dynamic command template)
   b. Verify suspended (re-run ha_role_query_command, evaluate)
   c. Verify activeDevice traffic unaffected (ha_peer_verify_command)
   d. childJob → Agnostic Upgrade (UNMODIFIED) — devices=passiveDevice, version, bootMode
   e. Evaluate Agnostic Upgrade's `success` output:
        failure ──► ABORT: do not proceed to Leg 2. Best-effort resume passiveDevice
                    if only suspended (not yet mid-upgrade). Send failure notification. End.
        success ──► continue
   │
   ▼
5. Verify passiveDevice healthy on new version; run ha_resume_command if role doesn't
   auto-clear on reboot (vendor-dependent — encode via ha_resume_command being empty/no-op
   in the catalog for platforms where it's automatic)
   │
   ▼
6. LEG 2 — Active member (now safe to touch — Passive is healthy on new code)
   a. Run ha_suspend_command against activeDevice → forces failover to the upgraded Passive
   b. Verify traffic shifted to passiveDevice
   c. childJob → Agnostic Upgrade (UNMODIFIED) — devices=activeDevice, version, bootMode
   d. Evaluate `success`:
        failure ──► notify; flag for manual intervention (HA pair is now in a mixed/uncertain
                    state — do not attempt further automated changes)
        success ──► continue
   │
   ▼
7. Optional: if restoreOriginalRoles — suspend/resume to swap roles back to original assignment
   │
   ▼
8. Final combined verification (both members same version, HA re-synced)
   │
   ▼
9. Consolidated final notification — ONE email summarizing both legs' outcomes
   │
   ▼
workflow_end
```

**Why call `Agnostic Upgrade` directly, not `Upgrade Wrapper`:** `Upgrade Wrapper` sends its own per-call email. Calling it twice would produce two separate, uncoordinated notifications for what is operationally one change. `HA Upgrade Wrapper` bypasses `Upgrade Wrapper` and calls `Agnostic Upgrade` directly, taking over the notification responsibility itself (step 3 and step 9) so the engineer gets one coherent start/end pair of emails for the whole HA operation.

### What this reuses vs. adds

| | Reused as-is | New |
|---|---|---|
| Per-device upgrade (backup, checks, upgrade, reconnect, post-checks, rollback, approval) | ✅ `Agnostic Upgrade`, unmodified | — |
| Dynamic command templating mechanism | ✅ `DynamicTemplateCreation` / `Create and Run Command Template` | New command *content* only (role-query/suspend/resume, as catalog data) |
| NetBox-catalog-driven, vendor-neutral command resolution | ✅ same pattern | 4 new schema fields |
| Notification mechanism | ✅ `mailWithOptions` / `Parse Emails` | New call sites (start-of-run, consolidated end) |
| HA role detection, ordering, abort gate | — | ✅ New — the core addition |
| `HA Upgrade Wrapper` workflow itself | — | ✅ New |
| `HA Upgrade Form` | — | ✅ New |

### Status

Design only. Not implemented, and not testable in this sandbox — no HA-capable adapters or devices are configured here (see §I). Before building: confirm the `ha_resume_command` behavior assumption (automatic-on-reboot vs. requires-explicit-command) per target vendor, and decide whether `Get Netbox Device and Context` should be extended to accept a pair directly or continue being called once per member from the wrapper.
