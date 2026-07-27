# Solution Design: Palo Alto HA Firewall PAN-OS Upgrade

> **Pre-build design — no live Feasibility was performed.** No target platform was available for this delivery. Per the engineer's direction, this design proceeds on the same sandbox platform used for `AgnosticUpgrade` (`https://ws-solution-design-poc-iap01.trial.itential.io`), which has **no** Panorama, NMS, or BMC Helix-write adapters configured. Every integration point that depends on those is built as an explicit, clearly-labeled placeholder/stub — structurally correct and ready to rewire, but not functional until a real platform with those adapters is identified. Treat this document as provisional until a real Feasibility pass happens against an actual target platform.

## A. What's reused vs. new

| | Source | Status |
|---|---|---|
| Per-device upgrade mechanics (backup, dynamic MOP templating, reconnect-retry, `ViewDiff` approval, rollback) | `Agnostic Upgrade` workflow, project `_id 6a60d7316556606de51220d1` (iid 408) | **Reused unmodified** via cross-project childJob |
| Dynamic command-template mechanism (`DynamicTemplateCreation` → `MOP.importTemplate` → `RunCommandTemplate` → `Clean Up Template`) | Same project | **Reused unmodified**, called with new HA-operation command content |
| NetBox-catalog-driven per-device context resolution | Same pattern as `Get Netbox Device and Context` | **Reused pattern**, new workflow (`Get HA Pair Context`) since the query needs 2 devices + HA fields |
| Email notification | `EmailOpensource` adapter, instance `email` | Reused |
| HA role detection, ordered two-leg orchestration, abort gate | — | **New** — this use case's actual contribution |
| NMS maintenance-mode suppression | — | **New, stubbed** — no adapter available in this sandbox |
| CRQ close/update | — | **New, stubbed** — only fetch exists (`Fetch CRQ Details`, unwired, in a different project) |
| Panorama integration | — | **Not built** — explicitly out of scope per the Panorama-bypass decision in `customer-spec.md` §4 |

## B. Critical cross-project reference — learned from AgnosticUpgrade's earlier mistake

`AgnosticUpgrade/solution-design.md` §F documents a defect where childJob tasks carried a stale project prefix. To avoid repeating it: every childJob call from this new project to `Agnostic Upgrade` **must** use the exact current project-scoped name, confirmed live from the platform immediately before building:

```
@6a60d7316556606de51220d1: Agnostic Upgrade
```

Do not hard-code any other project `_id` prefix for this reference.

## C. New Project

**Name:** `Palo Alto HA Upgrade`
Build via `POST /automation-studio/projects/import` (atomic — per Rule 11), then immediately PATCH membership to add the engineer's account as owner (Rule 11a — the OAuth service account will otherwise be sole owner).

## D. Component Inventory (built — project `_id` `32f7a975af37635566047d48`, iid 1, "Palo Alto HA Upgrade")

| # | Component | Type | UUID / ref | Purpose |
|---|---|---|---|---|
| 1 | HA Upgrade Wrapper | workflow | `0cdd078f-3358-42cd-bbb2-56cc8c128afd` | Entry point — orchestrates the two-leg HA upgrade (52 tasks) |
| 2 | Get HA Pair Context | workflow | `460d0e9e-d7f8-45b2-aafe-6e9cad9ab4d5` | Resolves both members' bundles + HA fields from NetBox |
| 3 | Suspend HA Member | workflow | `3308f99a-fb48-4182-9fec-7ad23d58c0e1` | Runs `ha_suspend_command` via dynamic MOP template |
| 4 | Verify Peer Traffic | workflow | `ed92d925-84c4-47b3-ae44-b8870419474b` | Runs `ha_peer_verify_command`, evaluates unaffected |
| 5 | Resume HA Member | workflow | `9ce0d5bd-db8c-4c26-b767-d870d7299a6f` | Runs `ha_resume_command` (no-op branch if command is empty) |
| 6 | NMS Maintenance Mode (STUB) | workflow | `ff0f17aa-ddb1-46c6-83b9-fe468a173cd2` | Placeholder — `stub` task only, clearly labeled, no real adapter call |
| 7 | Close CRQ (STUB) | workflow | `5d5b32d1-771e-430d-b7e7-877587acf811` | Placeholder — `stub` task only, clearly labeled, no real adapter call |
| 8 | HA Upgrade Form | jsonForm | `6a6767645620cecbbbb68185` | Trigger UI — `version`, `haPair` (array, exactly 2), `bootMode`, `emails`, `restoreOriginalRoles`, `crqId` |

**Build status: all 8 components deployed and component-tested.** See §I for test evidence and two real structural bugs found and fixed during testing.

Components 3–5 reuse the same "dynamic command template" shape as `Agnostic Upgrade`'s `Create and Run Command Template`, so they are thin wrappers rather than new mechanism — each takes a device + a command string from the NetBox bundle and runs it via the existing `DynamicTemplateCreation`/`RunCommandTemplate`/`Clean Up Template` chain in the `Agnostic Upgrade` project (cross-project childJob, same prefix rule as §B).

## E. NetBox Schema Expectation (data contract, not a live schema change)

This build cannot modify the real NetBox `upgrade-catalog` plugin (no write access to a real NetBox instance in this sandbox). `Get HA Pair Context`'s task/output schema will **declare** the expected shape so the contract is documented and ready once real NetBox data exists:

```json
{
  "ha_role_query_command": "string",
  "ha_suspend_command": "string",
  "ha_resume_command": "string",
  "ha_peer_verify_command": "string"
}
```
(alongside the existing `check_command_sets`, `upgrade_sequence`, `rollback_sequence`, `validation_rules`, `images`, `image_transfer`, `diff_ignore_patterns`, `plugin` fields already used by `Agnostic Upgrade`)

`Get HA Pair Context`'s adapter task will call the same `genericAdapterRequest` shape against `plugins/upgrade-catalog/upgrade-bundles`, using a **real** adapter instance name from this platform (`NetBox` or `Netbox_March_2026` — not the nonexistent `netbox-grid` that `Agnostic Upgrade` itself incorrectly references, per its own §I finding). This will return 404/empty against the real NetBox plugin until PAN-OS catalog entries exist there — expected and acceptable for a structural build.

## F. HA Upgrade Wrapper — Task Flow

```
workflow_start
   │
   ▼
[newVariable] initialize job vars
   │
   ▼
[childJob] Get HA Pair Context (haPair[0], haPair[1], version)
   │
   ▼
[childJob] Suspend HA Member — role-query only, no suspend yet (query mode)
   → OR: separate lightweight command-template call per device for ha_role_query_command
   │
   ▼
[evaluation] Determine passiveDevice / activeDevice
   │
   ▼
[stub] NMS Maintenance Mode (STUB — both members)         ← placeholder, labeled
   │
   ▼
[childJob] mailWithOptions — "upgrade started" notification
   │
   ══════════════ LEG 1 — PASSIVE ══════════════
   ▼
[childJob] Suspend HA Member (passiveDevice)
   │
   ▼
[evaluation] Verify suspended
   │  failure → [newVariable] emailMessage="Failed to suspend passive member" → notify → workflow_end
   ▼
[childJob] Verify Peer Traffic (activeDevice)
   │
   ▼
[childJob] Agnostic Upgrade  (@6a60d7316556606de51220d1: Agnostic Upgrade — UNMODIFIED)
             devices=passiveDevice, version=$var.job.version, bootMode=$var.job.bootMode
   │
   ▼
[evaluation] Leg 1 success?
   │  failure → [newVariable] emailMessage="Passive leg failed — aborting before Active member"
   │            → notify → workflow_end   (ABORT GATE — never reaches Leg 2)
   ▼ success
[childJob] Resume HA Member (passiveDevice) — no-op per catalog if automatic
   │
   ══════════════ LEG 2 — ACTIVE (only reached if Leg 1 succeeded) ══════════════
   ▼
[childJob] Suspend HA Member (activeDevice)   ← triggers failover
   │
   ▼
[evaluation] Verify traffic shifted to passiveDevice
   │
   ▼
[childJob] Agnostic Upgrade  (UNMODIFIED)
             devices=activeDevice, version=$var.job.version, bootMode=$var.job.bootMode
   │
   ▼
[evaluation] Leg 2 success?
   │  failure → [newVariable] emailMessage="Active leg failed — HA pair in mixed state, manual intervention required"
   │            → notify → workflow_end
   ▼ success
[evaluation] restoreOriginalRoles == true?
   │  true → [childJob] Suspend HA Member (activeDevice, now-upgraded) → [childJob] Resume HA Member (original passive)
   ▼
[stub] Close CRQ (STUB)                                    ← placeholder, labeled
   │
   ▼
[newVariable] emailMessage="Successfully completed HA upgrade" → [childJob] mailWithOptions (consolidated)
   │
   ▼
workflow_end
```

**Error transitions:** per Rule 19, every adapter/childJob task above needs an explicit `error` transition, not just `success`/`failure` evaluation branches — routed to the same failure-notification pattern `Agnostic Upgrade` already uses (a dedicated `newVariable` task setting a descriptive `emailMessage`, then to the notification task, then `workflow_end`).

**Task ID convention:** per Rule 9, all task IDs must be hex-only (`[0-9a-f]{1,4}`) — generate accordingly during build, not descriptive names like `suspendPassive`.

## G. Stub Task Specification

Both `NMS Maintenance Mode` and `Close CRQ` are built as `automatic`/`stub` type tasks (the same `WorkFlowEngine.stub` task type already used inside `Agnostic Upgrade` for internal routing), with:
- `summary`: `"PLACEHOLDER — requires [Spectrum NMS adapter | BMC Helix write operation] not available in this environment"`
- No adapter call — purely structural, so the workflow is complete and importable, and the exact insertion point in the flow is unambiguous once a real adapter is wired in.

## H. Open Items Before This Can Run For Real

1. Real target platform with Panorama-bypass-compatible PAN-OS device access (device-direct CLI/API, not through Panorama) — unconfirmed assumption from `customer-spec.md` §4.
2. Real NMS adapter for maintenance-mode suppression.
3. Real BMC Helix write/close-CRQ capability.
4. Real NetBox `upgrade-catalog` entries for the target PAN-OS models, including the 4 new HA fields (§E).
5. Confirm `ha_resume_command` behavior (automatic-on-reboot vs. explicit) for the actual PAN-OS version in use.

## I. Build & Component Test Evidence

All 8 components were imported atomically (`POST /automation-studio/projects/import`), pre-flight validated (`POST /automation-studio/workflows/validate` — zero errors/warnings on every workflow before and after fixes), and membership was patched immediately after import (owner/editor/group matching `Agnostic Upgrade`'s project, per Rule 11a).

**Two real structural bugs were found and fixed during component testing** (both now corrected in the deployed workflow):

1. **Missing error transitions on terminal `mailWithOptions` tasks.** The four terminal notification tasks (`mail_err`, `mail_abort1`, `mail_abort2`, `mail_done`) only had success transitions to `workflow_end`. When a real send failure occurred (see below), the job halted with `"Job has no available transitions"`. Fixed by adding a shared `terminal_err_sink` task (a `newVariable` recording that the final notification failed) that each of the four routes to on error, which then reaches `workflow_end`.
2. **Missing failure transitions on `query` tasks with `pass_on_null: false`.** Every `query` task in `HA Upgrade Wrapper` that reads a field out of NetBox-bundle or childJob-output data only had a success transition. Once real (empty) NetBox data flowed through — see test evidence below — these queries hit their `failure` state with no transition defined, and the job hung with the same "no available transitions" symptom, this time pointing at a different, generic-looking task list. Fixed by adding a `failure` transition on every such query, routed to the shared `err_handler` chain.

**Component test results:**

| Component | Test | Result |
|---|---|---|
| NMS Maintenance Mode (STUB) | Started with `device1`/`device2` | `complete`, `success: true` — stub behaves as designed |
| Close CRQ (STUB) | Started with `crqId` | `complete`, `success: true` — stub behaves as designed |
| Get HA Pair Context | Started with `device1`/`device2`/`version` | `complete` (via its own error-handling branch) — the NetBox adapter call genuinely timed out (`ETIMEDOUT`, "The Adapter has run out of time for the request") against the real `NetBox` adapter instance, which is active but not backed by a real `upgrade-catalog` plugin instance reachable from here. Confirms the adapter reference is correct (unlike `Agnostic Upgrade`'s own broken `netbox-grid` reference) and the error-handling path works. |
| Resume HA Member (no-op path) | Started with empty `command` | `complete`, `success: true` — the skip-evaluation branch correctly bypasses the command-template mechanism |
| Suspend HA Member | Started with a real device/command | Reached `running` and correctly resolved the cross-project childJob to `@6a60d7316556606de51220d1: Create and Run Command Template` — confirmed via the job's `childJobs` field. Left running/canceled rather than awaited to completion — the underlying MOP command-template chain depends on an adapter (`Command Template MOP`) that `Agnostic Upgrade`'s own `Clean Up Template` workflow already documents as *"not a configured adapter within the platform"* in this sandbox, so full completion isn't achievable here regardless of this build's correctness. |
| Verify Peer Traffic | Same test as above | Same result — correct cross-project resolution confirmed, same downstream MOP-adapter limitation |
| **HA Upgrade Wrapper (full run)** | `version=11.1.4-h18`, `haPair=[fw01,fw02]`, `bootMode=Default`, `emails=test@example.com` | After both fixes: **`complete`**, `success: "failed"`, `emailMessage: "HA upgrade finished but the final notification email failed to send"`. The run genuinely exercised: haPair split → context resolution (NetBox timeout, handled) → graceful failure routing through every downstream query → the shared error handler → a **real SMTP rejection** on the final notification (`501 5.1.5 Recipient address reserved by RFC 2606`, because `test@example.com` is a reserved test domain) → graceful completion via the newly-added error sink. This is strong evidence the `email` adapter is genuinely live (it made a real SMTP connection and got a real protocol-level rejection), and that the workflow's error handling is now robust end-to-end even under total backend absence. |

**What this proves and doesn't prove:** every workflow is structurally sound (validates cleanly, starts, routes correctly on both success and failure paths, resolves all cross-project references correctly including the one this repo previously got wrong). It does **not** prove the HA upgrade logic is functionally correct against a real PAN-OS HA pair — that requires the four items in §H, none of which exist in this sandbox. Re-run the "HA Upgrade Wrapper (full run)" test with a real email address and populated NetBox catalog data once available, and expect it to progress further into Leg 1 before hitting the next real gap (most likely the MOP adapter, per the Suspend/Verify test results above).
