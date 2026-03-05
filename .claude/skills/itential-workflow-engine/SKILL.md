---
name: itential-workflow-engine
description: Run and debug Itential workflows, wire utility tasks (query, merge, evaluation, childJob, forEach), understand $var resolution, and apply workflow patterns. Use when running jobs, debugging failures, or building task logic inside workflows.
argument-hint: "[action or task-name]"
---

# WorkFlow Engine - Developer Skills Guide

WorkFlowEngine is the runtime engine for Itential workflows. It executes jobs, provides 90+ built-in utility tasks for data manipulation and control flow, and handles variable resolution between tasks. Use this skill when running workflows, wiring tasks, debugging job failures, or using utility tasks like query, merge, evaluation, or childJob.

## Concepts

- **Jobs** are running instances of workflows. You start a job by posting to `/operations-manager/jobs/start` with a workflow name and input variables.
- **Tasks** are individual steps in a workflow. Each task has incoming variables (inputs) and outgoing variables (outputs). Tasks are connected by transitions.
- **`$var`** is the runtime variable reference syntax. `$var.job.x` references a job variable; `$var.taskId.outVar` references a previous task's output.
- **WorkFlowEngine provides built-in utility tasks** that require no adapter -- query, merge, evaluation, childJob, forEach, newVariable, push, pop, shift, delay, transformation, decision, and 60+ more for string, array, object, and time operations.
- **Core tasks** (documented in detail below) handle control flow, data extraction, conditional branching, and child workflow orchestration.
- **Helper tasks** (string, array, object, time) are discoverable from `tasks.json` -- see the Additional Utility Tasks section.

## $var Resolution Rules (Source-Code Verified)

Task IDs are generated as **hex strings** (`[0-9a-f]{1,4}`, range `0`-`ffff`). The engine validates `$var` references against this regex:

```
taskIdRegex = /^([0-9a-f]{1,4}|workflow_start|workflow_end)$/
```

**If a task ID contains non-hex characters, `$var` references to it silently fail** -- the string is classified as `type: "static"` and stored literally, never resolved at runtime.

| $var Pattern | Resolves? | Why |
|---|---|---|
| `$var.job.deviceName` | Yes | `job` is a recognized keyword |
| `$var.a1b2.result` | Yes | `a1b2` is valid hex |
| `$var.ff09.return_data` | Yes | `ff09` is valid hex |
| `$var.apush.result` | **NO** | `p`, `u`, `s`, `h` are not hex chars |
| `$var.myTask.output` | **NO** | `m`, `y`, `T`, `k` are not hex chars |

**$var only resolves at the top level of `incoming` variables.** The engine iterates `Object.values(incoming)` and only resolves direct string values. It does NOT recurse into nested objects:

| Wiring | Works? | Why |
|---|---|---|
| `"deviceName": "$var.job.x"` | Yes | Direct top-level string value |
| `"variables": {"key": "$var.job.x"}` | **NO** | Nested inside an object -- stored as literal string |
| `"body": {"data": "$var.job.x"}` | **NO** | Same -- nested object, never resolved |

**Workaround for nested objects:** Use a `merge` or `query` intermediate task (with a hex ID) to build the object, then reference that task's output with `$var.taskId.merged_object`.

## Starting and Monitoring Jobs

### Test Individual Tasks

Some tasks have standalone REST endpoints (e.g., `POST /workflow_engine/query`), but many tasks (array ops, string ops, etc.) only work inside a running workflow. The reliable way to test any task:

1. **Get the schema first** -- check `{use-case}/task-schemas.json` locally, or call `multipleTaskDetails?dereferenceSchemas=true` if not cached
2. **Create a minimal test workflow** -- `start -> task -> end` with the task's exact incoming/outgoing variable names from the schema
3. **Start the job** -- `POST /operations-manager/jobs/start`
4. **Check the result** -- `GET /operations-manager/jobs/{jobId}` and inspect the `variables` object

### Start a Workflow (Run a Job)

```
POST /operations-manager/jobs/start
```
```json
{
  "workflow": "Test Array Concat",
  "options": {
    "description": "Testing arrayConcat task",
    "type": "automation",
    "variables": {
      "arr": ["IOS-CAT8KV-1", "IOS-CAT8KV-2"],
      "arrayN": ["IOS-CSR-AWS-1"]
    }
  }
}
```
- `workflow` -- the workflow name (string, not ID)
- `options.variables` -- input values that map to the workflow's `inputSchema`
- `options.type` -- `"automation"`
- `options.description` -- optional job description

**Response:**
```json
{
  "message": "Successfully started job",
  "data": {
    "_id": "da97dcf248b942a089fe7dc4",
    "status": "running"
  }
}
```

### Check Job Status and Results

```
GET /operations-manager/jobs/{id}
```

Response is wrapped in `{message, data, metadata}`. The job object is inside `data`:
- `data.status` -- `"running"`, `"complete"`, `"error"`, `"canceled"`
- `data.variables` -- all job variables including outputs mapped via `$var.job.x`
- `data.tasks` -- each task with `status` and `metrics.finish_state`

**Example result:**
```json
{
  "message": "Successfully retrieved job",
  "data": {
    "status": "complete",
    "variables": {
      "arr": ["IOS-CAT8KV-1", "IOS-CAT8KV-2"],
      "arrayN": ["IOS-CSR-AWS-1"],
      "result": ["IOS-CAT8KV-1", "IOS-CAT8KV-2", "IOS-CSR-AWS-1"]
    }
  },
  "metadata": {}
}
```

**Check errors on failed jobs** (error array is inside `data`):
```json
{
  "data": {
    "status": "error",
    "error": [
      {
        "task": "371e",
        "message": {
          "icode": "AD.312",
          "IAPerror": {
            "displayString": "Schema validation failed on must have required property 'summary'",
            "recommendation": "Verify the information provided is in the correct format"
          }
        }
      }
    ]
  }
}
```

### Task Endpoint Patterns

Some tasks have standalone REST endpoints you can call directly for quick testing -- **faster and cheaper than creating test workflows**:
- **WorkFlowEngine:** `POST /workflow_engine/{method}` (e.g., `/workflow_engine/query`) -- requires `job_id` parameter (can use a dummy ObjectId like `"4321abcdef694aa79dae47ad"`)
- **ConfigurationManager:** `POST /configuration_manager/{route}` (e.g., `/configuration_manager/devices`)
- **MOP (command templates):** `POST /mop/RunCommandTemplate` with `{"template":"name","devices":["dev"],"variables":{...}}` -- test command templates directly without a workflow
- **TemplateBuilder (render):** `POST /template_builder/templates/{name}/renderJinja` with `{"context":{...}}` -- test Jinja2 template rendering directly. Note: the REST API uses `context` as the parameter name, not `variables`.

Most WorkFlowEngine utility tasks (array ops, string ops, forEach, childJob, merge, etc.) do **NOT** have standalone endpoints. The reliable way to test those is to create a minimal workflow and run it via `jobs/start`.

## Core Utility Tasks

### query

**Purpose:** Extract nested values from objects using json-query syntax.

**Incoming variables:**
- `pass_on_null` (boolean) -- Determines behavior when query returns null. `true` = success transition with null; `false` = failure transition.
- `query` (string) -- The json-query expression (dot/bracket notation). Example: `"name"`, `"devices[0].hostname"`, `"[**].result"`.
- `obj` (object) -- Data to query. Usually a `$var` reference like `$var.a1b2.job_details`.

**Outgoing variables:**
- `return_data` (any type) -- The extracted value.

**Standalone endpoint:** `POST /workflow_engine/query` (requires `job_id`).

**Example wiring in a workflow task:**
```json
{
  "id": "c3d4",
  "name": "query",
  "canvasName": "query",
  "summary": "Extract Device Name",
  "app": "WorkFlowEngine",
  "type": "operation",
  "displayName": "WorkFlowEngine",
  "variables": {
    "incoming": {
      "pass_on_null": false,
      "query": "hostname",
      "obj": "$var.a1b2.return_data"
    },
    "outgoing": {
      "return_data": "$var.job.deviceName"
    }
  },
  "actor": "Pronghorn",
  "groups": []
}
```

**Note:** `pass_on_null` controls whether a null result follows the success or failure transition. Set to `false` when you need to detect missing data. Set to `true` when null is an acceptable result.

### evaluation

**Purpose:** Conditional branching based on data comparisons. This is the only way to do if/else logic in workflows.

**MUST have BOTH success AND failure transitions.** If you only have a `success` transition and the condition is `false`, the **job will error out**.

**Incoming variables:**
- `all_true_flag` (boolean) -- Whether ALL evaluation groups must pass (`true`) or ANY group passing is sufficient (`false`).
- `evaluation_groups` (array) -- Array of evaluation group objects, each containing `evaluations` and `all_true_flag`.

**Outgoing variables:**
- `return_value` (boolean) -- Result of the evaluation.

**Transitions:**
- `success` transition -- condition evaluated to `true`
- `failure` transition -- condition evaluated to `false`

**Example wiring:**
```json
{
  "id": "a120",
  "name": "evaluation",
  "canvasName": "evaluation",
  "summary": "Check Status",
  "app": "WorkFlowEngine",
  "type": "operation",
  "displayName": "WorkFlowEngine",
  "variables": {
    "incoming": {
      "all_true_flag": true,
      "evaluation_groups": [
        {
          "all_true_flag": true,
          "evaluations": [
            {
              "operand_1": {"variable": "createStatus", "task": "job"},
              "operator": "==",
              "operand_2": {"variable": "success", "task": "static"}
            }
          ]
        }
      ]
    },
    "outgoing": {
      "return_value": null
    }
  },
  "actor": "Pronghorn",
  "groups": []
}
```

**Evaluation operand reference format:**
- `{"task": "job", "variable": "varName"}` -- reference a job variable
- `{"task": "static", "variable": "literalValue"}` -- a literal/static value
- `{"task": "taskId", "variable": "outVar"}` -- reference a task's output

**Standalone endpoints:** `POST /workflow_engine/runEvaluationGroup` and `POST /workflow_engine/runEvaluationGroups` for testing evaluations outside a workflow.

### merge

**Purpose:** Build an object from multiple resolved values. This is the primary workaround for `$var` not resolving inside nested objects.

**Incoming variables:**
- `data_to_merge` (array, min 2 items) -- Array of `{key, value}` objects. Each `value` is a reference object with `task` and `variable` fields.

**Outgoing variables:**
- `merged_object` (object) -- The assembled object with all keys populated.

**IMPORTANT: The field is `"variable"` NOT `"value"`** in the reference objects inside `data_to_merge`. This is a common mistake.

**Reference object format in `data_to_merge`:**
- `{"task": "job", "variable": "varName"}` -- pull from a job variable
- `{"task": "static", "variable": "literalValue"}` -- use a literal value
- `{"task": "taskId", "variable": "outVar"}` -- pull from a previous task's output

**Example: Building a request body from job variables:**
```json
{
  "id": "b2c3",
  "name": "merge",
  "canvasName": "merge",
  "summary": "Build Request Body",
  "app": "WorkFlowEngine",
  "type": "operation",
  "displayName": "WorkFlowEngine",
  "variables": {
    "incoming": {
      "data_to_merge": [
        {"key": "hostname", "value": {"task": "static", "variable": "IOS-CAT8KV-1"}},
        {"key": "details", "value": {"task": "job", "variable": "deviceInfo"}},
        {"key": "config", "value": {"task": "a1b2", "variable": "renderedTemplate"}}
      ]
    },
    "outgoing": {
      "merged_object": "$var.job.requestBody"
    }
  },
  "actor": "Pronghorn",
  "groups": []
}
```

**Result:** `merged_object` = `{"hostname": "IOS-CAT8KV-1", "details": <value of job.deviceInfo>, "config": <value of task a1b2 renderedTemplate>}`

### deepmerge

**Purpose:** Deep merge data using extend -- merges nested objects recursively rather than overwriting top-level keys.

**Incoming variables:**
- `data_to_merge` (array, min 2 items) -- Same format as `merge`: array of `{key, value}` objects where `value` has `task` and `variable`.

**Outgoing variables:**
- `merged_object` (object) -- The deep-merged result.

**When to use deepmerge vs merge:** Use `merge` when building flat objects from disparate sources. Use `deepmerge` when combining objects that share nested keys and you want nested properties merged rather than overwritten.

### newVariable

**Purpose:** Create or set a job variable at runtime.

**Incoming variables:**
- `name` (string) -- The variable name to create/set.
- `value` (any type: array, string, boolean, integer, number, object) -- The value to assign.

**Outgoing variables:**
- `value` (any type) -- The value that was set.

**Example wiring:**
```json
{
  "id": "a1b2",
  "name": "newVariable",
  "canvasName": "newVariable",
  "summary": "Set Status Flag",
  "app": "WorkFlowEngine",
  "type": "operation",
  "displayName": "WorkFlowEngine",
  "variables": {
    "incoming": {
      "name": "taskStatus",
      "value": "success"
    },
    "outgoing": {
      "value": "$var.job.taskStatus"
    }
  },
  "actor": "Pronghorn",
  "groups": []
}
```

**GOTCHA: `$var` inside the `value` field does NOT resolve.** If you set `"value": "$var.job.someVar"`, the literal string `"$var.job.someVar"` is stored, not the resolved value. To set a job variable to the resolved value of another variable, use a `merge` task to build the object first, then a `query` task to extract it, then `newVariable` with the extracted value wired via `$var.taskId.return_data`.

### transformation

**Purpose:** Perform JSON transformation using the JST (JSON Schema Transformation) library.

**Incoming variables:**
- `tr_id` (string) -- The transformation ID (MongoDB ObjectId).
- `variableMap` (object) -- A map between the transformation's incoming schemas and their data locations in the job.
- `options` (object, optional) -- Options like `{"extractOutput": true}`.

**Outgoing variables:**
- `outgoing` (any type) -- The transformed data.

**Example wiring:**
```json
{
  "id": "d4e5",
  "name": "transformation",
  "canvasName": "transformation",
  "summary": "Transform Device Data",
  "app": "WorkFlowEngine",
  "type": "operation",
  "displayName": "WorkFlowEngine",
  "variables": {
    "incoming": {
      "tr_id": "62955855e2dff7146fb1c269",
      "variableMap": {
        "deviceList": "$var.a1b2.return_data"
      }
    },
    "outgoing": {
      "outgoing": "$var.job.transformedData"
    }
  },
  "actor": "Pronghorn",
  "groups": []
}
```

### childJob

The `childJob` task calls another workflow as a sub-job. **Use the helper template** `helpers/workflow-task-childjob.json` as your starting point.

**Complete childJob task template (copy-paste ready):**
```json
{
  "name": "childJob",
  "canvasName": "childJob",
  "summary": "Run Child Job",
  "description": "Runs a child job inside a workflow.",
  "location": "Application",
  "locationType": null,
  "app": "WorkFlowEngine",
  "type": "operation",
  "displayName": "WorkFlowEngine",
  "variables": {
    "incoming": {
      "task": "",
      "workflow": "Child Workflow Name",
      "variables": {},
      "data_array": "",
      "transformation": "",
      "loopType": ""
    },
    "outgoing": {
      "job_details": null
    },
    "decorators": []
  },
  "groups": [],
  "actor": "job",
  "nodeLocation": { "x": 600, "y": 600 }
}
```

**Critical fields that differ from normal tasks:**
- **`actor` MUST be `"job"`** -- not `"Pronghorn"` (which is used for all other tasks)
- **`task` MUST be `""`** (empty string) -- the engine auto-sets this to the task ID at runtime
- **`outgoing.job_details` MUST be `null`** (or `""`) -- do NOT override with `$var.job.X` or it silently breaks
- **All incoming fields are required** -- even unused ones must be present as empty strings: `"data_array": ""`, `"transformation": ""`, `"loopType": ""`
- **`loopType`** values: `""` (no loop), `"parallel"`, or `"sequential"`

#### Mode 1: No Loop -- Single Child Job

Run a single child job with variables passed from the parent. **Variables use `{"task", "value"}` syntax — NOT `$var`.**
Using `$var.job.x` inside the `variables` object causes the childJob to hang indefinitely (confirmed on live platform):
```json
{
  "incoming": {
    "task": "",
    "workflow": "My Child Workflow",
    "variables": {
      "deviceName": {"task": "job", "value": "deviceName"},
      "configData": {"task": "a1b2", "value": "return_data"}
    },
    "data_array": "",
    "transformation": "",
    "loopType": ""
  },
  "outgoing": {"job_details": null}
}
```

#### Mode 2: Loop Parallel -- Multiple Simultaneous Child Jobs

Run multiple child jobs simultaneously, one per item in `data_array`:
```json
{
  "incoming": {
    "task": "",
    "workflow": "My Child Workflow",
    "variables": {},
    "data_array": "$var.6254.return_data",
    "transformation": "",
    "loopType": "parallel"
  },
  "outgoing": {"job_details": null}
}
```

#### Mode 3: Loop with Transformation

Transform each `data_array` element before passing to the child:
```json
{
  "incoming": {
    "task": "",
    "workflow": "My Child Workflow",
    "variables": {},
    "data_array": "$var.6254.return_data",
    "transformation": "62955855e2dff7146fb1c269",
    "loopType": "parallel"
  },
  "outgoing": {"job_details": null}
}
```

#### childJob Variable Passing

- `{"task": "static", "value": [...]}` -- literal value passed directly to child
- `{"task": "job", "value": "varName"}` -- parent job variable (must exist at start time)
- `{"task": "taskId", "value": "outVar"}` -- previous task's output (preferred for runtime-produced data)
- When using `data_array`, each object in the array becomes the child job's variables for that iteration. The `variables` field should be `{}`.
- **Prefer `{"task": "taskId"}` over `{"task": "job"}` for runtime data** -- job variables referenced with `{"task": "job"}` must exist when the job starts. Task references resolve at execution time.
- **Never use `{"task": "static", "value": ["placeholder"]}` as a stand-in** -- the literal `["placeholder"]` persists at runtime. Use `{"task": "job", "value": "varName"}` instead.
- The engine auto-injects `childJobLoopIndex` into each loop iteration's variables.

#### Dynamic data_array with newVariable

Use `newVariable` to build the child loop data at runtime, then reference it with `$var.job.varName`:
```json
{
  "tasks": {
    "a1b2": {
      "name": "newVariable",
      "variables": {
        "incoming": {
          "name": "myList",
          "value": [
            {"arr": ["alpha", "beta"], "arrayN": ["gamma"]},
            {"arr": ["one"], "arrayN": ["two", "three"]}
          ]
        },
        "outgoing": {"value": "$var.job.myList"}
      }
    },
    "c3d4": {
      "name": "childJob",
      "variables": {
        "incoming": {
          "task": "",
          "workflow": "Test Array Concat",
          "variables": {},
          "data_array": "$var.job.myList",
          "transformation": "",
          "loopType": "sequential"
        },
        "outgoing": {"job_details": null}
      },
      "actor": "job"
    }
  }
}
```

#### childJob Output (`job_details`) -- CRITICAL

**`job_details` is often `null` or empty** on the live platform. It only gets populated if the child workflow explicitly writes to `job_output`. Most workflows don't do this — child results stay in the child job's `variables` object.

When `job_details` IS populated (via `job_output`), it contains the child's outputSchema variables as a flat object — NOT the full job document. Query paths use the variable name directly.

**No loop** -- flat spread of child's output variables:
```json
{
  "status": "complete",
  "arr": ["hello", "world"],
  "arrayN": ["foo", "bar"],
  "result": ["hello", "world", "foo", "bar"]
}
```

**With loop** -- `status` + `loop` array, each entry is a flat spread of that iteration's variables:
```json
{
  "status": "complete",
  "loop": [
    {"status": "complete", "childJobLoopIndex": 0, "result": ["device1", "device2", "device3"]},
    {"status": "complete", "childJobLoopIndex": 1, "result": ["switch1", "switch2", "switch3"]},
    {"status": "complete", "childJobLoopIndex": 2, "result": ["router1", "router2", "router3", "router4"]}
  ]
}
```

#### Querying childJob Output

Use flat variable names, NOT nested paths:
```json
{
  "name": "query",
  "variables": {
    "incoming": {
      "query": "validateStatus",
      "obj": "$var.f48f.job_details",
      "pass_on_null": false
    }
  }
}
```
The query path is `"validateStatus"`, NOT `"variables.job.validateStatus"`. For loop output, use `"[**].healthCheckArray"` to extract a field from all loop iterations.

### forEach

The `forEach` task iterates over an array. Each iteration runs the loop body tasks. **Note: `forEach` is deprecated** -- prefer `childJob` with `loopType` for new workflows. However, `forEach` is still widely used in existing workflows.

**Incoming variables:**
- `data_array` (array) -- The array to iterate over.

**Outgoing variables:**
- `current_item` (any type) -- Set to the current array element each iteration.

**Transition pattern (critical):**
```
forEach --state:loop--> firstLoopBodyTask -> ... -> lastLoopBodyTask --(empty {})
forEach --state:success--> nextTaskAfterLoop
```

- `forEach` has TWO outgoing transitions: `loop` (into the body) and `success` (after loop completes)
- The LAST task in the loop body has an **empty transition `{}`** -- forEach handles the looping automatically
- The processing task does NOT connect back to forEach

**Example transitions:**
```json
{
  "transitions": {
    "workflow_start": {"a1b2": {"type": "standard", "state": "success"}},
    "a1b2": {
      "c3d4": {"type": "standard", "state": "loop"},
      "workflow_end": {"type": "standard", "state": "success"}
    },
    "c3d4": {},
    "workflow_end": {}
  }
}
```

**forEach behavior:**
- `current_item` is set to the current array element each iteration
- Job variable mapped from `current_item` gets overwritten each iteration (only last value remains after loop)
- To accumulate results, use `push` (or `arrayPush`) inside the loop body

### decision

**Purpose:** Multi-way branching based on conditions. Unlike `evaluation` (binary true/false), `decision` can branch to different tasks based on multiple conditions.

**Incoming variables:**
- `decisionArray` (array) -- Array of decision objects, each specifying conditions and a target task ID.

**Outgoing variables:**
- `return_value` (string) -- The ID of the next task to process.

### delay

**Purpose:** Pause workflow execution for a specified duration.

**Incoming variables:**
- `time` (integer) -- Time in seconds to delay. Minimum: 1.

**Outgoing variables:**
- `time_in_milliseconds` (integer) -- The delayed time in milliseconds.

**Example wiring:**
```json
{
  "id": "d1e2",
  "name": "delay",
  "canvasName": "delay",
  "summary": "Wait 60 Seconds",
  "app": "WorkFlowEngine",
  "type": "operation",
  "displayName": "WorkFlowEngine",
  "variables": {
    "incoming": {
      "time": 60
    },
    "outgoing": {
      "time_in_milliseconds": null
    }
  },
  "actor": "Pronghorn",
  "groups": []
}
```

### push / pop / shift

**Purpose:** Array manipulation on job variables. These tasks operate on job variables **by name** (a string), not by `$var` reference.

#### push -- Add item to end of array
**Incoming:**
- `job_variable` (string) -- The **name** of the job variable (e.g., `"myArray"`). Created if it does not exist.
- `item_to_push` (string) -- The item to add.

**Outgoing:**
- `job_variable_value` (array) -- The updated array after push.

#### pop -- Remove item from end of array
**Incoming:**
- `job_variable` (string) -- The **name** of the job variable.

**Outgoing:**
- `popped_item` (any type) -- The removed item (last element).

#### shift -- Remove item from beginning of array
**Incoming:**
- `job_variable` (string) -- The **name** of the job variable.

**Outgoing:**
- `shifted_item` (any type) -- The removed item (first element).

**GOTCHA:** These tasks take the variable **name as a plain string** (e.g., `"myResults"`), NOT a `$var` reference. If you pass `"$var.job.myResults"`, it will look for a job variable literally named `"$var.job.myResults"`.

**Example -- push inside a forEach loop to accumulate results:**
```json
{
  "id": "e5f6",
  "name": "push",
  "canvasName": "push",
  "summary": "Accumulate Results",
  "app": "WorkFlowEngine",
  "type": "operation",
  "displayName": "WorkFlowEngine",
  "variables": {
    "incoming": {
      "job_variable": "collectedResults",
      "item_to_push": "$var.c3d4.return_data"
    },
    "outgoing": {
      "job_variable_value": null
    }
  },
  "actor": "Pronghorn",
  "groups": []
}
```

### updateJobDescription

**Purpose:** Overwrite the job description at runtime. Useful for adding context as the job progresses.

**Incoming variables:**
- `description` (string) -- The new job description.

**Outgoing variables:**
- `description` (string) -- The value of the new job description.

### eventListenerJob

**Purpose:** Pause the workflow until an event matching a topic and schema is received. Used for event-driven automation.

**Incoming variables:**
- `source` (string) -- The source that provides the topic (e.g., `"someSourceName"`).
- `topic` (string) -- The event topic to listen for (e.g., `"someTopicName"`).
- `schema` (object) -- JSON schema that uniquely identifies the event.

**Outgoing variables:**
- `result` (object) -- The payload of the captured event.

### modify

**Purpose:** Modify data by optionally querying into an object and replacing with a new value.

**Incoming variables:**
- `object_to_update` (any type) -- The data to modify.
- `query` (string, optional) -- Query path into the object (uses json-query syntax).
- `new_value` (any type) -- The replacement value.

**Outgoing variables:**
- `updated_object` (any type) -- The modified data.

### validateJsonSchema

**Purpose:** Validate JSON data against a JSON schema.

**Incoming variables:**
- `jsonData` (object, required) -- The JSON data to validate.
- `schema` (object, required) -- The JSON schema to validate against.

**Outgoing variables:**
- `result` (object) -- Contains `{"valid": true}` or `{"valid": false}`.

**Standalone endpoint:** `POST /workflow_engine/validateJsonSchema`.

### makeData

**Purpose:** Construct data objects with variable substitution using `<!var!>` syntax.

**Incoming variables:**
- `input` (string) -- Template string with `<!variable_name!>` placeholders
- `outputType` (string) -- Output type: `"string"`, `"json"`, `"number"`, `"boolean"`
- `variables` (object) -- Object containing the values to substitute. Key names must match `<!var!>` names exactly.

**Outgoing variables:**
- `output` -- The result with all `<!var!>` placeholders replaced

**The `variables` field must be a resolved object, not inline `$var` references.** Since `$var` doesn't resolve inside nested objects, you can't put `$var.job.x` as values inside the `variables` object. Instead, use a `merge` task first to build the variables object, then pass it via a top-level `$var` reference:

```
merge (build variables object) → makeData (use $var.taskId.merged_object as variables)
```

**Working pattern:**
```json
{
  "a1b2": {
    "name": "merge",
    "variables": {
      "incoming": {
        "data_to_merge": [
          {"key": "deviceLabel", "value": {"task": "job", "variable": "deviceLabel"}},
          {"key": "vlanId", "value": {"task": "job", "variable": "vlanId"}}
        ]
      },
      "outgoing": {"merged_object": null}
    },
    "actor": "Pronghorn"
  },
  "c3d4": {
    "name": "makeData",
    "variables": {
      "incoming": {
        "input": "REPORT: <!deviceLabel!> | VLAN: <!vlanId!>",
        "outputType": "string",
        "variables": "$var.a1b2.merged_object"
      },
      "outgoing": {"output": "$var.job.result"}
    },
    "actor": "Pronghorn"
  }
}
```

### restCall

**Purpose:** Make external HTTP calls from within a workflow. Use when you need to call APIs that are not exposed through adapters.

## Additional Utility Tasks

WorkFlowEngine provides 60+ additional helper tasks for string, array, object, number, and time operations. These are available in every workflow without requiring adapters.

**To discover them:** Search `tasks.json` by app `WorkFlowEngine`:
```bash
jq '.[] | select(.app == "WorkFlowEngine") | {name, summary}' {use-case}/tasks.json
```

**To get full schemas:** Use `multipleTaskDetails` with `dereferenceSchemas=true`:
```
POST /automation-studio/multipleTaskDetails?dereferenceSchemas=true
```

| Category | Count | Examples |
|----------|-------|---------|
| String | 31 | `stringConcat`, `replace`, `split`, `toLowerCase`, `toUpperCase`, `trim`, `substring` |
| Array | 19 | `arrayConcat`, `arrayPush`, `sort`, `join`, `arraySlice`, `map`, `reverse` |
| Object | 6 | `assign`, `keys`, `values`, `objectHasOwnProperty`, `setObjectKey` |
| Time | 8 | `getTime`, `addDuration`, `convertTimezone`, `calculateTimeDiff`, `convertTimeFormat` |
| Number | 1 | `numberToString` |
| Tools | 9 | `restCall`, `makeData`, `csvStringToJson`, `excelToJson`, `asciiToBase64` |

## Value Reference Patterns

Different tasks use different patterns for referencing values. There are two systems:

**`$var` references** (used by most tasks):
- `$var.job.varName` -- reference a job variable
- `$var.taskId.outgoingVar` -- reference a previous task's output

**`task`/`value` objects** (used by `childJob` variables):
- `{"task": "static", "value": "literal value"}` -- pass a static/literal value
- `{"task": "job", "value": "varName"}` -- reference a parent job variable
- `{"task": "taskId", "value": "outVar"}` -- reference a previous task's output

**`task`/`variable` objects** (used by `merge` and `evaluation`):
- `{"task": "static", "variable": "literal value"}` -- a literal value
- `{"task": "job", "variable": "varName"}` -- reference a job variable
- `{"task": "taskId", "variable": "outVar"}` -- reference a task's output

**IMPORTANT:** `merge` uses `"variable"`, `childJob` uses `"value"`. Do not mix them up.

**Variable syntax comparison table:**

| Context | Syntax | Example |
|---------|--------|---------|
| Jinja2 templates | `{{ var }}` | `interface Vlan{{ vlan_id }}` |
| Command templates (MOP) | `<!var!>` | `show interface <!interface!>` |
| `makeData` input | `<!var!>` | `{"name": "<!name!>", "ip": "<!ipaddress!>"}` |
| Workflow variable refs | `$var.job.x` or `$var.taskId.x` | `$var.job.deviceName` |
| childJob variable refs | `{"task":"job","value":"varName"}` | `{"task": "static", "value": ["a"]}` |
| merge/evaluation refs | `{"task":"job","variable":"varName"}` | `{"task": "static", "variable": "success"}` |

## Workflow Patterns

### Error Handling: Try-Catch Pattern

Workflows with no error transitions on tasks will get **stuck** when a task fails -- the job stays running with no path forward. Every task that can fail needs error handling.

**Try-catch in child workflows:**

Inside each child workflow, catch errors using `newVariable` to set a status flag:

```
task --success--> newVariable("taskStatus" = "success") -> workflow_end
task --error--> newVariable("taskStatus" = "error") -> workflow_end
```

The child workflow **always completes** (never gets stuck), and the parent can check the result.

**Try-catch in parent workflows:**

After each `childJob`, extract and evaluate the child's `taskStatus`:

```
childJob -> query (extract taskStatus from job_details) -> evaluation (is it "success"?)
  |-- success -> continue
  |-- failure -> handle error
```

**Example:**
```json
{
  "a110": {
    "name": "query",
    "variables": {
      "incoming": {"query": "taskStatus", "obj": "$var.a100.job_details", "pass_on_null": false},
      "outgoing": {"return_data": "$var.job.createStatus"}
    }
  },
  "a120": {
    "name": "evaluation",
    "variables": {
      "incoming": {
        "all_true_flag": true,
        "evaluation_groups": [{
          "all_true_flag": true,
          "evaluations": [{
            "operand_1": {"variable": "createStatus", "task": "job"},
            "operator": "==",
            "operand_2": {"variable": "success", "task": "static"}
          }]
        }]
      }
    }
  }
}
```

### Manual Tasks (Human-in-the-Loop)

Manual tasks (`type: "manual"`) pause the workflow for human interaction. They require a `view` property pointing to the UI controller:

```json
{
  "name": "ViewData",
  "type": "manual",
  "view": "/workflow_engine/task/ViewData",
  "variables": {
    "incoming": {
      "header": "Approval Required",
      "message": "Review and approve to continue.",
      "body": "$var.job.dataToReview",
      "btn_success": "Approve",
      "btn_failure": "Reject"
    }
  }
}
```

The workflow pauses at manual tasks until a human interacts via the UI. `btn_success` triggers `success` transition, `btn_failure` triggers `failure` transition.

### autoApprove Pattern

Use an `evaluation` task to conditionally skip manual approval:

```
evaluation (autoApprove == true?)
  |-- success -> skip to next task (auto-approved)
  |-- failure -> ViewData (human reviews and approves/rejects)
```

The workflow accepts an `autoApprove` boolean input. When `true`, skips the manual step. Useful for CI/CD pipelines that run unattended vs interactive operator sessions.

### Pre-Check / Post-Check Design

**Pre-checks** validate conditions BEFORE making changes:
- Base interface is up: `"line protocol is up"` (contains)
- Sub-interface does NOT exist yet: `"GigabitEthernet1.910"` (`!contains`) -- the `!contains` eval means "rule passes if string is NOT found"
- Target system is reachable

**Post-checks** verify the change was applied correctly:
- Config is present: `"encapsulation dot1Q 910"` (contains)
- Interface is up: `"line protocol is"` (contains)

Design pre-checks around what MUST be true before the change, and post-checks around what SHOULD be true after.

### Network Device Config Pattern

When automating network device changes, use this pattern:

**1. MOP command templates for validation and checks only** (reference `/itential-devices` for device operations)
- Pre-checks: `show vlan brief`, `show interfaces switchport`, `show ip bgp summary`
- Post-checks: same commands with validation rules to confirm changes applied
- MOP is for running show commands and evaluating output -- NOT for pushing config

**2. Jinja2 templates to generate configuration** (reference `/itential-studio` for template creation)
- Render config snippets using Jinja2 templates with variables
- Test with `POST /template_builder/templates/{name}/renderJinja` before pushing

**3. Push config via existing workflow or `itential_cli` task**
- Search for existing push workflows first -- these may not exist in every environment
- If no push workflow exists, use `itential_cli` task directly
- Ask the engineer which push method they prefer

**Never use MOP command templates to push configuration to network devices.** MOP is for read-only validation.

**4. Test commands against the actual device BEFORE building workflows**
- Run `show` commands via MOP first to understand device capabilities
- Test a single CLI command via `itential_cli` before building the full workflow
- **Always review task output** -- a job can show `status: complete` even when CLI commands return errors. Check `stdout` for actual command results.

### Revert Transitions (Retry Loops)

Use `"type": "revert"` transitions to go backward for retry scenarios:

```
renderTemplate -> viewConfig (approve/reject)
  |-- success -> pushConfig -> evalSuccess
  |                             |-- success -> end
  |                             |-- failure -> viewError (retry/abort)
  |                                             |-- success (retry) --revert--> renderTemplate
  |                                             |-- failure (abort) -> end
  |-- failure (reject) --revert--> renderTemplate
```

The `revert` transition moves execution back to a previous task, allowing the user to fix inputs and retry.

### Modular Workflow Design

Build workflows as small, testable child workflows composed via `childJob` in a parent:

```
Parent Workflow
  |-- childJob (parallel) -> Child: Data Gathering (one per item)
  |-- renderJinja2ContextWithCast -> Format results into report
  |-- childJob -> Child: External Action (e.g., create ticket)
```

**Principles:**
- **Check for existing workflows first** -- before building new, search with `GET /automation-studio/workflows?include=name&limit=100`. Reuse as childJobs instead of rebuilding. Don't assume specific workflows exist -- always search first.
- Each child workflow should be independently testable via `jobs/start`
- Child workflows have clear input/output contracts via `inputSchema`/`outputSchema`
- Use `data_array` + `loopType: "parallel"` to fan out across multiple items
- Pass childJob output directly to the next task's `$var` input -- don't try to restructure with `newVariable`
- **Keep ALL asset JSON files locally** in the use-case directory. Edit locally, PUT/PATCH to update. Don't recreate -- it's cheaper to patch.

**Chaining childJob output to a template:**

The childJob output has structure `{status, loop: [{...child vars...}, ...]}`. Pass it directly as `variables` to `renderJinja2ContextWithCast` and iterate over `loop` in the template:

```json
{
  "incoming": {
    "template": "Report\n{% for item in loop %}\n- {{ item.fieldName }}\n{% endfor %}",
    "variables": "$var.job.childJobResults",
    "castDataType": "string"
  }
}
```

## Filesystem-First Debugging

**CRITICAL: The local filesystem has complete API documentation and platform data. Always check local files before making API calls or guessing.**

After bootstrap, your use-case directory contains everything you need:

| File | What it answers |
|------|-----------------|
| `openapi.json` | What endpoints exist? What method (GET/POST/PUT)? What's the request body schema? What does the response look like? What fields are required vs optional? |
| `tasks.json` | What's the task called? What app provides it? What are the incoming/outgoing variable names? Is it deprecated? |
| `task-schemas.json` | What are the full types, descriptions, and examples for each task's variables? (saved after first schema fetch) |
| `apps.json` | What's the correct app name? What type is it (Application/Adapter)? |
| `adapters.json` | What's the adapter instance name? What package? Is it running? |
| `applications.json` | Is the application healthy? What state is it in? |
| `environment.md` | Quick reference: which adapters map to which types, task counts per source |

### When to Check Local Files

**Before building a request body:** Don't guess field names or structure. Look it up:
```bash
# What fields does this endpoint expect?
jq '.paths["/automation-studio/templates"].post.requestBody.content["application/json"].schema' {use-case}/openapi.json

# What does the response look like?
jq '.paths["/automation-studio/templates"].post.responses["200"]' {use-case}/openapi.json
```

**Before fetching task schemas:** Check if you already have them:
```bash
# Search local task-schemas.json first
jq '.[] | select(.name == "renderJinjaTemplate")' {use-case}/task-schemas.json

# Only call multipleTaskDetails for tasks NOT already saved locally
```

**When a task isn't found:** Search the local catalog, don't guess:
```bash
# Search by keyword
grep -i "compliance" {use-case}/tasks.json | grep '"name"'

# Search by app
jq '.[] | select(.app == "ConfigurationManager") | .name' {use-case}/tasks.json
```

**When a field name seems wrong:** Check the schema, don't try variations:
```bash
# Get the exact field names for a task
jq '.[] | select(.name == "childJob") | .variables.incoming | keys' {use-case}/task-schemas.json
```

**When an API call returns 404 or unexpected data:**
```bash
# Verify the endpoint exists and check the method
jq '.paths | keys[] | select(contains("templates"))' {use-case}/openapi.json

# Check if it's GET, POST, PUT, etc.
jq '.paths["/automation-studio/templates"] | keys' {use-case}/openapi.json
```

### Common Mistakes This Prevents

| Mistake | Local file that prevents it |
|---------|----------------------------|
| Wrong HTTP method (GET vs POST) | `openapi.json` -- `.paths[endpoint] \| keys` |
| Wrong field name in request body | `openapi.json` -- `.requestBody.content...schema.properties \| keys` |
| Wrong task name (typo or invented) | `tasks.json` -- grep for the real name |
| Wrong app casing (`servicenow` vs `Servicenow`) | `apps.json` -- exact name |
| Wrong adapter instance name | `adapters.json` -- `.results[].id` |
| Re-fetching schemas you already have | `task-schemas.json` -- search before calling API |
| Guessing response wrapper (`data` vs `items`) | `openapi.json` -- response schema |

**Rule: If you're about to guess, stop and read a file instead. The answer is already on disk.**

## API Response Shapes

Most platform API responses are wrapped in `{message, data, metadata}`. Always extract from `data`:

| Endpoint | Extract |
|---|---|
| `POST /operations-manager/jobs/start` | `response.data._id` (job ID) |
| `GET /operations-manager/jobs/{id}` | `response.data.status`, `response.data.variables`, `response.data.error` |
| `POST /automation-studio/projects` | `response.data._id` |
| `GET /automation-studio/projects` | `response.data` (array of projects) |
| `DELETE /automation-studio/projects/{id}` | `response.message` |
| `POST /automation-studio/automations` | `response.created._id` (uses `{created, edit}` shape, NOT `{data}`) |
| `POST /automation-studio/templates` | `response.created._id` (same `{created, edit}` shape) |

**Exception:** `GET /automation-studio/workflows` and `GET /automation-studio/templates` return `{items, skip, limit, total}` -- NO `data` wrapper.

**Exception:** `GET /automation-studio/workflows/detailed/{name}` returns the workflow document directly -- NO wrapper.

### Adapter URI Prefix

Adapters have a `base_path` configured in their adapter settings (e.g., `/api` for ServiceNow, NetBox). The `genericAdapterRequest` task **automatically prepends** this base_path to the `uriPath` you provide.

The task schema says: *"do not include the host, port, base path or version"*

| What you want to call | Correct `uriPath` | Wrong `uriPath` | Result of wrong |
|---|---|---|---|
| `https://snow.example.com/api/now/table/change_request` | `/now/table/change_request` | `/api/now/table/change_request` | `/api/api/now/table/...` -> 400 error |

If you need to bypass the base_path prepend, use `genericAdapterRequestNoBasePath` instead.

### Asset Validation Before Running

Before running a workflow, **always validate that all referenced assets exist** on the target platform:

- **Devices**: `GET /configuration_manager/devices/{name}`
- **Jinja2 templates**: `POST /template_builder/templates/{name}/renderJinja` with `{"context":{}}` -- if it renders, it exists
- **Command templates**: `GET /mop/listATemplate/{name}`
- **CM device templates**: `POST /configuration_manager/templates/search` with `{"name": "..."}`
- **Adapters**: `GET /health/adapters` -- check state is `RUNNING`
- **Child workflows**: `GET /automation-studio/workflows?include=name` -- verify names exist
- **Existing workflows to reuse**: before building a new workflow, check if one already exists that does what you need

Missing assets cause runtime errors or draft workflows that can't be started.

### Updating Assets (PUT/PATCH vs POST)

**Always keep asset JSON files locally** in the use-case directory. Edit the local file and use PUT/PATCH to update instead of creating new assets each time:

| Asset | Create | Update | Local File |
|-------|--------|--------|------------|
| Workflow | `POST /automation-studio/automations` | `PUT /automation-studio/automations/{id}` with `{"update": {...}}` | `wf-{name}.json` |
| Template | `POST /automation-studio/templates` | `PUT /automation-studio/templates/{id}` with `{"update": {...}}` | `tmpl-{name}.json` |
| Command Template | `POST /mop/createTemplate` | `POST /mop/updateTemplate/{name}` with `{"mop": {...}}` | `mop-{name}.json` |

**Development workflow:** create asset -> save JSON locally -> edit local file -> PUT/PATCH to update -> test -> iterate.

## Gotchas

### Building Tasks

1. **`canvasName` must come from `tasks.json`** -- look it up: `jq '.[] | select(.name == "arrayPush") | .canvasName' {use-case}/tasks.json`. Some differ from the method name: `arrayPush` → `push`, `stringConcat` → `concat`. Never invent a canvasName — it controls the task icon in the UI.
2. **Task IDs must be hex `[0-9a-f]{1,4}`** -- non-hex IDs cause silent `$var` failure.
3. **Set `actor: "Pronghorn"` on adapter and application tasks** -- tasks like `merge`, `makeData`, `arrayPush`, `RunCommandTemplate`, `itential_cli`, `renderJinjaTemplate` need it. Omitting it can cause silent null outputs. Tasks like `evaluation`, `query`, `ViewData` work without it. When in doubt, include it.
4. **Validation errors = draft workflow** that cannot be started. Check the `errors` array after creation.
5. **Every task that can fail needs an error transition** -- without it, errors cause "Job has no available transitions" and the job gets stuck.

### $var Resolution

6. **`$var` inside nested objects doesn't resolve** -- `{"body": {"key": "$var.job.x"}}` stores the literal string. Use `merge`, `makeData`, or `query` to build the object, then pass it as a top-level `$var` reference.
7. **`newVariable` value with `$var` stores the literal string** -- use merge + query to build dynamic values.
8. **Variable syntax differs by context** -- Jinja2: `{{ }}`, command templates: `<! !>`, workflow wiring: `$var`, childJob: `{task, value}`, merge/evaluation: `{task, variable}`. Don't mix them.

### childJob

9. **`actor` MUST be `"job"`** -- not `"Pronghorn"`.
10. **`task` MUST be `""` (empty string)** -- the engine auto-sets it at runtime.
11. **`variables` MUST use `{"task":"job","value":"varName"}`** -- NOT `$var.job.x`. Using `$var` inside the variables object causes the childJob to hang indefinitely.
12. **Pass task outputs using `{"task":"taskId","value":"outVar"}`** -- this references a previous task's outgoing variable at runtime.
13. **`outgoing.job_details` MUST be `null`** -- do NOT override with `$var.job.X`.
14. **`job_details` shows `""` in the job document but resolves at runtime** -- use `query` on `$var.taskId.job_details` to extract child output. The standard pattern: childJob → query. Don't be confused by the empty string in the GET response.
15. **Validates child's `inputSchema.required` at creation time** -- missing required inputs cause validation error when creating the parent, not at runtime. Check the child's inputSchema before wiring.
16. **"Cannot find workflow" after project move** -- project move renames workflows with `@projectId:` prefix but does NOT update childJob refs. Create the project first and build inside it.

### merge

17. **Uses `"variable"` not `"value"`** in `data_to_merge` reference items. childJob uses `"value"`. Don't mix them.
18. **Requires at least 2 items** -- with 1 item, `merged_object` is silently `null` (no error). The downstream task fails instead, hiding the root cause.
19. **Outgoing MUST declare `"merged_object": null`** -- empty `{}` makes the output unreachable via `$var.taskId.merged_object`.

### makeData

20. **`<!var!>` names must match source object keys exactly** -- `<!ipaddress!>` not `<!ip!>`.
21. **Two different uses** -- (a) variable substitution: `input` is template with `<!var!>`, `variables` is the data object, `outputType: "string"`. (b) JSON parsing: `input` is a JSON string, `variables` is `""`, `outputType: "json"`. Both common in production.
22. **`variables` must be a resolved object** -- use merge first to build it, then pass via `$var.taskId.merged_object`.

### Other Tasks

23. **`evaluation` MUST have both success AND failure transitions** -- missing failure transition causes job error.
24. **`forEach` last body task transition must be empty `{}`** -- do not connect it back to forEach.
25. **`push`/`pop`/`shift` operate on job variables by NAME** -- pass `"myArray"`, not `"$var.job.myArray"`.
26. **Tasks with `incomplete` status are normal** -- untaken branches leave tasks incomplete. Not an error.

### General

27. **Reuse task types freely** -- multiple instances with different hex task IDs (e.g., two `query` tasks with IDs `c3d4` and `e5f6`). Use `canvasName` from palette, differentiate with `summary`.
28. **`status: complete` doesn't mean CLI commands succeeded** -- check `stdout` for actual results.
29. **Use JSON files for API payloads** -- `curl -d @file.json` avoids shell escaping issues with `$var` and nested quotes.

## Helper Templates

| File | Purpose |
|------|---------|
| `helpers/workflow-task-childjob.json` | childJob task template (`actor: "job"`, `task: ""`) |
| `helpers/workflow-task-adapter.json` | Adapter task template |
| `helpers/workflow-task-application.json` | Application task template |
| `helpers/create-workflow.json` | Workflow scaffold with start/end tasks |
| `helpers/reference-parent-workflow.json` | **Reference:** parent with childJob → query → evaluation → newVariable → childJob → query. Read this when building multi-child workflows. |
| `helpers/reference-child-workflow.json` | **Reference:** child with makeData(json) → query → merge(from task output) → makeData(string). Read this when wiring makeData or merge. |
| `helpers/reference-merge-makedata.json` | **Reference:** merge → makeData pattern for template variable substitution. |

**ALWAYS start from a helper template when creating workflow tasks.** Read the helper file first, then modify for your use case. Do NOT build task JSON from scratch.

**When building multi-child workflows or wiring merge/makeData/query:** read the reference helpers first. They are complete working workflows exported from a tested project — correct `canvasName`, correct `actor`, correct variable syntax throughout.

## Developer Scenarios

### 1. Test a Utility Task

Create a minimal workflow (`start -> task -> end`) and run it:

1. Read the helper template: `helpers/create-workflow.json`
2. Add a single task (e.g., `query`) between `workflow_start` and `workflow_end`
3. Wire incoming variables from `$var.job.x` and outgoing to `$var.job.result`
4. Create the workflow: `POST /automation-studio/automations`
5. Start the job: `POST /operations-manager/jobs/start` with test input variables
6. Check the result: `GET /operations-manager/jobs/{jobId}` -- inspect `data.variables`

### 2. Debug a Failed Job

1. **Get job details:** `GET /operations-manager/jobs/{jobId}`
2. **Check `data.status`** -- if `"error"`, check `data.error` array
3. **Read `IAPerror.displayString`** for the human-readable error message
4. **Identify the failing task** -- `data.error[].task` gives the task ID
5. **Check task status** -- look for `metrics.finish_state` on the failing task
6. **Use filesystem-first debugging** -- check `tasks.json` for correct task name, `openapi.json` for correct endpoint schema, `task-schemas.json` for correct variable names

**Common failure causes:**
- Task name doesn't exist on the platform -> search `tasks.json`
- Wrong variable names -> check `task-schemas.json`
- `$var` reference to non-hex task ID -> verify task IDs are `[0-9a-f]{1,4}`
- Missing error transition -> job stuck in `running`
- Adapter not running -> check `GET /health/adapters`

### 3. Build a childJob Orchestrator

1. **Build and test each child workflow independently** -- run via `jobs/start`, verify outputs
2. **Create the parent workflow** using `helpers/create-workflow.json`
3. **Add childJob tasks** using `helpers/workflow-task-childjob.json` -- set `actor: "job"`, `task: ""`
4. **Wire variables** -- use `{"task": "taskId", "value": "outVar"}` for runtime data
5. **Add query tasks** after each childJob to extract results from `job_details`
6. **Add evaluation tasks** to check child success/failure (try-catch pattern)
7. **Test the parent** -- run via `jobs/start`, check that all child jobs complete

### Cross-References

- **Creating workflows and templates:** Use `/itential-studio`
- **Device operations and config management:** Use `/itential-devices`
- **Command templates and MOP:** Use `/itential-devices`
- **Golden config and compliance:** Use `/itential-golden-config`
- **IAG services from workflows:** Use `/iag`
