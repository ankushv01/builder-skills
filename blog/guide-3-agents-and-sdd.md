# SDD and Agentic Operations with Itential

AI is moving into infrastructure operations. Agents are taking on tasks that previously required human judgment — exploring environments, assembling workflows, responding to conditions in real time. That is a meaningful shift. But it also raises a question that most organizations are not yet asking clearly enough:

How do you govern what AI does on your infrastructure?

Spec-Driven Development is the answer.

---

## The Governance Problem in Agentic Operations

Agentic operations introduce a new kind of delivery risk. When a human engineer builds a workflow, their decisions are visible — in the code, the config, the pull request, the change ticket. When an AI agent operates on infrastructure, its decisions can be fast, effective, and nearly invisible. The right things may happen. But the record of why they happened, what was considered, what was approved, and what actually changed can be thin or absent.

That creates a governance gap.

It is not that agents make bad decisions. It is that the delivery system around them has not caught up. Most organizations still rely on informal checkpoints, tribal knowledge, and after-the-fact documentation. Those mechanisms were already strained under human-led delivery. In an AI-assisted model they become even more inadequate.

The organizations that operationalize agentic infrastructure well are the ones that treat governance as a first-class requirement — not an afterthought, not a checkbox, but a structural part of how AI participates in delivery.

---

## Why SDD Is the Governance Layer

Spec-Driven Development provides what agentic operations lack by default: a structured, artifact-driven chain from intent to delivered outcome.

In SDD, no agent touches the environment until intent is locked. No design work begins until feasibility is assessed and approved. No build begins until the design is reviewed and signed off. And after delivery, what actually happened is recorded — deviations, learnings, amendments — so the artifacts remain honest and usable going forward.

That chain does not slow down AI-assisted delivery. It makes it governable.

Consider what changes when SDD wraps agentic operations:

The requirement is explicit before anything runs. An agent operating from a locked requirements spec is operating within approved scope — not redefining the problem as it goes.

The design is approved before build. An agent that executes a locked solution design is traceable. If something goes wrong, the design is the reference point. Deviations are visible because there is something to deviate from.

The as-built record captures reality. After delivery, what was actually done is documented explicitly — not inferred from log files or reconstructed from memory. That record travels with the artifacts and becomes the baseline for future work.

That is what governance looks like in practice for agentic operations. Not restrictions on what agents can do, but structure around how their work fits into a system that is accountable and reusable.

---

## How Itential Operationalizes SDD

Itential does not treat SDD as a methodology teams adopt manually. The platform operationalizes it — each stage of the delivery lifecycle is owned by a dedicated agent, the handoffs between stages are artifact-based, and every approval produces a document that travels with the delivery.

**The Spec Agent owns Requirements.** It works with the engineer to define the use case, capture business context, refine scope, and produce a requirements spec — `customer-spec.md` — that the customer approves before anything touches the environment. No discovery, no API calls, no platform access. Pure structured conversation that ends with a locked artifact.

**The Solution Architecture Agent owns Feasibility and Design.** After the spec is approved, it connects to the platform, assesses what adapters and workflows are available, identifies reuse opportunities, and flags constraints. That assessment — `feasibility.md` — gets approved. Then it produces a solution design — `solution-design.md` — that maps exactly what gets built, what gets reused, and in what order. That design gets approved before build begins.

**The Builder Agent owns Build and As-Built.** It receives a complete workspace: the approved spec, the feasibility assessment, the approved design, and all platform data needed to execute. It builds from the locked plan — dependencies first, orchestration last — tests each component before composing the next, and delivers the project. After delivery, it produces `as-built.md`: what was actually built, where it diverged from the design, and what was learned. The design document is updated. The spec is amended if scope changed.

The handoffs between agents are file-based. No verbal instructions. No assumptions about what the previous stage discovered. The next agent reads the artifacts the previous one produced and acts from there.

Each agent produces an artifact. Each artifact requires approval. Nothing moves forward on assumption.

```
Requirements  →  Feasibility  →  Design  →  Build  →  As-Built
      │                │              │          │           │
  Spec Agent    Solution Arch.   Solution    Builder     Builder
                   Agent          Arch. Agent  Agent       Agent
      │                │              │          │           │
  customer-       feasibility.md  solution-    assets/    as-built.md
  spec.md         (approved)      design.md    configs    (approved)
  (approved)                      (approved)   (delivered)
```

That is SDD operationalized — not as a process teams have to remember to follow, but as a working delivery system with explicit ownership, explicit approvals, and explicit artifacts at every stage. The platform runs it. The engineers own the decisions.

---

## The Delivery Record That Stays

The most durable part of the Itential model is what it leaves behind.

Every delivery produces a chain of artifacts: the approved requirements spec, the feasibility assessment, the solution design, the delivered assets, and the as-built record. Those artifacts do not just document what happened. They become the starting point for the next delivery.

On a future engagement with the same use case, the team does not start from a blank page. They start from a reconciled spec and a proven design. The feasibility assessment reflects what was actually true in that environment. The as-built record captures the real implementation details — not what the design said, but what was built and why it diverged if it did.

That is a fundamentally different starting position than "we did something similar last time, let me find the workflow."

It is delivery truth that compounds. Each engagement makes the next one more grounded, more reusable, and faster to execute.

---

## What This Means for Infrastructure Teams

The infrastructure teams that will operate most effectively with AI are not the ones that give agents the most autonomy. They are the ones that build the right governance structure around that autonomy.

SDD is that structure. It keeps intent explicit. It keeps discovery scoped to approved requirements. It keeps build accountable to an approved design. It keeps the as-built record honest.

Itential brings that structure to life as a set of agents and artifacts that work together — not as a theoretical framework, but as a working delivery system that teams can use today.

Agentic infrastructure operations are not inherently risky. They become reliable when the operating model around them is strong. SDD is that operating model. And Itential is how it gets built.
