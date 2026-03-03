---
name: iag
description: Build and run IAG (Itential Automation Gateway) services — Python scripts, Ansible playbooks, OpenTofu plans. Manage repos, secrets, decorators. Call IAG services from Itential workflows.
argument-hint: "[action or service-name]"
---

# IAG — Itential Automation Gateway

IAG lets developers expose Python scripts, Ansible playbooks, and OpenTofu plans as REST APIs with centralized secrets, inventory, and dependency management.

**Two ways to use IAG:**
1. **`iagctl` CLI** — create repos, secrets, services, decorators. Run services from the command line.
2. **Itential workflows** — call IAG services from workflows using `GatewayManager.runService` task.

```
Developer builds:                    Itential workflows call:
  iagctl create repository ...         GatewayManager.runService
  iagctl create secret ...             GatewayManager.sendCommand
  iagctl create service ...            GatewayManager.sendConfig
  iagctl run service ...               GatewayManager.getServices
```

---

## Part 1: iagctl CLI

### Authentication

| Mode | Auth | How |
|------|------|-----|
| **Local** | None needed | `iagctl` talks to local IAG directly |
| **Server/Client** | Login required | `iagctl login <username>` → prompts for password → saves API key |
| **Itential workflows** | Pre-configured | Platform admin sets up the gateway connection. `clusterId` references it. |

**For server/client mode:**
```bash
iagctl login admin
# Prompts for password interactively, saves API key to api.key file
# Key expires after 24 hours by default — re-login if you get auth errors
```

**The agent cannot run `iagctl login`** — it requires an interactive terminal for the password prompt (no `--password` flag, no env var, no pipe). If the engineer hasn't logged in yet, tell them:
> "Run `iagctl login admin` in your terminal and enter your password. Once done, I can continue."

**Quick check — if this works, you're authenticated:**
```bash
iagctl get services
```
If it errors with auth issues, the engineer needs to run `iagctl login` manually first.

### Service Types

| Type | What it runs | Key flags |
|------|-------------|-----------|
| `python-script` | Python file from a Git repo | `--filename`, `--req-file` |
| `ansible-playbook` | Ansible playbook from a Git repo | `--playbook`, `--inventory`, `--extra-vars` |
| `opentofu-plan` | OpenTofu apply/destroy from a Git repo | `--var`, `--var-file` |

### End-to-End Workflow

```
1. Create secret(s)     → SSH keys, API tokens, passwords
2. Create repository    → Points to Git repo with your code
3. Create decorator     → JSON Schema defining service inputs (optional)
4. Create service       → Links type → repo → decorator → secrets
5. Run service          → Execute with inputs
```

### Check What Exists

```bash
iagctl get repositories
iagctl get secrets
iagctl get decorators
iagctl get services
iagctl get services --type python-script
```

### Create a Secret

```bash
# From string
iagctl create secret api-token --value "token-abc123"

# From file (SSH key)
iagctl create secret git-key --value @~/.ssh/id_rsa

# Interactive prompt (recommended for passwords — never in shell history)
iagctl create secret db-password --prompt-value

# With at-rest encryption
iagctl create secret api-key --value "abc123" --encryption-file /path/to/key
```

### Create a Repository

```bash
# Public repo
iagctl create repository my-repo \
  --url git@github.com:org/automations.git \
  --reference main

# Private repo (needs SSH key secret first)
iagctl create secret git-key --value @~/.ssh/id_rsa
iagctl create repository my-repo \
  --url git@github.com:org/private-repo.git \
  --private-key-name git-key

# Verify
iagctl describe repository my-repo
```

### Create a Decorator (Input Schema)

Decorators define the REST API interface — what inputs a service accepts. JSON Schema format.

```json
{
  "$id": "root",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["device", "interface"],
  "properties": {
    "device": {
      "type": "string",
      "description": "Target device hostname or IP"
    },
    "interface": {
      "type": "string",
      "description": "Interface name (e.g. GigabitEthernet0/0/0)"
    },
    "dry_run": {
      "type": "boolean",
      "default": false
    }
  }
}
```

```bash
# From file
iagctl create decorator my-decorator --schema @schema.json

# Inline
iagctl create decorator my-decorator --schema '{"$id":"root",...}'

# Verify
iagctl describe decorator my-decorator
```

### Create a Service

**Python script:**
```bash
iagctl create service python-script my-service \
  --repository my-repo \
  --filename main.py \
  --working-dir scripts/ \
  --req-file requirements.txt \
  --decorator my-decorator \
  --secret name=api-token,type=env,target=API_TOKEN \
  --description "Automates device configuration"
```

**Ansible playbook:**
```bash
iagctl create service ansible-playbook my-playbook \
  --repository my-repo \
  --playbook site.yml \
  --working-dir ansible/ \
  --inventory inventory.yml \
  --decorator my-decorator \
  --secret name=vault-key,type=env,target=ANSIBLE_VAULT_PASSWORD
```

**OpenTofu plan:**
```bash
iagctl create service opentofu-plan my-plan \
  --repository my-repo \
  --working-dir terraform/ \
  --var environment=production \
  --var-file prod.tfvars \
  --secret name=api-token,type=env,target=TF_VAR_TOKEN
```

### Run a Service

```bash
# See what inputs it expects
iagctl run service python-script my-service --use

# Run with inputs
iagctl run service python-script my-service \
  --set device=10.0.0.1 \
  --set interface=GigabitEthernet0/0/0

# Ansible
iagctl run service ansible-playbook my-playbook --set target_host=router1

# OpenTofu apply
iagctl run service opentofu-plan apply my-plan --set server_name=prod-host

# OpenTofu destroy
iagctl run service opentofu-plan destroy my-plan --state @opentofu.tfstate

# Raw JSON output
iagctl run service python-script my-service --raw
```

### Update a Service

IAG does not have an update command. Delete and recreate:
```bash
iagctl delete service my-service
iagctl create service python-script my-service ...
```

### Secret Injection

Secrets are injected as environment variables at runtime:
```
--secret name=<secret-name>,type=env,target=<ENV_VAR_NAME>
```

The `target` is the env var your script reads. Example: if your Python script does `os.environ['API_TOKEN']`, use `target=API_TOKEN`.

### Private Registry

For private PyPI or Ansible Galaxy:
```bash
iagctl create secret pypi-password --prompt-value
iagctl create registry pypi my-pypi \
  --url http://hostname:8080/simple \
  --username admin \
  --password-name pypi-password

# Then reference in service:
iagctl create service python-script my-service \
  --repository my-repo --filename main.py \
  --registry my-pypi
```

---

## Part 2: Calling IAG from Itential Workflows

### GatewayManager Tasks

| Task | What it does | Key inputs |
|------|-------------|------------|
| `runService` | Run an IAG service by name | `serviceName`, `clusterId`, `params`, `inventory` |
| `sendCommand` | Send CLI commands to devices | `clusterId`, `commands`, `inventory` |
| `sendConfig` | Send config text to devices | `clusterId`, `config`, `inventory` |
| `getServices` | List all IAG services | `queryParameters` (optional) |
| `getServiceById` | Get service details | `id` |
| `getGateways` | List connected gateways | (none) |

### runService — Run an IAG Service from a Workflow

This is the primary task for calling IAG services from Itential workflows.

**Task wiring:**
```json
{
  "name": "runService",
  "canvasName": "runService",
  "summary": "Run IAG Service",
  "description": "Run an IAG service by name",
  "location": "Application",
  "locationType": null,
  "app": "GatewayManager",
  "type": "automatic",
  "displayName": "GatewayManager",
  "variables": {
    "incoming": {
      "serviceName": "hello-python",
      "clusterId": "ankitcluster",
      "params": {},
      "inventory": ""
    },
    "outgoing": {
      "result": ""
    },
    "decorators": []
  },
  "actor": "Pronghorn",
  "groups": [],
  "nodeLocation": {"x": 600, "y": 1308}
}
```

**Incoming variables:**

| Variable | Type | Description |
|----------|------|-------------|
| `serviceName` | string | Name of the IAG service (e.g., `"hello-python"`) |
| `clusterId` | string | Gateway cluster ID (e.g., `"ankitcluster"`) |
| `params` | object | Key/value inputs for the service. For OpenTofu, must include `"action": "apply"` or `"destroy"` |
| `inventory` | array or `""` | Target nodes. Array of `{"inventory": "inv-name", "nodeNames": ["node1"]}`. Use `""` if not targeting specific nodes. |

**Outgoing variables:**

| Variable | Type | Description |
|----------|------|-------------|
| `result` | object | JSON-RPC envelope containing the service execution result |

**Result shape — `runService` returns a JSON-RPC wrapper, NOT raw stdout:**
```json
{
  "id": "dc7c4a5d-8ba7-47e2-acf9-cd020fb931ee",
  "jsonrpc": "2.0",
  "result": {
    "return_code": 0,
    "stdout": "{ ... script output ... }",
    "stderr": "",
    "start_time": "2026-03-03T19:26:37.178124Z",
    "end_time": "2026-03-03T19:26:37.837354Z",
    "elapsed_time": 0.659
  },
  "receiveTime": 1772565997870,
  "status": "completed"
}
```

**To extract stdout in a workflow, use a `query` task with `"query": "result.stdout"` — NOT `"query": "stdout"`.**
The `result` key in the envelope is nested: `iagResult.result.stdout`.

**Dynamic inputs from workflow variables:**
```json
{
  "incoming": {
    "serviceName": "$var.job.iagServiceName",
    "clusterId": "$var.job.clusterId",
    "params": "$var.job.serviceParams",
    "inventory": ""
  },
  "outgoing": {
    "result": "$var.job.iagResult"
  }
}
```

### sendCommand — Send CLI Commands via Gateway

```json
{
  "name": "sendCommand",
  "app": "GatewayManager",
  "type": "automatic",
  "location": "Application",
  "displayName": "GatewayManager",
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

### sendConfig — Send Config Text via Gateway

```json
{
  "name": "sendConfig",
  "app": "GatewayManager",
  "type": "automatic",
  "location": "Application",
  "displayName": "GatewayManager",
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

### Finding Your Cluster ID

The `clusterId` is required for all GatewayManager tasks. To find it:
```bash
# From iagctl
iagctl get clusters

# From Itential workflow — use getGateways task, or check platform UI
```

The cluster ID comes from IAG configuration. Ask the engineer if unsure.

### Workflow Pattern: Run IAG Service with Error Handling

```
workflow_start
    → runService (call IAG)
        → success → query (extract result)
            → process result → workflow_end
        → error → newVariable (set errorStatus)
            → workflow_end
```

```json
{
  "tasks": {
    "a100": {
      "name": "runService",
      "app": "GatewayManager",
      "type": "automatic",
      "actor": "Pronghorn",
      "variables": {
        "incoming": {
          "serviceName": "$var.job.serviceName",
          "clusterId": "$var.job.clusterId",
          "params": "$var.job.params",
          "inventory": ""
        },
        "outgoing": {
          "result": "$var.job.iagResult"
        }
      }
    },
    "a200": {
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
  },
  "transitions": {
    "workflow_start": {"a100": {"type": "standard", "state": "success"}},
    "a100": {
      "a200": {"type": "standard", "state": "success"},
      "error_handler": {"type": "standard", "state": "error"}
    }
  }
}
```

---

## Part 3: Bridging iagctl and Workflows

**Build with iagctl, call from workflows:**

1. **iagctl** — create repos, secrets, services, decorators (the development side)
2. **Itential workflows** — call those services via `GatewayManager.runService` (the orchestration side)

The typical flow:
```
Developer:
  iagctl create repository ...
  iagctl create secret ...
  iagctl create service python-script configure-device ...
  iagctl run service python-script configure-device --use    ← test inputs
  iagctl run service python-script configure-device --set device=10.0.0.1  ← test run

Then in Itential:
  Workflow uses GatewayManager.runService with:
    serviceName = "configure-device"
    clusterId = "mycluster"
    params = {"device": "10.0.0.1"}
```

**The service name in iagctl = the serviceName in the workflow task.** They're the same thing — iagctl creates it, the workflow calls it.

### When to Use Which

| Need | Use |
|------|-----|
| Run a Python script / Ansible playbook / OpenTofu plan | `GatewayManager.runService` |
| Send ad-hoc CLI commands to devices (show commands) | `GatewayManager.sendCommand` or `AGManager.itential_cli` |
| Push config text to devices | `GatewayManager.sendConfig` or `AGManager.itential_set_config` |
| Run MOP validation checks | `MOP.RunCommandTemplate` (separate from IAG) |

### AGManager vs GatewayManager

Both talk to the Automation Gateway, but differently:

| | AGManager | GatewayManager |
|---|-----------|---------------|
| **Task source** | Individual scripts/playbooks registered on IAG | Named services created with iagctl |
| **How tasks appear** | One task per script/playbook (e.g., `itential_cli`, `sample_ios_run_command`) | Generic tasks (`runService`, `sendCommand`, `sendConfig`) |
| **Input style** | Task-specific variables (e.g., `_hosts`, `command`) | `serviceName` + `params` object |
| **When to use** | Built-in IAG capabilities, quick CLI commands | Custom services the developer built with iagctl |

---

## Quick Reference

```bash
# List everything
iagctl get repositories
iagctl get secrets
iagctl get decorators
iagctl get services
iagctl get services --type python-script

# Inspect
iagctl describe repository <name>
iagctl describe secret <name>
iagctl describe decorator <name>
iagctl describe service <name>

# Run
iagctl run service python-script <name> --use           # show inputs
iagctl run service python-script <name> --set key=value  # run
iagctl run service ansible-playbook <name> --set key=val
iagctl run service opentofu-plan apply <name> --set key=val

# Delete
iagctl delete service <name>
iagctl delete repository <name>
iagctl delete secret <name>
iagctl delete decorator <name>
```

## Common Pitfalls

- `iagctl` CLI path: the binary is at `/path/to/iagctl-darwin-arm64` (or platform-specific). Make sure it's accessible.
- `clusterId` in workflows must match the IAG cluster configuration — ask the engineer
- `params` in `runService` maps to the service's decorator schema — check with `iagctl run service <type> <name> --use`
- `inventory` is `""` (empty string) when not targeting specific nodes, not `[]` or `null`
- OpenTofu `params` MUST include `"action": "apply"` or `"action": "destroy"`
- Secret `target` in `--secret name=x,type=env,target=VAR` is the env var name your code reads
- JSON arrays in `--set` need single quotes: `--set 'commands=["cmd1","cmd2"]'`
- **`$var` references don't resolve inside `newVariable` value objects.** If you use `newVariable` to consolidate outputs like `{"deviceInfo": "$var.job.deviceInfoStdout", ...}`, the `$var` references stay as literal strings. Instead, use separate `query` tasks to extract and store each value individually, or use `transformation` tasks.
- **`query` path for runService output is `result.stdout`**, not `stdout` — because the result is wrapped in a JSON-RPC envelope (`{jsonrpc, id, result: {stdout, stderr, return_code, ...}}`)
