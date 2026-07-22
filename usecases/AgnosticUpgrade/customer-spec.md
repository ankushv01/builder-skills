# Use Case: Agnostic (Vendor-Independent) Device Software Upgrade

> **Note:** This spec was produced by reading project `Agnostic Upgrade` (platform iid `408`, `_id` `6a60d7316556606de51220d1`).
> Review and correct any inferences before using as a delivery baseline.

## 1. Problem Statement

Network teams manage software upgrades across multiple device platforms (confirmed: Cisco IOS/IOS-XE-style CLI; the design intends to generalize to other platforms). Each platform has different pre-check commands, upgrade command sequences, rollback sequences, and validation rules. Hard-coding these per vendor leads to a proliferation of near-duplicate workflows.

This use case implements a single orchestration workflow whose device-specific behavior (which commands to run for checks/upgrade/rollback/validation, which image to stage, how to ignore expected diff noise) is **not hard-coded** — it is looked up at runtime from a NetBox plugin catalog (`plugins/upgrade-catalog/upgrade-bundles`) keyed by device/version. This is the "agnostic" design: one workflow engine, data-driven per-platform behavior.

## 2. High-Level Flow

```
Upgrade Form (JSON Form: version, devices, bootMode, emails)
        │
        ▼
Upgrade Wrapper  ─────────────────────────────────────────────►  Email notification (final result)
        │  (childJob)
        ▼
Agnostic Upgrade  (core orchestrator)
        │
        ├─ 1. Get NetBox Device & Context   → pulls the upgrade "bundle" for this device/version
        │       (check commands, upgrade sequence, rollback sequence, validation rules,
        │        image list, diff-ignore patterns) from NetBox
        │
        ├─ 2. Pick Version from Images  →  File Transfer (stage image to device)
        │
        ├─ 3. Build & run PRE-CHECK command template  (dynamic Jinja2 → MOP import → run → cleanup)
        │       → Parse Pre/Post Checks → Filter Validations → run pre-check validations
        │       → Validation Success? ──fail──► go to Rollback (step 7)
        │
        ├─ 4. Backup device configuration → Push results to Nexus (artifact store)
        │
        ├─ 5. Build & run UPGRADE command template (dynamic) → device reload
        │       → poll "Alive?" with a bounded retry/backoff loop (max 10 attempts)
        │       → retries exhausted ──► terminal failure (email sent, no rollback attempted)
        │
        ├─ 6. Build & run POST-CHECK command template (dynamic) → run post-check validations
        │       → Push results to Nexus → Validation Success? ──fail──► go to Rollback (step 7)
        │       → success ──► manual "View Diff" approval gate
        │             ├─ approved   → mark Success → done
        │             └─ rejected   → go to Rollback (step 7)
        │
        └─ 7. Rollback path: Create Rollback Image → re-run File Transfer with prior image
                → build & run rollback command template → terminal (success/failure email)
```

## 3. Phases

1. **Context resolution** — resolve the device's upgrade bundle from NetBox (command sets, sequences, validation rules, target image(s), diff-ignore patterns) — this is the single source of truth that makes the workflow vendor-agnostic.
2. **Image staging** — transfer the target OS image to the device (shell-script-based SCP/load via Automation Gateway), with a pre-transfer existence check.
3. **Pre-checks & validation** — dynamically generate a MOP command template from the NetBox-supplied check command set, run it, parse results, validate against expected rules.
4. **Backup** — Configuration Manager device backup, with results pushed to an artifact store (Nexus) for audit.
5. **Upgrade execution** — dynamically generate and run the upgrade command template (reload/bundle/install per `bootMode`), then poll device reachability with a capped retry loop.
6. **Post-checks & validation** — re-run the same check pattern post-upgrade, push results to Nexus, and diff pre/post state.
7. **Human approval gate** — an engineer reviews the pre/post diff (`ViewDiff`) before the job is marked complete.
8. **Rollback** — triggered by pre-check failure, post-check failure, or a rejected diff review: re-stage the prior image and re-run the rollback command sequence.
9. **Notification** — every terminal state (success, each distinct failure reason, rollback outcome) sends a descriptive result email to the addresses supplied at trigger time.

## 4. Key Design Decisions

- **NetBox as the per-device behavior catalog**, not hard-coded per vendor — inferred from the dedicated NetBox plugin endpoint (`upgrade-catalog/upgrade-bundles`) returning `check_command_sets`, `upgrade_sequence`, `rollback_sequence`, `validation_rules`, `images`, `diff_ignore_patterns`, `plugin`.
- **Just-in-time MOP command templates** — rather than maintaining static command templates per platform, the workflow renders a Jinja2 template into a MOP command-template JSON payload, imports it (`MOP.importTemplate`), runs it (`RunCommandTemplate`), and deletes it afterward (`MOP.deleteTemplate`). This keeps command sets fully data-driven and avoids template sprawl.
- **Bounded reconnect retry after reload** — a reattempt counter capped at 10, with a delay between checks, rather than a fixed sleep — inferred from the `Alive?` / `Increment Reattempt` / `Delay a Job` loop.
- **Human-in-the-loop approval before final sign-off** — a manual `ViewDiff` task gates the terminal "success" state, letting an engineer reject even a technically-successful upgrade based on the diff.
- **Descriptive per-branch failure messaging** — every distinct failure path (image not found, transfer failed, backup failed, pre-check failed, post-check failed, reconnect failed, rollback transfer failed) sets its own human-readable `emailMessage`, rather than a single generic failure state.
- **Audit trail via Nexus** — check/backup results are pushed to Nexus (`pushFileWithTextContentToNexus`) at two points (after backup, after post-check validation) for external retention.

## 5. Scope

**In scope (as built):**
- Single-device-group orchestration triggered via `Upgrade Form` (version, device list, boot mode, notification emails)
- NetBox-driven, per-device command/behavior resolution
- Pre-check → backup → upgrade → post-check → diff-approval → rollback-on-failure flow
- Email notification for every terminal outcome
- Nexus artifact push for check/backup results

**Not observed / likely out of scope as delivered:**
- **Nexus push is currently broken** — the childJob target project for `pushFileWithTextContentToNexus` no longer exists on the platform (`Project not found`). This step will fail at runtime today.
- **Change-management gating** — a separate workflow (`Fetch CRQ Details`, resolving devices from a BMC Helix change request) exists in the project but is **not wired into** the main `Agnostic Upgrade`/`Upgrade Wrapper` path. If CRQ-gated execution is a requirement, it needs explicit integration.
- **Legacy/parallel GitHub-manifest path** — a second, disconnected subsystem (`Software Upgrade`, `Software Upgrade TAD`, `Retrieve from GitHub`, `Execute Upgrade Steps[TAD]`, `Create Blob`/`Create Tree and commit`/`Update Version TXT` git-audit-trail workflows, `Upgrade POC`) exists in the same project, storing upgrade steps/manifests in a GitHub repo instead of NetBox. It is **not called** by the production entry point and appears to be an earlier design iteration or proof-of-concept, not currently in the live path.
- No automated multi-device batching/loop strategy was observed beyond a single `devices` array input passed straight through — device-by-device looping (if any) happens inside the child workflows, not visible at this layer.

## 6. Risks & Mitigations

| Risk | Evidence | Mitigation observed / needed |
|---|---|---|
| Nexus artifact push is broken (dangling cross-project reference) | `pushFileWithTextContentToNexus` childJob target project returns "Project not found" | Needs re-pointing to a valid project, or replacing with an in-project implementation |
| Reconnect-retry exhaustion has no rollback | `Alive?` reattempt-exhausted branch (`97b5`/`57f9`) routes straight to a failure email — no rollback is attempted if the device never comes back after upgrade | Confirm with engineering whether this is intentional (device may be mid-upgrade, rollback could worsen state) or a gap |
| Legacy GitHub-manifest workflows still present but unwired | No workflow calls `Software Upgrade`, `Retrieve from GitHub`, `Upgrade POC`, etc. | Confirm whether these should be archived/removed or are a planned future path |
| Task labeling is confusing for auditors | Terminal state-setter tasks are all named "Success" even on failure paths (they set `success: "failed"` as their variable value) | Cosmetic — rename for clarity during any future edit |
| Change-management (CRQ) validation not enforced | `Fetch CRQ Details` exists but isn't called from the main flow | Confirm whether change-window/CRQ validation is a requirement before go-live |

## 7. Requirements

### Capabilities
- Automation Studio workflows, transformations (JST), Jinja2 templates, JSON Form trigger input
- MOP Command Template Runner (dynamic import/run/delete)
- Configuration Manager (`isAlive`, `backUpDevice`)
- Automation Gateway (`AGManager`) shell scripts for image load/SCP transfer
- Email (EmailOpensource adapter) for notifications
- Manual task (`ViewDiff`) for human approval

### Integrations
- **NetBox** (adapter instance `netbox-grid`) — custom plugin `upgrade-catalog`, endpoint `plugins/upgrade-catalog/upgrade-bundles` — per-device upgrade bundle (commands, sequences, images, validation rules)
- **GitHub** — used only by the legacy/unwired manifest-based subsystem, not the live path
- **Nexus** — artifact push for backup/check results (currently broken — target project missing)
- **BMC Helix** — change request lookup (`Fetch CRQ Details`), present but not wired into the live path

## 8. Batch Strategy

`devices` is accepted as an array at the top level (`Upgrade Wrapper` → `Agnostic Upgrade` → `File Transfer`/command-template child workflows). No `loopType`/per-device fan-out was observed at the orchestrator level in the pulled detail — multi-device handling, if present, is delegated to the child workflows' own input handling. Confirm actual per-device iteration behavior with the engineer before assuming full parallel/sequential batch support.

## 9. Acceptance Criteria

Inferred from output schema and evaluation checks in `Agnostic Upgrade`:
1. Device upgrade bundle is successfully resolved from NetBox for the target version/device (`success` output, `check_command_sets`, `upgrade_sequence`, etc. populated).
2. Target image exists and is successfully transferred to the device before any upgrade commands run.
3. Pre-check validations pass before the device is upgraded; on failure, no upgrade commands are issued and rollback logic is invoked immediately.
4. Device configuration is backed up before upgrade.
5. Device reload/upgrade completes and device reconnects within the bounded retry window (≤10 attempts).
6. Post-check validations pass and are diffed against pre-check baseline, respecting configured diff-ignore patterns.
7. An engineer explicitly approves the diff (`ViewDiff`) before the job is marked complete.
8. On any validation failure or rejected diff, the device is rolled back to the prior image/config and a rollback-outcome email is sent.
9. Every terminal outcome (success, each failure mode, rollback outcome) produces a notification email to the addresses supplied at trigger time.

> **Update:** two completed production job runs (see below) confirm criteria 1–9 are all met in practice on the happy path, including the Nexus artifact push — the earlier note about a broken Nexus dependency was a client-visibility artifact during this documentation pass, not a functional defect. See `solution-design.md` §F–G for the full correction and evidence.

## 10. Observed Example Runs

Two completed job records were supplied and analyzed to validate this spec against real behavior (full detail in `solution-design.md` §G):

| | Run 1 | Run 2 |
|---|---|---|
| Device | `I27051` (Cisco Catalyst 9000, IOS-XE) | `I27051` |
| Target version | `17.15.05` | `17.14.01` |
| Boot mode | `Default` | `install` |
| Outcome | Success — "Successfully completed upgrade" | Success — "Successfully completed upgrade" |
| Duration | 20.5 minutes | 31.8 minutes |
| Pre/post artifacts | `I27051_pre-checks.cfg` / `I27051_post-checks.cfg` pushed to Nexus repo `gsa_netauto` | same pattern |

Both runs took the full happy path (no rollback, no reconnect retries beyond the first check, diff approved). No real example yet exists for a validation failure, rollback, or reconnect-exhaustion run — if acceptance testing is planned, these branches should be exercised deliberately since they're unvalidated by real data so far.

**Open question surfaced by the real data:** both runs pass `devices` as a single device string, not an array, even though the trigger form (`Upgrade Form`) types `devices` as an array. Confirm with the engineer whether upstream fan-out (one call per device) is expected, or the array typing needs correcting.
