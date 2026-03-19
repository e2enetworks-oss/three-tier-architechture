---
name: create-dr-plan
description: Create Disaster Recovery (DR) plans for nodes on E2E Networks. Accepts node names or IDs, resolves them, and creates a DR plan for each via the DRaaS API.
disable-model-invocation: true
allowed-tools: Bash, ToolSearch, AskUserQuestion
argument-hint: "[node names or IDs, e.g. 'my-node-1, my-node-2' or '299743, 299744']"
---

# Create DR Plan on E2E Networks

You are creating Disaster Recovery plans for one or more nodes on E2E Networks. Follow the **collect-then-execute** pattern strictly.

## Execution Rules

- **Proceed autonomously** — do NOT ask for confirmation between steps.
- **Apply all defaults silently** — use defaults without asking.
- **Only pause for user input in these two cases**:
  1. Required credentials are completely missing and cannot be found in `~/.e2e/config.json`
  2. A hard error occurs (API returns non-200, node not found, plan creation fails)

---

## Phase 1 — Collect All Inputs

### Step 1.0 — Check for saved credentials

```bash
cat ~/.e2e/config.json 2>/dev/null
```

Load credentials using Python:
```bash
export API_TOKEN=$(python3 -c "import json,os; d=json.load(open(os.path.expanduser('~/.e2e/config.json'))); print(d['api_token'])")
export API_KEY=$(python3 -c "import json,os; d=json.load(open(os.path.expanduser('~/.e2e/config.json'))); print(d['api_key'])")
export PROJECT_ID=$(python3 -c "import json,os; d=json.load(open(os.path.expanduser('~/.e2e/config.json'))); print(d['project_id'])")
export LOCATION=$(python3 -c "import json,os; d=json.load(open(os.path.expanduser('~/.e2e/config.json'))); print(d.get('location','Delhi'))")
```

If credentials are missing, ask the user for everything in a **single message**:

```
To create DR plans on E2E Networks, I need:

CREDENTIALS (required):
  1. API Token     : <your token>
  2. API Key       : <your API key>
  3. Project ID    : <your project ID>

DR PLAN CONFIGURATION:
  4. Node(s)                  : <names or IDs, comma-separated>
  5. Target Location          : <Chennai, Mumbai, Delhi, Delhi-NCR-2>
  6. RPO (hours)              : <1–24, default: 3>
  7. Retention (days)         : <1–30, default: 7>
  8. Source Location          : <Delhi (default), Mumbai, Chennai, Delhi-NCR-2>
```

### Defaults (apply silently)

| Field | Default |
|---|---|
| Source Location | from `~/.e2e/config.json` or Delhi |
| RPO hours | 3 |
| Retention days | 7 |
| Resource type | Node |
| Volumes | [] |

### Plan naming convention

Auto-generate plan names as: `{SourceLocation}_DR_{NodeName}_{timestamp}`
Example: `Delhi_DR_my-node-1_1710000000`

---

## Phase 2 — Resolve Node IDs

If the user provided **node names** (not numeric IDs), resolve them to IDs by fetching the nodes list:

```bash
GET https://api.e2enetworks.com/myaccount/api/v1/nodes/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
Authorization: Bearer {API_TOKEN}
```

Use Python to match names case-insensitively:

```python
import subprocess, json

def get_nodes(api_key, project_id, location, token):
    r = subprocess.run([
        "curl", "-s",
        f"https://api.e2enetworks.com/myaccount/api/v1/nodes/?apikey={api_key}&project_id={project_id}&location={location}",
        "-H", f"Authorization: Bearer {token}"
    ], capture_output=True, text=True)
    return json.loads(r.stdout).get("data", [])

all_nodes = get_nodes(API_KEY, PROJECT_ID, LOCATION, API_TOKEN)

# Match by name (case-insensitive) or numeric ID
target_inputs = ["my-node-1", "my-node-2"]  # from user input
resolved = []
for inp in target_inputs:
    inp = inp.strip()
    if inp.isdigit():
        # User gave a numeric ID — find node details
        match = next((n for n in all_nodes if str(n["id"]) == inp), None)
        if match:
            resolved.append({"id": match["id"], "name": match["name"]})
        else:
            print(f"WARNING: No node found with ID {inp}")
    else:
        # User gave a name
        match = next((n for n in all_nodes if n["name"].lower() == inp.lower()), None)
        if match:
            resolved.append({"id": match["id"], "name": match["name"]})
        else:
            print(f"WARNING: No node found with name '{inp}'")

print(f"Resolved {len(resolved)} node(s):")
for n in resolved:
    print(f"  {n['name']} (ID: {n['id']})")
```

If any node cannot be resolved, report it clearly and skip it (continue with the rest).

---

## Phase 3 — Create DR Plans

Create one DR plan per resolved node. Use the following API:

```
POST https://api.e2enetworks.com/myaccount/api/v1/draas/create-plan/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
Authorization: Bearer {API_TOKEN}
Content-Type: application/json
```

Payload:
```json
{
    "plan_name": "{SourceLocation}_DR_{NodeName}_{timestamp}",
    "target_location": "{TARGET_LOCATION}",
    "resource_type": "Node",
    "source_resource_id": {NODE_ID},
    "rpo_hours": {RPO_HOURS},
    "recovery_point_retention_days": "{RETENTION_DAYS}",
    "volumes": []
}
```

**IMPORTANT**: `recovery_point_retention_days` must be a **string**, not an integer (e.g. `"7"` not `7`).

Use Python to create plans sequentially and report results:

```python
import subprocess, json, time

def create_dr_plan(api_key, project_id, location, token, payload):
    r = subprocess.run([
        "curl", "-s", "-X", "POST",
        f"https://api.e2enetworks.com/myaccount/api/v1/draas/create-plan/?apikey={api_key}&project_id={project_id}&location={location}",
        "-H", f"Authorization: Bearer {token}",
        "-H", "Content-Type: application/json",
        "-d", json.dumps(payload)
    ], capture_output=True, text=True)
    return json.loads(r.stdout)

results = []
for node in resolved:
    plan_name = f"{SOURCE_LOCATION}_DR_{node['name']}_{int(time.time())}"
    payload = {
        "plan_name": plan_name,
        "target_location": TARGET_LOCATION,
        "resource_type": "Node",
        "source_resource_id": node["id"],
        "rpo_hours": RPO_HOURS,
        "recovery_point_retention_days": str(RETENTION_DAYS),
        "volumes": []
    }
    resp = create_dr_plan(API_KEY, PROJECT_ID, LOCATION, API_TOKEN, payload)
    code = resp.get("code", "?")
    msg = resp.get("message", "")
    plan_id = resp.get("data", {}).get("id", "") if isinstance(resp.get("data"), dict) else ""
    status = "SUCCESS" if code in (200, 201) else "FAILED"
    print(f"  [{status}] {node['name']} (ID: {node['id']}) → plan '{plan_name}' | code={code} {msg}")
    results.append({
        "node": node["name"],
        "node_id": node["id"],
        "plan_name": plan_name,
        "plan_id": plan_id,
        "status": status,
        "code": code,
        "message": msg
    })
    time.sleep(1)  # small delay between creates
```

---

## Phase 4 — Report Summary

After all plans are processed:

```
DR Plans Created!

  Node             | Node ID | Plan Name                        | Status
  -----------------|---------|----------------------------------|--------
  my-node-1        | 299743  | Delhi_DR_my-node-1_1710000000   | SUCCESS
  my-node-2        | 299744  | Delhi_DR_my-node-2_1710000001   | SUCCESS

CONFIGURATION:
  Source Location  : Delhi
  Target Location  : Chennai
  RPO              : 3 hours
  Retention        : 7 days
  Resource Type    : Node
```

If any plan failed, include the error message:
```
  my-node-3        | 299745  | Delhi_DR_my-node-3_1710000002   | FAILED — <error message>
```

---

## Error Handling

| Error | Fix |
|---|---|
| `code != 200/201` | Print full API response, report as FAILED, continue with remaining nodes |
| Node name not found | Warn and skip — list all available node names from the GET /nodes/ response to help user correct the input |
| Node ID not found in list | The node may be in a different location — remind user that location must match the source node's location |
| `recovery_point_retention_days` type error | Ensure it is passed as a string (`"7"` not `7`) |
| Duplicate plan name | Append a different timestamp suffix |
| Credentials missing | Ask user in a single prompt before proceeding |
