# Use Case: Palo Alto HA Firewall PAN-OS Upgrade

> **Note:** This spec is derived from a real manual runbook (`Palo Alto OS upgrade step by step process.docx`, ref. `CRQ00000515633 — Upgrade GSIL Firewalls to PAN-OS 11.1.4-h18`) and extends the architecture already built and proven for Cisco devices in the `Agnostic Upgrade` project (`usecases/AgnosticUpgrade/`). Review and correct before treating as an approved baseline.

## 1. Problem Statement

Palo Alto firewalls in this environment are deployed as Active/Passive High Availability (HA) pairs. Upgrading PAN-OS today is a fully manual, GUI-driven runbook: log into Panorama and each device individually, manually determine HA roles, suspend and upgrade one pair member at a time in a specific order, verify at each step, then fail over and repeat on the second member — all while coordinating NMS alert suppression, compliance evidence capture, and a change-ticket (CRQ) closure.

This is the same category of problem `Agnostic Upgrade` already solved for standalone Cisco devices: replace a manual, error-prone, per-vendor runbook with a single data-driven orchestration engine. The core difference driving a new use case (rather than just a new NetBox catalog entry) is that Palo Alto devices are **paired**, not standalone — the existing single-device orchestrator has no concept of HA ordering, and this gap was identified and design-resolved in `usecases/AgnosticUpgrade/solution-design.md` §J (the `HA Upgrade Wrapper` pattern) before this use case was scoped.

## 2. High-Level Flow

```
HA Upgrade Form (version, haPair: [device1, device2], bootMode, emails, restoreOriginalRoles)
        │
        ▼
HA Upgrade Wrapper  (new — see AgnosticUpgrade/solution-design.md §J for full design)
        │
        ├─ 1. Resolve HA pair context (NetBox bundle per member, incl. HA role-query/
        │       suspend/resume commands — extends the existing upgrade-catalog schema)
        │
        ├─ 2. Detect current Active/Passive roles
        │
        ├─ 3. Notify start of change (email/Teams) + suppress NMS alerting (both members)
        │
        ├─ 4. LEG 1 — Passive member:
        │       suspend → verify peer traffic unaffected → backup (config + device-state)
        │       → childJob: Agnostic Upgrade (UNMODIFIED, reused as-is) → evaluate success
        │       ── failure ──► ABORT, do not touch Active member, notify, end
        │
        ├─ 5. LEG 2 — Active member (only reached if Leg 1 succeeded):
        │       suspend (forces failover to upgraded Passive) → verify traffic shifted
        │       → backup → childJob: Agnostic Upgrade (UNMODIFIED) → evaluate success
        │       ── failure ──► notify, flag for manual intervention, end
        │
        ├─ 6. Optional: restore original Active/Passive role assignment
        │
        ├─ 7. Final combined verification (both members on target version, HA re-synced)
        │
        └─ 8. Close CRQ (BMC Helix) + consolidated final notification
```

## 3. Phases

1. **Context resolution** — resolve each HA member's upgrade bundle from NetBox, extended with HA role-query/suspend/resume/peer-verify commands (see §4). Confirms the two devices are a valid pair.
2. **Change-window prep** — suppress NMS alerting for both members (NMS integration — vendor/tool TBD at Feasibility), notify stakeholders of outage start.
3. **HA role detection** — determine which member is currently Active vs Passive; this ordering decision drives everything downstream.
4. **Passive-member upgrade** — suspend, verify peer unaffected, back up, then hand off to the existing, unmodified `Agnostic Upgrade` orchestrator for the actual backup/checks/upgrade/reconnect/post-checks/approval/rollback sequence.
5. **Failover gate** — do not proceed to the Active member unless the Passive member's upgrade fully succeeded.
6. **Active-member upgrade** — suspend (triggering failover), verify traffic shifted, then the same `Agnostic Upgrade` hand-off.
7. **Role restoration (optional)** — if operationally required, swap back to the original Active/Passive assignment.
8. **Close-out** — final verification, CRQ closure, consolidated notification.

## 4. Key Design Decisions

- **Reuse `Agnostic Upgrade` unmodified.** All per-device upgrade mechanics (backup, dynamic MOP command templating, reconnect-retry loop, post-checks, `ViewDiff` approval, rollback-on-failure) are proven and already built. This use case adds an HA-pair-aware wrapper *above* it rather than duplicating or modifying that logic. Full technical design already exists in `usecases/AgnosticUpgrade/solution-design.md` §J.
- **Image sourcing is simplified, matching the Cisco pattern:** the target `version` is a user-supplied input, and the image's location is a known, pre-registered entry in the same NetBox `upgrade-catalog`/Nexus mechanism already used for Cisco. This **removes any dependency on Panorama for image discovery/download/validation** — the manual runbook's "Check Now / Download / Validate" dance against Panorama is out of scope; the automation goes straight from a known catalog entry to device, the same way it already does for Cisco.
- **HA-pair logic lives in a new `HA Upgrade Wrapper`, not in `Agnostic Upgrade`.** Confirmed with the engineer: (1) if the Passive leg fails or rolls back, the wrapper aborts before ever touching the Active member — never touch the device carrying live traffic if its peer isn't confirmed healthy; (2) `ViewDiff` approval stays inside `Agnostic Upgrade` and is approved once per leg, not combined into a single cross-member approval.
- **HA-specific behavior (role query, suspend, resume, peer-verify) is expressed as NetBox catalog data**, not vendor-specific workflow code — consistent with the "agnostic" architecture principle already established for Cisco. This is **additive only** — Cisco catalog entries simply leave these fields unpopulated; nothing about the existing Cisco flow changes.
- **Device-direct, not Panorama-mediated, for HA operations.** Following the same philosophy as the existing Cisco design (talk to the device's own CLI/API, not through a centralized manager), HA role-query/suspend/resume commands are proposed to run directly against each firewall rather than through Panorama — pending Feasibility confirmation that this is viable in the target environment.

## 5. Scope

**In scope:**
- HA-pair-aware orchestration for exactly 2 devices per run (Active/Passive PAN-OS firewall pair)
- Ordered suspend → upgrade → verify → failover → repeat, with an abort gate between legs
- Reuse of `Agnostic Upgrade`'s existing per-device upgrade mechanics unchanged
- Config + device-state backup per member, pushed to the same artifact store used today (Nexus)
- Start-of-change and consolidated end-of-change notifications
- Optional role restoration after both legs complete
- CRQ closure (extends the existing, currently-unwired `Fetch CRQ Details` BMC Helix integration with a new close/update operation)

**Explicitly out of scope for this spec (deferred to Feasibility/Design):**
- **Panorama integration** — not needed under the image-sourcing simplification in §4; if Feasibility finds a reason PAN-OS install/HA operations must go through Panorama rather than device-direct, this spec's design principle will need revisiting.
- **NMS maintenance-mode suppression (Spectrum or equivalent)** — the manual process requires this; no NMS adapter exists in the current sandbox environment, and the target production platform's actual NMS tooling/adapter availability is unconfirmed. Captured as a requirement (§7) but not designed in detail here.
- **Compliance evidence capture (CIP-10-style checksum)** — the manual process captures and archives a formal integrity checksum as regulatory evidence; today's Cisco pre/post-checks are generic `show`-command validation, not a formal compliance artifact. Needs its own design pass.
- **Model-specific post-upgrade commands** (e.g., the manual runbook's PA-850-specific debug command) — handled generically via the same "per-device NetBox catalog entry" mechanism already proven for Cisco; specific PAN-OS model entries are a Design-stage/Build-stage task, not a spec-level concern.

## 6. Risks & Mitigations

| Risk | Source | Mitigation |
|---|---|---|
| Touching the Active member while its peer isn't healthy could cause a full outage | Manual process treats ordering as safety-critical | Hard abort gate before Leg 2 if Leg 1 fails (§4, engineer-confirmed) |
| HA role/suspend/resume command syntax and behavior (auto-clear-on-reboot vs. requires-explicit-resume) is vendor/model dependent and unverified for PAN-OS | Not yet tested against real PAN-OS devices | Confirm exact op-command syntax and resume behavior during Feasibility, before Build |
| No NMS, Panorama, or Helix-write adapter currently exists in the available sandbox | Confirmed this session — sandbox has zero relevant adapters | Feasibility stage must confirm adapter availability on the actual target platform before Build proceeds |
| Compliance checksum capture is a regulatory requirement (CIP-10) with no current automation equivalent | Manual process treats this as mandatory, pre-outage | Needs explicit design — do not assume the existing generic pre/post-check mechanism satisfies a compliance evidence requirement without confirming with the compliance/audit stakeholder |
| `Fetch CRQ Details` only fetches — there is no existing "close CRQ" operation to reuse | Confirmed in `AgnosticUpgrade` solution-design.md §F.5 | New BMC Helix write operation needs to be built; not a reuse |

## 7. Requirements

### Capabilities
- Everything `Agnostic Upgrade` already provides (dynamic MOP command templating, reconnect-retry, `ViewDiff` approval, rollback, Nexus artifact push, email/Teams notification)
- New: HA role detection, ordered two-leg orchestration with an abort gate
- New: NMS maintenance-mode suppression (tool TBD)
- New: CRQ close/update operation against BMC Helix
- New (deferred design): compliance checksum evidence capture

### Integrations
- **NetBox** — same `upgrade-catalog` plugin, extended with HA-specific fields (§4)
- **Nexus** — same artifact store, reused as-is
- **BMC Helix** — extends the existing (unwired) `Fetch CRQ Details` integration with a new write/close capability
- **PAN-OS device CLI/API** — new integration; adapter availability TBD at Feasibility
- **NMS (Spectrum or equivalent)** — new integration; tool and adapter availability TBD at Feasibility
- **Panorama** — explicitly NOT required under the current design (§4), pending Feasibility confirmation

## 8. Batch Strategy

Exactly 2 devices per run (one HA pair). Unlike `Agnostic Upgrade`'s loose, largely-unenforced `devices` typing, this use case's entry point should validate `haPair` as exactly a 2-element array — running an odd number of devices through HA-pair logic is a data-entry error, not a valid batch variation. Multi-pair batching (upgrading several HA pairs in one invocation) is not in scope for this spec; each pair is one run.

## 9. Acceptance Criteria

1. Given a target `version`, a valid `haPair`, and `bootMode`, the system correctly identifies which member is Active and which is Passive before making any change.
2. NMS alerting is suppressed for both members before any suspend/upgrade action begins (once designed).
3. The Passive member is always suspended, upgraded, and verified healthy on the target version **before** the Active member is touched.
4. If the Passive member's upgrade fails or rolls back, the Active member is never suspended or upgraded, and a failure notification is sent.
5. Failover to the Active member only proceeds after the Passive member is confirmed healthy on the new version.
6. Each leg's `ViewDiff`-equivalent approval is presented and approved individually before that leg is marked complete.
7. After both legs succeed, both members report the target version and HA re-syncs (mismatches during the transition window are expected and not treated as failures).
8. If `restoreOriginalRoles` is set, the original Active/Passive assignment is restored after both legs complete.
9. A single consolidated notification is sent summarizing both legs' outcomes (not two independent, uncoordinated emails).
10. The associated CRQ is closed/updated on successful completion (once the close operation is built).

> **Note:** criteria 2 and 10 depend on capabilities explicitly deferred to Design (§5) — they cannot be tested until NMS suppression and CRQ-close are designed and built. Criteria 1, 3–9 can be validated once the `HA Upgrade Wrapper` is built against a real (or adequately simulated) PAN-OS HA pair.
