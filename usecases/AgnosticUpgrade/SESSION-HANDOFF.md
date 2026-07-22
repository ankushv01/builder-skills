# Session Handoff: Agnostic Upgrade — Project-to-Spec Documentation

> Purpose: let a fresh Claude Code session resume this work with zero re-discovery. This file is the narrative/"how we got here" record. `use-case-memory.md` is the durable structured reference (platform IDs, what's built, open items) — read both, but if they ever disagree, trust the actual files over this narrative.

**Session date:** 2026-07-22
**Skill used:** `/project-to-spec`
**Working directory:** `/Users/maanas.pm/builder-skills/usecases/AgnosticUpgrade/`

---

## 1. What was asked

1. Document existing Itential project "408" (a numeric reference that turned out to be the project's `iid`, not its Mongo `_id`) using `/project-to-spec`, with credentials from this directory's `.env`.
2. After initial docs were produced: enhance them using two supplied job export files (`job1.json`, `job2.json`) — real completed production runs — to capture real input/output and schemas.
3. Then: capture per-child-job input/output detail too.
4. Then: list what's missing in the environment that isn't captured in the spec.
5. Now (this file): capture all session data into a handoff file for a new session.

## 2. How to resume — read in this order

1. `use-case-memory.md` — structured facts: platform refs, component inventory, architecture decisions, gotchas, open items. **Start here.**
2. `customer-spec.md` — inferred HLD (business-level, what the automation does, gaps, risks, acceptance criteria, observed example runs).
3. `solution-design.md` — as-built LLD. This is the most detailed document, with sections:
   - §A Environment Summary, §B Component Inventory, §C Adapter Mappings, §D Workflow Structure (core path), §E Data Flow, §F Known Gaps (project-internal), §G Observed Production Job Runs, §H Child Job Reference Schemas (structural, not observed), §I Environment Gaps (platform-level, verified against live adapters/services/triggers)
4. Supporting raw data (not narrative, just evidence — see §5 below).

## 3. Platform & auth

- Platform: `https://ws-solution-design-poc-iap01.trial.itential.io`
- Credentials: `usecases/AgnosticUpgrade/.env` (OAuth client_credentials) — `CLIENT_ID=6a5deb10f7dbd835875b4155`
- Token cache: `.auth.json` (expires hourly — re-POST to `/oauth/token` with the `.env` creds if any call 401s; this happened once mid-session, already handled)
- Client identity note: this specific OAuth client has a **limited, ACL-gated view** of the platform. Several critical findings in this session stem directly from what this client can/can't see — don't assume "not found" or "insufficient permissions" means something doesn't exist. See §6.

## 4. The project itself

- User-given reference "408" = the project's `iid` (platform accepts iid or Mongo `_id` interchangeably at `GET /automation-studio/projects/{id}`)
- Resolved: **project name "Agnostic Upgrade"**, `_id: 6a60d7316556606de51220d1`, `iid: 408`
- 73 components: 30 workflows, 37 transformations, 5 templates, 1 JSON form
- **What it does:** vendor-agnostic network device software upgrade orchestration. Behavior (check/upgrade/rollback commands, validation rules, images) is resolved at runtime from a NetBox plugin catalog (`plugins/upgrade-catalog/upgrade-bundles`) rather than hard-coded per vendor. Entry point: `Upgrade Wrapper` (childJob → `Agnostic Upgrade`, the 64-task core orchestrator). Trigger UI: `Upgrade Form` JSON form.
- Full structural detail is in `solution-design.md` — don't re-derive it, it's already there.

## 5. Raw evidence files in this directory (why each exists)

| File | What it is | Why it's here |
|---|---|---|
| `project-components.json` | Full `GET /automation-studio/projects/408` response | Component inventory source |
| `raw/wf_*.json` | Full detail (`GET /automation-studio/workflows/detailed/{name}`) for all 30 workflows | Task/transition/schema source for §B–§E, §H |
| `raw/tmpl_*.json`, `raw/form_*.json` | Template and JSON form detail | §B template/form rows |
| `workflow-summaries.md` | Condensed per-workflow childJob/adapter task extraction (intermediate working file) | Built before writing solution-design.md; still useful as a quick index |
| `workflow-schemas.md` | Condensed per-workflow typed input/output schema extraction (intermediate working file) | Source for §H; also still useful standalone |
| `job1.json`, `job2.json` | Two engineer-supplied **completed production job exports** | Source for §G (real observed data) — **see §6, these are NOT jobs on the platform in `.env`** |
| `openapi.json` | Full platform OpenAPI spec (14MB) | Used to find `/operations-manager/jobs` query params and `/gateway_manager/v1/services` path — don't re-fetch, it's already here |
| `adapters.json` | `GET /adapters` — all 25 configured adapter instances on this platform | Source for §I adapter-mismatch findings |
| `iag-services.json` | `GET /gateway_manager/v1/services` — all 20 registered IAG services on this platform | Source for §I missing-shell-script finding |
| `om-automations.json` | `GET /operations-manager/automations?limit=500` — all 59 platform-wide OM automations | Source for §I "no trigger found" finding |

## 6. Key corrections made mid-session — do not repeat these mistakes

This session got several things wrong initially and corrected them with evidence. If you're continuing this work, **trust the corrected version** (already reflected in solution-design.md), not the earlier framing:

1. **Initial assumption: childJob references to project `6994dbcc6d90cb331ac67eb3` were "stale/broken."** All childJob tasks in `Agnostic Upgrade` reference workflows like `"@6994dbcc6d90cb331ac67eb3: File Transfer"` — a project ID that isn't project 408's own ID. First read: this was a leftover stale reference from a clone, and project 408's own local same-named workflows were authoritative.
   **Corrected after reading `job1.json`/`job2.json`:** both are real completed jobs named `"@6994dbcc6d90cb331ac67eb3: Agnostic Upgrade"`, dated 2026-07-14. This proves `6994dbcc6d90cb331ac67eb3` is the genuine **live production project**, and project 408 is most likely a **duplicate/snapshot** of it, not the live orchestrator. Don't re-litigate this — see solution-design.md §F.1 and §G for the full evidence trail.

2. **Initial assumption: `pushFileWithTextContentToNexus` (Nexus artifact push) is broken**, because its target project `698e67953d6a63dd1a8b851a` returns "Project not found" via this API client.
   **Corrected after reading the job files:** both jobs show this step succeeding for real, with live Nexus URLs (`https://nexus.gnscet.com/...`). The "not found" is this client's visibility gap (or the project genuinely no longer exists on *this* platform specifically, consistent with §6.1's "project 408 is a snapshot" theory) — not proof the capability is broken. See §F.2.

3. **Chased wrong project IDs during ACL troubleshooting.** The user twice pointed at project IDs that turned out to be red herrings: `409`/`410` (409 = unrelated "Netbox" project; 410 = "BMC-Helix", which *did* resolve the `Fetch CRQ Details` dependency) and `67d8f9e1a2b3c4d5e6f78901` ("NetBox Device Software Upgrade" — completely unrelated content, 5 different workflow names). **Do not re-try these as if they're the missing `6994dbcc6d90cb331ac67eb3`/`6984ff4371ab83a64cbf75c9` — they are confirmed different, unrelated projects.**

4. **Two cross-project references were never resolved and remain genuinely open:**
   - `6994dbcc6d90cb331ac67eb3` — insufficient permissions, never granted despite multiple retries this session
   - `6984ff4371ab83a64cbf75c9` — insufficient permissions, holds `TAD Transfer File - IOS-XR` (only used by the unwired legacy subsystem, so low priority)
   - If a future session gets ACL access to `6994dbcc6d90cb331ac67eb3`, that would let §F.1's "snapshot" theory be confirmed/refuted directly — this is the single highest-value follow-up.

## 7. Environment gaps found (platform-level, not just project-level — solution-design.md §I)

Verified by cross-checking the project's workflow definitions against what's actually configured on **this** platform (the one in `.env`):

- `netbox-grid` adapter (used by `Get Netbox Device and Context`) does not exist. Real NetBox instances: `NetBox`, `Netbox_March_2026`, `Netbox-Maanas`.
- `Email` adapter_id (used by `Upgrade Wrapper`) doesn't case-match either real instance (`email`, `Email-2`).
- The 3 AGManager shell scripts `File Transfer` depends on (`agnostic_load_image.py`, `agnostic_scp.py`, `scp_cisco.py`) are absent from all 20 registered IAG services on this platform.
- Zero of the platform's 59 OM automations reference project 408 — the `Upgrade Form` → `Upgrade Wrapper` trigger wiring is unconfirmed here.
- `Github-TAD` adapter (legacy subsystem dependency) is inactive.

These reinforce §6.1: this platform is very likely a documentation/trial sandbox holding a *copy* of the real project, not the environment it actually runs in.

## 8. Open items (also in use-case-memory.md — kept in sync, check both)

- [ ] Get ACL access to `6994dbcc6d90cb331ac67eb3` to confirm/refute the "project 408 is a snapshot" theory directly
- [ ] Get ACL access to `6984ff4371ab83a64cbf75c9` (lower priority — only feeds the unwired legacy subsystem)
- [ ] Child-job-level real input/output (Get Netbox Device and Context, File Transfer, etc.) still undocumented from real execution — only structural schemas exist (§H). Would need either child-job export files or access to wherever `job1.json`/`job2.json` actually ran.
- [ ] Resolve the 5 environment gaps in §7/§I with the engineer — confirm which are real blockers vs. sandbox-only artifacts
- [ ] Decide fate of the 11-workflow legacy GitHub-manifest subsystem (archive vs. keep)
- [ ] Decide whether `Fetch CRQ Details` (BMC Helix change-gate) should be wired into the main flow
- [ ] Confirm whether "no rollback on reconnect-timeout exhaustion" is intentional
- [ ] Clean up 4 dev/scratch workflows (`test`, `testEmail`, `test email`, `update ref test`)

## 9. Decisions the engineer has already made this session (don't re-ask)

- Grant ACL access to project 408 → done, confirmed working
- Grant ACL access to project 410 (BMC-Helix) → done, confirmed working
- When childJob project-prefix mismatch was found: use project 408's local copies as authoritative → **later superseded by the job-evidence correction in §6.1** — if this comes up again, present the corrected understanding, not the original decision
- When child-job-level real data couldn't be sourced: document from workflow structure/schema instead (structural, clearly labeled) → done, this is §H
- Stage in `use-case-memory.md` is set to `build` (not `delivered`) because of the unresolved defects/gaps found — this was my judgment call, not an explicit engineer decision; revisit if the engineer disagrees

## 10. If starting fresh, the fastest path is

1. Read `use-case-memory.md` in full (small, structured, current).
2. Skim `solution-design.md` §F, §G, §I — that's where all the hard-won corrections and evidence live.
3. Do NOT re-fetch `project-components.json`, `raw/*.json`, `adapters.json`, `iag-services.json`, `om-automations.json`, or `openapi.json` — all already pulled and current as of this session's date. Only re-fetch if the engineer indicates the platform has changed.
4. Do NOT re-attempt ACL troubleshooting on `409`, `67d8f9e1a2b3c4d5e6f78901` — confirmed unrelated. Only `6994dbcc6d90cb331ac67eb3` and `6984ff4371ab83a64cbf75c9` are the real open cross-project targets.
