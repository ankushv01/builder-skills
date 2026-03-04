---
name: iag
description: Build and run IAG (Itential Automation Gateway) services — Python scripts, Ansible playbooks, OpenTofu plans. YAML-driven service definitions, imported with iagctl. Call services from Itential workflows via GatewayManager.
argument-hint: "[action or service-name]"
---

# IAG — Itential Automation Gateway

IAG exposes Python scripts, Ansible playbooks, and OpenTofu plans as REST APIs. Everything is defined in YAML, imported with `iagctl db import`.

```
Write YAML → iagctl db import → Services available → Workflows call them
```

---

## How It Works

1. **Write a YAML service file** — defines repos, decorators, secrets, services
2. **`iagctl db import`** — loads into IAG
3. **`iagctl run service`** — test from CLI
4. **`GatewayManager.runService`** — call from Itential workflows

**Always start from a helper template.** Read the matching example from `helpers/iag/` first, then modify:
- Python service → `helpers/iag/example-python-service.yaml`
- Ansible service → `helpers/iag/example-ansible-service.yaml`
- OpenTofu service → `helpers/iag/example-opentofu-service.yaml`
- Multi-service chain → `helpers/iag/example-multi-service-chain.yaml`
- Full schema reference → `helpers/iag/service-file-schema.md`

**Do NOT build YAML from scratch. Read the helper first.**

---

## Authentication

| Mode | Auth | How |
|------|------|-----|
| **Local** | None needed | `iagctl` talks to local IAG directly |
| **Server/Client** | Login required | `iagctl login <username>` → interactive password prompt |
| **Itential workflows** | Pre-configured | Platform admin sets up gateway. `clusterId` references it. |

**The agent cannot run `iagctl login`** — it requires an interactive terminal. If the engineer hasn't logged in yet, tell them:
> "Run `iagctl login admin` in your terminal and enter your password. Once done, I can continue."

Quick check — if this works, you're authenticated:
```bash
iagctl get services
```

---

## Writing Service Files

### YAML Structure

A service file has these top-level sections (all optional — include only what you need):

```yaml
decorators: []      # Input schemas for services
repositories: []    # Git repos with code
services: []        # Python/Ansible/OpenTofu services
registries: []      # Package registries (PyPI, Galaxy)
secrets: []         # Credentials and keys
```

### Service Types

| Type | Key fields | Runs |
|------|-----------|------|
| `python-script` | `filename` or `runtime.pyproj-script` | Python file from repo |
| `ansible-playbook` | `playbooks` (array) | Ansible playbook(s) from repo |
| `opentofu-plan` | `plan-vars`, `plan-var-files` | OpenTofu apply/destroy |
| `executable` | `filename`, `arg-format` | Custom executable |

### Minimal Python Service

```yaml
repositories:
  - name: my-repo
    url: https://github.com/org/repo.git
    reference: main

services:
  - name: my-service
    type: python-script
    filename: main.py
    working-directory: scripts
    repository: my-repo
```

### Minimal Ansible Service

```yaml
services:
  - name: my-playbook
    type: ansible-playbook
    playbooks:
      - site.yml
    working-directory: ansible
    repository: my-repo
```

### Minimal OpenTofu Service

```yaml
services:
  - name: my-plan
    type: opentofu-plan
    working-directory: terraform
    repository: my-repo
```

### Adding Input Validation (Decorators)

```yaml
decorators:
  - name: my-inputs
    schema:
      $id: root
      $schema: https://json-schema.org/draft/2020-12/schema
      type: object
      required:
        - device_ip
      properties:
        device_ip:
          type: string
          description: Target device IP
        dry_run:
          type: boolean
          default: false

services:
  - name: my-service
    type: python-script
    filename: main.py
    working-directory: scripts
    repository: my-repo
    decorator: my-inputs          # ← links to decorator above
```

### Adding Secrets

```yaml
secrets:
  - name: api-token
    value: "your-secret-value"    # raw value in YAML — be careful with this

services:
  - name: my-service
    type: python-script
    filename: main.py
    working-directory: scripts
    repository: my-repo
    secrets:                       # injected as env vars at runtime
      - name: api-token
        type: env
        target: API_TOKEN          # your script reads os.environ['API_TOKEN']
```

**Note:** For sensitive secrets, prefer `iagctl create secret <name> --prompt-value` (interactive, never in file) over putting raw values in YAML.

**WARNING:** `--force` import overwrites secrets too. If your YAML has placeholder secret values (e.g., `REPLACE_VIA_IAGCTL_OR_VAULT`), a `--force` import will replace real secrets with placeholders, breaking your service. **Best practice: keep the top-level `secrets:` section out of `services.yaml` entirely.** Define secret references inside the service (the `secrets:` array under each service), but create the actual secrets separately with `iagctl create secret --prompt-value`. This way `--force` imports never touch your credentials.

### Private Git Repos

```yaml
secrets:
  - name: git-ssh-key
    value: "REPLACE_WITH_SSH_KEY"

repositories:
  - name: private-repo
    url: git@github.com:org/private.git
    private-key-name: git-ssh-key    # SSH auth
    reference: main

  # OR for HTTPS:
  - name: https-repo
    url: https://github.com/org/repo.git
    username: myuser
    password-name: git-password      # name of secret with password
```

---

## Import / Export

```bash
# Validate only (no changes)
iagctl db import services.yaml --validate

# Dry run with checks
iagctl db import services.yaml --check

# Import (additive — new added, existing skipped)
iagctl db import services.yaml

# Import with overwrite (existing replaced by name)
iagctl db import services.yaml --force

# Export current state
iagctl db export state.yaml

# Import directly from Git repo
iagctl db import --repository https://github.com/org/repo.git --reference main
```

**Import behavior:**
- New resources → **added**
- Existing (same name) → **skipped** without `--force`, **replaced** with `--force`
- Resources not in the YAML → **untouched** (never deleted)

---

## Development Loop

When iterating on service code, every change requires pushing to Git and re-importing — IAG pulls code from the repo, not from local files.

```
Edit code → git commit + push → iagctl db import services.yaml --force → iagctl run service → repeat
```

**Tip:** Keep secrets out of `services.yaml` so `--force` imports don't clobber them (see Secrets warning above).

---

## Testing Services (CLI)

```bash
# List services
iagctl get services
iagctl get services --type python-script

# See what inputs a service expects
iagctl run service python-script my-service --use

# Run with inputs
iagctl run service python-script my-service \
  --set device_ip=10.0.0.1 \
  --set device_type=ios

# Ansible
iagctl run service ansible-playbook my-playbook --set target_host=router1

# OpenTofu apply
iagctl run service opentofu-plan apply my-plan --set region=us-east-1

# OpenTofu destroy
iagctl run service opentofu-plan destroy my-plan

# Raw JSON output
iagctl run service python-script my-service --raw
```

---

## Calling IAG from Itential Workflows

### Finding the clusterId

The `clusterId` is required for all GatewayManager tasks. Discover it via the platform API:

```
GET /gateway_manager/v1/gateways/
```

This returns the list of configured gateway clusters. Use the cluster name as the `clusterId` value in workflow tasks.

### GatewayManager Tasks

| Task | What it does |
|------|-------------|
| `runService` | Run an IAG service by name |
| `sendCommand` | Send CLI commands to inventory nodes |
| `sendConfig` | Send config text to inventory nodes |
| `getServices` | List available services |
| `getGateways` | List connected gateways |

### runService Task Wiring

```json
{
  "name": "runService",
  "app": "GatewayManager",
  "type": "automatic",
  "location": "Application",
  "displayName": "GatewayManager",
  "actor": "Pronghorn",
  "variables": {
    "incoming": {
      "serviceName": "device-info",
      "clusterId": "ankitcluster",
      "params": {"device_ip": "10.0.0.1", "device_type": "ios"},
      "inventory": ""
    },
    "outgoing": {
      "result": "$var.job.iagResult"
    }
  }
}
```

**Incoming:**
| Field | Type | Description |
|-------|------|-------------|
| `serviceName` | string | IAG service name (same name as in YAML/iagctl) |
| `clusterId` | string | Gateway cluster ID — ask the engineer |
| `params` | object | Key/value inputs matching the decorator schema |
| `inventory` | array or `""` | Target nodes: `[{"inventory": "inv-name", "nodeNames": ["node1"]}]` or `""` if not needed |

**Outgoing:**
| Field | Type | Description |
|-------|------|-------------|
| `result` | object | JSON-RPC envelope with service execution result |

### Result Shape — JSON-RPC Wrapper

`runService` returns a JSON-RPC envelope, NOT raw stdout:

```json
{
  "id": "dc7c4a5d-...",
  "jsonrpc": "2.0",
  "result": {
    "return_code": 0,
    "stdout": "{ ... script output ... }",
    "stderr": "",
    "start_time": "2026-03-03T19:26:37Z",
    "end_time": "2026-03-03T19:26:37Z",
    "elapsed_time": 0.659
  },
  "status": "completed"
}
```

**To extract stdout in a workflow:** use a `query` task with path `result.stdout`:

```json
{
  "name": "query",
  "app": "WorkFlowEngine",
  "type": "operation",
  "variables": {
    "incoming": {
      "pass_on_null": false,
      "query": "result.stdout",
      "obj": "$var.job.iagResult"
    },
    "outgoing": {
      "return_data": "$var.job.serviceOutput"
    }
  }
}
```

### Chaining Services in a Workflow

Pass output from one service as input to the next:

```
runService(device-info)
    → query: extract result.stdout → parse JSON
        → runService(config-generator) with params from previous output
            → query: extract result.stdout
                → runService(config-validator)
```

Each `query` extracts `result.stdout` from the JSON-RPC envelope. If the stdout is JSON, parse it before passing as params to the next service.

### sendCommand Task Wiring

```json
{
  "name": "sendCommand",
  "app": "GatewayManager",
  "type": "automatic",
  "actor": "Pronghorn",
  "variables": {
    "incoming": {
      "clusterId": "ankitcluster",
      "commands": ["show version", "show ip interface brief"],
      "inventory": [{"inventory": "my-inventory", "nodeNames": ["router1"]}]
    },
    "outgoing": {
      "result": "$var.job.commandResult"
    }
  }
}
```

### sendConfig Task Wiring

```json
{
  "name": "sendConfig",
  "app": "GatewayManager",
  "type": "automatic",
  "actor": "Pronghorn",
  "variables": {
    "incoming": {
      "clusterId": "ankitcluster",
      "config": "$var.job.renderedConfig",
      "inventory": [{"inventory": "my-inventory", "nodeNames": ["switch1"]}]
    },
    "outgoing": {
      "result": "$var.job.configResult"
    }
  }
}
```

### Testing IAG Services via Workflow

After CLI testing passes (`iagctl run service`), test the full workflow integration:

**1. Create the workflow** (runService → query to extract stdout):
```
POST /automation-studio/automations
```

**2. Start a job:**
```
POST /operations-manager/jobs/start
```
```json
{
  "workflow": "My IAG Workflow",
  "options": {
    "type": "automation",
    "variables": {
      "device_ip": "172.20.100.63",
      "device_type": "cisco_xr",
      "interfaces": "GigabitEthernet0/0/0/0",
      "clusterId": "ankitcluster"
    }
  }
}
```

**3. Check the job:**
```
GET /operations-manager/jobs/{jobId}
```
Verify:
- `data.status` is `"complete"` (not `"error"`)
- `data.error` is `null` (no task errors)
- `data.variables.serviceOutput` contains the extracted stdout from the IAG service

**If the job errors with "Service not found on cluster":** the `clusterId` is wrong. Check `GET /gateway_manager/v1/gateways/` for the correct cluster name.

---

## When to Use Which

| Need | Use |
|------|-----|
| Run a Python/Ansible/OpenTofu service | `GatewayManager.runService` |
| Send ad-hoc CLI commands | `GatewayManager.sendCommand` or `AGManager.itential_cli` |
| Push config text to device | `GatewayManager.sendConfig` or `AGManager.itential_set_config` |
| Run MOP validation checks | `MOP.RunCommandTemplate` (separate from IAG) |

### AGManager vs GatewayManager

| | AGManager | GatewayManager |
|---|-----------|---------------|
| **Tasks** | One per script/playbook (e.g., `itential_cli`) | Generic (`runService`, `sendCommand`) |
| **Input style** | Task-specific variables | `serviceName` + `params` object |
| **When to use** | Built-in IAG capabilities | Custom services built with iagctl |

---

## Operational Commands (Inspect, Verify, Clean Up)

After importing, use these to verify and manage resources:

```bash
# === LIST RESOURCES ===
iagctl get services
iagctl get services --type python-script
iagctl get services --type ansible-playbook
iagctl get services --type opentofu-plan
iagctl get repositories
iagctl get secrets
iagctl get decorators
iagctl get registries
iagctl get clusters                          # find clusterId for workflows

# === INSPECT A SPECIFIC RESOURCE ===
iagctl describe service <name>               # full details: repo, decorator, secrets, runtime
iagctl describe repository <name>            # URL, reference, auth method
iagctl describe decorator <name>             # JSON schema
iagctl describe secret <name>                # secret metadata (value redacted)

# === DELETE ===
iagctl delete service <name>
iagctl delete repository <name>
iagctl delete decorator <name>
iagctl delete secret <name>

# === EXPORT CURRENT STATE ===
iagctl db export current-state.yaml          # full dump of everything in IAG
```

**After every import, verify with:**
```bash
iagctl describe service <name>
```
This confirms the service was created with the correct repo, decorator, secrets, and working directory.

---

## Organizing Services for Teams

### Naming Conventions

Consistent naming prevents collisions when multiple teams deploy to the same IAG.

```
Services:     {team}-{domain}-{action}        e.g. netops-device-health-check
Decorators:   {service-name}-decorator         e.g. netops-device-health-check-decorator
Repositories: {team}-{purpose}                 e.g. netops-automation
Secrets:      {team}-{system}-{purpose}        e.g. netops-git-ssh-key
```

Tag services with ownership:
```yaml
services:
  - name: netops-health-check
    tags:
      - team:netops
      - domain:network
```

Filter by team: `iagctl get services --tag team:netops`

### Repository Structure

**Small team (< 20 services) — mono-repo:**
```
automation-services/
├── services.yaml              ← one file defines everything
├── network/
│   ├── device-info/main.py
│   └── config-push/main.py
├── cloud/
│   └── vpc-deploy/main.tf
└── decorators/
    └── device-input.json
```

**Large team (20+ services) — multi-repo, each team owns a repo:**
```
Team: Network     → repo: netops-automation/services.yaml
Team: Cloud       → repo: cloudops-automation/services.yaml
Team: Security    → repo: secops-automation/services.yaml
```

Each repo has its own `services.yaml`. Teams import independently — import is additive.

**Separation of concerns — service definitions separate from code:**
```
iag-service-definitions/          ← YAML service files only
├── network-services.yaml
├── cloud-services.yaml
└── shared-decorators.yaml

netops-automation/                ← code only (scripts, playbooks)
├── device-info/main.py
└── config-push/main.py
```

Service YAML references the code repo:
```yaml
repositories:
  - name: netops-automation
    url: git@github.com:org/netops-automation.git
services:
  - name: device-info
    repository: netops-automation   # cross-repo reference
    working-directory: device-info
    filename: main.py
```

### Service File Patterns

**Self-contained** — one file per use case, includes repo + decorator + service:
```yaml
# network-health-check.yaml — everything for one service
decorators:
  - name: netops-health-check-decorator
    schema: ...
repositories:
  - name: netops-automation
    url: ...
services:
  - name: netops-health-check
    repository: netops-automation
    decorator: netops-health-check-decorator
    ...
```

**Shared base + service files** — common repos/decorators imported first:
```bash
iagctl db import base.yaml          # shared repos, decorators
iagctl db import health-check.yaml  # just the service, references base resources
iagctl db import config-push.yaml   # another service, same base
```

### Environment Promotion

```
Dev:     services.yaml with reference: devel    → iagctl db import --force
Staging: services.yaml with reference: release  → iagctl db import --check then import
Prod:    services.yaml with reference: v1.2.3   → iagctl db import --validate → --check → import
```

| Setting | Dev | Staging | Production |
|---------|-----|---------|------------|
| Git `reference` | branch | release branch | tagged version |
| Secrets | `--prompt-value` | vault | vault only |
| Import mode | `--force` | `--check` first | `--validate` → `--check` → import |
| Who imports | developer | CI/CD | CI/CD with approval |

### Secrets — Never in Git

```bash
# Create interactively (recommended)
iagctl create secret netops-git-key --prompt-value

# Or inject from vault in CI/CD
iagctl create secret netops-git-key --value "$VAULT_GIT_KEY"
```

Only put placeholder values in YAML files committed to Git:
```yaml
secrets:
  - name: netops-git-key
    value: "REPLACE_VIA_IAGCTL_OR_VAULT"    # never the real value
```

---

## Before Handing Off

- [ ] Service has a decorator (input validation + API docs)
- [ ] Service tested individually: `iagctl run service <type> <name> --set ...`
- [ ] Service output is valid JSON if it feeds into other services
- [ ] Itential workflow tested end-to-end with `runService` task
- [ ] Workflow correctly extracts `result.stdout` from JSON-RPC envelope
- [ ] Error handling in workflow: what happens when the service fails?
- [ ] Service YAML validates: `iagctl db import file.yaml --validate`
- [ ] Secrets created via `--prompt-value` (not raw values in committed YAML)
- [ ] Top-level `secrets:` section removed from `services.yaml` if using `--force` imports
- [ ] Service, decorator, repo names follow team naming convention

## Common Pitfalls

- **Read helper templates first** — `helpers/iag/` has examples for every service type
- **`clusterId` must match** the IAG cluster config — discover with `GET /gateway_manager/v1/gateways/`
- **`params` maps to decorator schema** — check with `iagctl run service <type> <name> --use`
- **`inventory` is `""` (empty string)** when not targeting nodes, not `[]` or `null`
- **OpenTofu `params` MUST include `"action": "apply"` or `"action": "destroy"`**
- **`runService` result is JSON-RPC wrapped** — extract with `query` path `result.stdout`, not `stdout`
- **`$var` doesn't resolve inside `newVariable` objects** — use separate `query` tasks instead
- **Secrets in YAML files contain raw values** — prefer `iagctl create secret --prompt-value` for sensitive data. Better yet, keep `secrets:` out of `services.yaml` entirely so `--force` never overwrites them.
- **Import is additive** — use `--force` to overwrite existing services
- **`--force` overwrites secrets too** — if your YAML has placeholder secrets, `--force` replaces real ones with placeholders
- **Decorators reject unknown params** — every `--set` key must exist in the decorator schema or IAG returns `extra input found`
- **Validate first** — always run `iagctl db import file.yaml --validate` before importing
