# Itential — Agentic Automation & Orchestration

From spec to delivery — infrastructure automation and orchestration driven by AI agents.

---

## What This Is

A set of AI agent skills for the Itential Platform. Agents deliver automation end-to-end — from requirements through feasibility, design, build, and as-built documentation — or explore and build freestyle.

```
Requirements  →  Feasibility  →  Design  →  Build  →  As-Built
      │                │              │          │           │
  /spec-agent    /solution-       /solution-  /builder-  /builder-
                  arch-agent       arch-agent    agent      agent
      │                │              │          │           │
  customer-       feasibility.md  solution-    assets     as-built.md
  spec.md         (approved)      design.md    (delivered) (approved)
  (approved)                      (approved)
```

---

## Getting Started

```bash
git clone https://github.com/itential/itential-skills.git
cd itential-skills
claude
```

Claude Code loads `CLAUDE.md` and all skills automatically. Point at your platform:

```bash
cp environments/cloud-lab.env my-use-case/.env   # edit with your credentials
```

---

## Interaction Modes

**Deliver from Spec** — structured end-to-end delivery with artifact-based approvals:
```
/spec-agent → /solution-arch-agent → /builder-agent
```

**FlowAgent to Spec** — convert an agent's proven pattern into a deterministic workflow:
```
/flowagent-to-spec → /solution-arch-agent → /builder-agent
```

**Generate Spec from Project** — extract formal documentation from existing automation:
```
/project-to-spec
```

**Explore** — connect to a platform, browse capabilities, build freely:
```
/explore
```

See [`docs/developer-flow.md`](docs/developer-flow.md) for the full flow diagram and design principles.

---

## Skills

| Skill | What It Does |
|-------|-------------|
| `/spec-agent` | Requirements — refine use case, produce approved HLD |
| `/solution-arch-agent` | Feasibility + Design — assess platform, produce solution design |
| `/builder-agent` | Build + As-Built — implement design, test, deliver, document |
| `/flowagent-to-spec` | Read a FlowAgent → produce deterministic workflow spec |
| `/project-to-spec` | Read an existing project → produce spec + design docs |
| `/explore` | Auth, discover platform, browse freely |
| `/flowagent` | Create and run AI agents (LLM providers, tools, missions) |
| `/iag` | IAG services — Python, Ansible, OpenTofu via iagctl |
| `/itential-devices` | Devices, backups, diffs, device groups |
| `/itential-golden-config` | Golden config, compliance, grading, remediation |
| `/itential-mop` | Command templates with validation rules |
| `/itential-inventory` | Device inventories, nodes, actions, tags |
| `/itential-lcm` | Resource models, instances, lifecycle actions |

---

## Spec Library

22 technology-agnostic HLD specs in `spec-files/` covering:

**Networking** — Port Turn-Up, VLAN, Circuit, BGP, VPN, WAN Bandwidth

**Operations** — Software Upgrade, Config Backup, Health Check, Device Onboarding/Decommissioning, Change Management, Incident Remediation

**Security** — Firewall Rules, Cloud Security Groups, SSL Certificates

**Infrastructure** — DNS Records, IPAM, Load Balancer VIP, Config Drift, Compliance Audit

Each spec has 9 sections: Problem Statement, Flow, Phases, Design Decisions, Scope, Risks, Requirements, Batch Strategy, and Acceptance Criteria.

---

## Docs

- [`docs/developer-flow.md`](docs/developer-flow.md) — full lifecycle diagram and design principles
- [`docs/builder-flow.md`](docs/builder-flow.md) — build sequence and import pattern
- [`evals/`](evals/) — skill evals (52 test cases) and e2e test runner
- [`helpers/`](helpers/) — JSON scaffolds and reference workflow patterns
