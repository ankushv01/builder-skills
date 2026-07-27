# Use-Case Memory: Palo Alto HA Firewall PAN-OS Upgrade

> Automate the manual, HA-pair-aware PAN-OS upgrade runbook (ref. CRQ00000515633) by extending the proven `Agnostic Upgrade` architecture with a new HA-pair wrapper — reusing the existing per-device orchestrator unmodified.

**Stage:** `test` (build complete — all 8 components deployed and component-tested; hand off to `/qa-agent` when a real target platform/test data exists. No target platform available yet, so acceptance testing cannot proceed until then.)
**Status:** `active`
**Last updated:** 2026-07-27
**Platform:** Not yet connected for this use case — see note below.

> **Resuming this use-case?** `Stage: requirements` means only a draft `customer-spec.md` should exist — no `feasibility.md` yet. Verify against AGENTS.md's "Resuming a Use-Case" table before trusting this field.

---

## Platform References

| What | Value |
|------|-------|
| Platform URL | `https://ws-solution-design-poc-iap01.trial.itential.io` — same sandbox as `AgnosticUpgrade`, reused for this scaffold build per explicit engineer direction (no dedicated target platform exists yet). Has **zero** relevant adapters for this use case's new integrations (no Panorama, no NMS/Spectrum, no BMC Helix write capability) — confirmed by the engineer directly. Feasibility stage must determine the actual target platform and confirm adapter availability there before this can be a real delivery. |
| Project | `Palo Alto HA Upgrade` — `_id` `32f7a975af37635566047d48`, `iid` 1 |
| Related use case | `usecases/AgnosticUpgrade/` — this use case extends its architecture. The `HA Upgrade Wrapper` design originates in `usecases/AgnosticUpgrade/solution-design.md` §J and `customer-spec.md` §11. The built `HA Upgrade Wrapper` childJob-calls `Agnostic Upgrade` (`_id 6a60d7316556606de51220d1`) UNMODIFIED — confirmed correct cross-project reference resolution during testing (see Gotchas). |
| Source document | `Palo Alto OS upgrade step by step process.docx` (copied into this directory) — real manual runbook, ref. `CRQ00000515633`, SCE GSIL firewalls, PAN-OS 11.1.4-h18 target |
| Adapters used (real, this sandbox) | `NetBox` (type `NetboxV33`) for context resolution; `email` (type `EmailOpensource`) for notifications — confirmed genuinely live via a real SMTP rejection during testing |
| Build scripts | `usecases/PaloAltoUpgrade/build/` — `gen.py` (task-builder helper library), `build_wrapper.py` (generates `HA Upgrade Wrapper`), `assemble.py` (generates the full `project_import.json`). Kept for reproducibility — re-run `assemble.py` after any `gen.py`/`build_wrapper.py` edit, then re-validate and re-PUT/re-import. |

---

## What Was Built

| Asset | Type | ID / Name | Status |
|-------|------|-----------|--------|
| customer-spec.md | spec | — | drafted, not yet formally approved by engineer |
| solution-design.md | spec | — | drafted and updated with real IDs + test evidence (§I) |
| Palo Alto HA Upgrade | project | `32f7a975af37635566047d48` | built, all 8 components deployed |
| HA Upgrade Wrapper | workflow | `0cdd078f-3358-42cd-bbb2-56cc8c128afd` | built, component-tested, 2 bugs found+fixed (see Gotchas) |
| Get HA Pair Context | workflow | `460d0e9e-d7f8-45b2-aafe-6e9cad9ab4d5` | built, component-tested (NetBox call times out as expected — no live catalog) |
| Suspend HA Member | workflow | `3308f99a-fb48-4182-9fec-7ad23d58c0e1` | built, component-tested (cross-project childJob resolves correctly; full completion blocked by missing MOP adapter, same as `AgnosticUpgrade`) |
| Verify Peer Traffic | workflow | `ed92d925-84c4-47b3-ae44-b8870419474b` | built, component-tested (same result as Suspend) |
| Resume HA Member | workflow | `9ce0d5bd-db8c-4c26-b767-d870d7299a6f` | built, component-tested (no-op branch confirmed working) |
| NMS Maintenance Mode (STUB) | workflow | `ff0f17aa-ddb1-46c6-83b9-fe468a173cd2` | built, component-tested, passes cleanly (stub) |
| Close CRQ (STUB) | workflow | `5d5b32d1-771e-430d-b7e7-877587acf811` | built, component-tested, passes cleanly (stub) |
| HA Upgrade Form | jsonForm | `6a6767645620cecbbbb68185` | built (had to be created standalone via `/json-forms/forms` then added to the project — see Gotchas) |

The `HA Upgrade Wrapper` design itself still originates in `AgnosticUpgrade/solution-design.md` §J (reusable-by-any-HA-vendor pattern) — this project is its first concrete implementation, explicitly a scaffold (see solution-design.md header and §I for the placeholder/real-adapter boundary).

---

## Architecture Decisions

- **Why extend `Agnostic Upgrade` instead of building a standalone Palo Alto workflow set:** the per-device upgrade mechanics (dynamic MOP templating, reconnect-retry, approval gate, rollback) are already proven for Cisco and are vendor-neutral by construction (NetBox-catalog-driven). Rebuilding them for Palo Alto would duplicate working logic. Only the HA-pair ordering is genuinely new.
- **Why image sourcing skips Panorama entirely:** engineer confirmed the target version is always user-supplied and the image's Nexus location is pre-registered — the manual runbook's Panorama check/download/validate dance exists to *discover* an unknown image, which this use case assumes away. This removes Panorama as a dependency for the imaging half of the problem.
- **Why HA logic is a wrapper, not a modification to `Agnostic Upgrade`:** see `AgnosticUpgrade/use-case-memory.md`'s Architecture Decisions section — same rationale, decided during that session, applies here unchanged.

---

## Gotchas Hit

- **No adapters exist in the available sandbox for any of this use case's new integration points** (Panorama — moot per the imaging decision above, NMS/Spectrum, BMC Helix write). Confirmed directly by the engineer: the sandbox is a bare project import, not a configured environment. Do not attempt to discover/query these via API in this environment — there's nothing to find. Feasibility must happen against whatever the real target platform is.

- **Issue:** Project import failed with `"Argument passed in must be a string of 12 bytes or a string of 24 hex characters or an integer"` on first attempt.
  **Root cause:** the `jsonForm` component's `reference` field was a UUID (correct for `workflow`/`transformation` components) instead of a 24-hex Mongo-style ID (required for `jsonForm` and `template` components — confirmed by checking the real `Upgrade Form` in `AgnosticUpgrade`, whose reference is 24-hex).
  **Fix:** generate 24-hex references for jsonForm/template components, UUIDs only for workflow/transformation.

- **Issue:** After fixing the reference format, the project imported but the jsonForm component itself failed with `"Error: No valid forms were found"`.
  **Root cause:** an empty `struct: []` array — `struct` must contain actual field-definition objects (one per schema property, each with `nodeId`, `type`, `title`, `required`, `binding`, `rel`, etc.), not just mirror `schema.properties`.
  **Fix:** built a proper `struct` array (6 field objects matching each schema property), created the form standalone via `POST /json-forms/forms` (which validated it correctly), then added it to the already-imported project via `POST /automation-studio/projects/{id}/components/add`.

- **Issue:** `HA Upgrade Wrapper`'s first test run failed with `"Job has no available transitions. 901f, 9030, 9029, 9016 could have led to the workflow end task, but did not."`
  **Root cause:** the 4 terminal `mailWithOptions` notification tasks only had success transitions to `workflow_end` — no error transition (Rule 19). Easy to miss on tasks that feel "terminal" and therefore safe to skip.
  **Fix:** added a shared `terminal_err_sink` task (newVariable) that all 4 route to on error, then to `workflow_end`.

- **Issue:** After fixing the mail tasks, the same "no available transitions" error persisted, now also listing the new `terminal_err_sink` task.
  **Root cause:** every `query` task in the workflow (with `pass_on_null: false`) only had a success transition. Once real (empty) NetBox data flowed through the graph, these queries legitimately hit their `failure` state — which had no transition defined, silently halting the job with a misleading error message pointing at unrelated terminal tasks rather than the actual stuck query.
  **Fix:** added a `failure` transition on every query task, routed to the shared error handler. **Lesson for future builds: any `query` task reading a field that might not exist (i.e. any field originating from an external adapter or another workflow's output) needs both `success` and `failure` transitions wired from the start — don't treat query tasks as "safe" the way evaluation tasks are already known to require both branches.**

---

## Test Log

| Date | What was tested | Result | Notes |
|------|----------------|--------|-------|
| 2026-07-27 | NMS Maintenance Mode (STUB) | pass | Stub completes cleanly, `success: true` |
| 2026-07-27 | Close CRQ (STUB) | pass | Stub completes cleanly, `success: true` |
| 2026-07-27 | Get HA Pair Context | pass (structural) | Completes via error-handling branch after a real NetBox adapter timeout — confirms correct adapter reference and error routing, not real catalog data |
| 2026-07-27 | Resume HA Member (no-op path) | pass | Empty-command skip branch confirmed working |
| 2026-07-27 | Suspend HA Member | pass (structural, not awaited to completion) | Correctly resolves cross-project childJob to `Agnostic Upgrade` project's `Create and Run Command Template`; canceled rather than awaited since the MOP adapter isn't configured in this sandbox (known, documented limitation shared with `AgnosticUpgrade`) |
| 2026-07-27 | Verify Peer Traffic | pass (structural, not awaited to completion) | Same result as Suspend HA Member |
| 2026-07-27 | HA Upgrade Wrapper (full run) | pass (after 2 fixes) | Completes cleanly end-to-end under total backend absence: NetBox timeout → graceful failure routing → real SMTP rejection on notification → graceful completion with `success: "failed"` and a clear `emailMessage`. Component-level structural test only — not an acceptance test against real PAN-OS/NetBox/NMS data (none exists here). |

This is component-level testing per the builder-agent process ("confirm a workflow runs without error and wires variables correctly — not verifying end-to-end acceptance criteria"). No acceptance testing has occurred — that's `/qa-agent`'s job, once a real target platform exists.

---

## Open Items

- [ ] Engineer approval of `customer-spec.md`
- [ ] Identify the actual target platform for this delivery (the AgnosticUpgrade sandbox has no usable adapters)
- [ ] Feasibility: confirm Panorama-bypass assumption holds (device-direct HA/install commands actually work on real PAN-OS, don't require Panorama mediation)
- [ ] Feasibility: identify and confirm access to an NMS adapter (or equivalent maintenance-mode API) for alert suppression
- [ ] Feasibility: confirm BMC Helix write/close-CRQ capability is available (only fetch exists today, in the unrelated `Fetch CRQ Details` workflow)
- [ ] Design: compliance checksum (CIP-10-equivalent) evidence capture mechanism — explicitly deferred, not covered by the existing generic pre/post-check pattern
- [ ] Design: exact PAN-OS op-command syntax for role-query/suspend/resume/peer-verify, and confirm resume-on-reboot behavior (automatic vs. explicit)
- [ ] Design: NetBox `upgrade-catalog` schema extension (4 new fields, already specified in `AgnosticUpgrade/solution-design.md` §J) — needs real PAN-OS catalog entries authored
- [ ] Once a real target platform exists: re-run the full `HA Upgrade Wrapper` test with a valid (non-RFC-2606) email and real NetBox catalog data, and expect it to progress into Leg 1 before hitting the MOP-adapter gap
- [ ] Hand off to `/qa-agent` once a real target platform and test data exist — build itself is otherwise complete

---

## Spec Deviations

| Spec said | What was built | Reason |
|-----------|---------------|--------|
| solution-design.md §D listed "Get HA Pair Context" resolving both members' bundles in one call | Built as calling the NetBox adapter once per device (2 separate `genericAdapterRequest` calls within the workflow), not a single batched call | Simpler to build and matches the pattern already proven in `Agnostic Upgrade`'s own NetBox call; NetBox's real API shape for batch lookups was never confirmed since no live catalog exists to test against |
| §D implied `ha_suspend_command`/`ha_resume_command`/etc. would be looked up per-device (once per HA member) | Built extracting these 4 fields **only from `device1Bundle`**, reused for whichever device needs them | HA pair members are identical hardware/software in practice, so the same command syntax applies regardless of which physical unit is currently active/passive — documented simplification, not a functional gap |
