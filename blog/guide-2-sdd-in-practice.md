# Spec-Driven Development for Infrastructure Automation — Four Use Cases

Spec-Driven Development is not abstract. The five stages — Requirements, Feasibility, Design, Build, As-Built — apply directly to the infrastructure use cases teams deliver every day.

This guide walks through four of them: port turn-up, load balancer VIP provisioning, DNS record management, and firewall rule lifecycle. Each one shows what SDD looks like in practice, what each stage produces, and why the structure matters for that specific use case.

---

## What SDD Looks Like in Practice

Every use case moves through the same five stages. What changes is the content — the adapters, the systems, the constraints, the reuse opportunities, the edge cases specific to that domain.

The Requirements stage does not touch the environment. It is a structured conversation: what is being requested, what the scope is, what the acceptance criteria are, what business context matters. That conversation produces a spec the customer approves before anything else happens.

The Feasibility stage connects to the platform and checks what is actually available. What adapters are running? What existing workflows can be reused? What constraints does this environment impose on the approved requirements? The output is an assessment and a decision to proceed.

The Design stage turns approved feasibility into a concrete plan. What components need to be built? What already exists and can be reused? In what order should things be built and tested? What defines done for each component?

The Build stage executes that plan. Components are built in dependency order — simpler pieces first, composite orchestration last. Each component is tested before the next layer depends on it.

The As-Built stage records what actually happened. Where did delivery diverge from the design? What was learned? The artifacts get updated so the next team starts from truth, not recollection.

That is the pattern. Here is what it looks like across four use cases.

---

## Port Turn-Up

Port turn-up is one of the most common network delivery tasks: configure a switch port for a new device, validate the interface is up, confirm VLAN assignment, update inventory.

It looks simple. In practice it is where delivery discipline shows its value most clearly.

**Requirements** captures what seems obvious but rarely is: which port, which device, which VLAN, what constitutes a successful turn-up, what happens if the port is already in use, whether a change ticket is required. The spec captures these questions and gets them approved before anyone opens a CLI.

**Feasibility** confirms what the platform can actually do. Is an adapter available and connected to the target device? Does a pre-check command template already exist for interface validation? Are there existing port turn-up workflows that could be reused or extended?

**Design** produces the component plan. A pre-check template validates the interface state before the change. A Jinja2 template generates the configuration. A child workflow applies the config and verifies the result. A parent workflow orchestrates the sequence, handles the ITSM ticket if required, and runs the post-check. The design records what is net-new and what is reused.

**Build** follows that plan. The pre-check template is built and tested against a real device before anything else is touched. The Jinja2 template is validated independently. The child workflow is run standalone. Only after each piece is proven does the parent orchestration get assembled.

**As-Built** records the actual configuration pushed, the device names, the template versions used, and any deviations from the design. That record becomes the baseline if the same use case is delivered again in a different environment.

The deeper value is not the automation itself. It is that every decision — which VLAN, which validation rule, which error handling path — is captured explicitly rather than embedded silently in the workflow.

---

## Load Balancer VIP Provisioning

Load balancer VIP provisioning involves more moving parts: allocate an IP from IPAM, configure the VIP with pool members and health monitors, set up persistence profiles, update the change ticket, and verify the VIP is serving traffic.

Each system has its own API, its own failure modes, and its own sequence dependencies. That is exactly where delivery without structure breaks down.

**Requirements** captures the full business need clearly: what application this VIP is for, what the pool members are, what health monitor type is required, what persistence is needed, what constitutes a successful provisioning. It also surfaces the business rules that often live only in someone's head — which IP ranges are reserved, what naming conventions apply, whether DNS needs to be updated alongside the VIP.

**Feasibility** checks the real environment. Is the IPAM adapter running and connected to the right network block? Does the load balancer adapter support the required configuration calls? Are there existing workflows for IPAM allocation or health monitor configuration that can be reused?

**Design** maps out the dependency chain explicitly: IPAM allocation must succeed before the VIP is created. Health monitors must be created before pool members are added. The VIP configuration is last. Rollback paths are defined at each step — if VIP creation fails, the allocated IP must be released. The design makes these dependencies visible before build begins.

**Build** respects that dependency order. The IPAM allocation child workflow is built and tested first. The health monitor workflow follows. The VIP creation workflow comes after. The parent orchestration is assembled only after all dependencies are validated.

**As-Built** records the allocated IP, the VIP ID, the pool configuration, and any deviations from the planned sequence.

The key contribution of SDD here is visible dependency management. Multi-system provisioning is where delivery systems most often lose coherence. SDD forces those dependencies to be designed explicitly rather than discovered during build.

---

## DNS Record Management

DNS record management looks administrative. In practice it is high-stakes: a bad record can take down services, propagation is not instant, and rollback without a snapshot is guesswork.

**Requirements** captures the full scope of what reliable DNS automation needs: conflict detection before any change is applied, propagation verification by querying resolvers rather than sleeping for a fixed duration, rollback to a snapshot if verification fails, and an evidence report for every run regardless of outcome. It also captures the business constraints — which provider, which record types, whether PTR sync is required, whether production zones need approval.

**Feasibility** checks the real environment against those requirements. Is the Route53 or Infoblox adapter running? Are there existing DNS workflows that can be reused? Does the platform support the query tasks needed for propagation verification?

**Design** maps the component structure: a pre-flight check child workflow validates the zone, detects conflicts, and takes a snapshot. An execute child workflow applies the record change. A verification child workflow queries resolvers and confirms propagation. A rollback child workflow restores the snapshot if verification fails. An orchestrator sequences them, gates production changes on approval, generates the evidence report, and updates the ticket.

**Build** tests each child independently before the orchestrator is assembled. The pre-flight check is run against a real zone. The execute workflow is run against a test record. The verification workflow is confirmed to use actual resolver queries rather than a fixed sleep. Only after each piece is validated does the orchestrator get built.

**As-Built** records what was actually deployed — the record name, value, TTL, provider, actual propagation timing, and whether rollback was triggered.

DNS automation done without structure tends to drift toward fragility — changes that work most of the time, with no recovery path and no audit trail when they don't. SDD forces recovery and auditability to be designed in from the start.

---

## Firewall Rule Lifecycle

The firewall use case is different from the others. It is not a one-time delivery. It is a managed lifecycle: a rule is requested, validated, deployed, verified, periodically recertified, and eventually decommissioned.

**Requirements** captures the full lifecycle scope up front, not just the creation step. Who can request a rule? What validation is required before deployment? What constitutes a deployed and verified rule? When does recertification happen and who approves it? What is the decommission process when a rule expires? These are requirements that most teams defer until the first failure. SDD captures them before build begins.

**Feasibility** checks the real environment: which firewall platform is in use, what adapter capabilities are available, whether policy validation can be automated, whether a ticketing system is available for the approval workflow.

**Design** maps the lifecycle as a set of coordinated workflows rather than a single automation. A rule request workflow handles creation and deployment. A recertification workflow handles periodic review. A decommission workflow handles removal and cleanup. Each is independently testable. The orchestration layer ties them together. Reuse decisions are explicit — the validation logic used at creation time is the same logic used at recertification.

**Build** follows the dependency order. Validation logic is built and tested first. Creation and deployment workflows follow. The recertification workflow reuses the same validation component. The decommission workflow is built last because it depends on the rule record created by the first workflow.

**As-Built** for a lifecycle use case is especially important. The record captures not just what was deployed, but what changed across each lifecycle phase — which rules were recertified, which were decommissioned, what the approval history looks like.

The firewall lifecycle use case illustrates something the other three do not: SDD is not just for point-in-time delivery. It applies to any infrastructure use case that has phases, approvals, and a need for ongoing governance.

---

## What These Use Cases Share

Across port turn-up, load balancer VIP, DNS record management, and firewall rule lifecycle — four different domains, four different sets of adapters and systems — the same SDD pattern holds.

Requirements captures intent before the environment is touched. Feasibility checks reality against approved intent. Design makes dependencies explicit and reuse decisions visible. Build follows the approved plan in dependency order, testing each component before composing the next. As-Built records what actually became true.

The domain changes. The artifacts, the approvals, and the structure do not.

Infrastructure use cases are varied enough to require domain-specific knowledge, but they cluster around repeatable patterns — validate inputs, inspect the environment, allocate resources, configure systems, verify outcomes, update systems of record, handle exceptions. SDD provides the structure that makes those patterns governable and reusable across every domain.

Each use case leaves behind verified components, explicit reuse decisions, and recorded as-built truth that makes the next similar delivery faster, more reliable, and easier to govern.

That is what Spec-Driven Development looks like in practice.
