---
name: three-tier-arch
description: Create a complete three-tier architecture with load balancers on E2E Networks. Launches a frontend LB + N frontend nodes + a backend LB + N backend nodes + database nodes, installs HAProxy on LBs, installs app tools, connects all tiers, and optionally seeds dummy data. Use when the user wants to set up a full-stack infrastructure.
disable-model-invocation: true
allowed-tools: Bash, WebFetch, ToolSearch, Read, Glob, Grep, Write, Edit, AskUserQuestion
argument-hint: "[stack specs e.g. 'React frontend x2 with LB, Django backend x3 with LB, PostgreSQL db x1']"
---

# Three-Tier Architecture on E2E Networks

You are setting up a complete three-tier architecture (Frontend, Backend, Database) on E2E Networks. Follow the **collect-then-execute** pattern strictly.

> **EXECUTION RULE — MUST FOLLOW**: After inputs are collected, execute every phase and step automatically without pausing. **Never ask "Do you want to proceed?", "Shall I continue?", "Ready to create?", or any confirmation question between steps.** Only stop if an API call returns an error or a required value cannot be resolved. Treat the user's invocation of this skill as full authorization to run all steps to completion.

---

## Phase 0 — Load Config File (Always run first)

Before asking the user for anything, check for a config file at `~/.e2e/config.json`. Read it silently:

```bash
cat ~/.e2e/config.json 2>/dev/null
```

If the file exists and is valid JSON, extract any fields present and use them as pre-filled values. Fields found in the config file are treated as if the user provided them — do NOT ask for them again.

### Config File Format

```json
{
  "api_token": "your-bearer-token",
  "api_key": "your-api-key",
  "project_id": "your-project-id",
  "location": "Delhi",
  "ssh_key": "ssh-rsa AAAA...",
  "disable_password": true,
  "backups": false,
  "enable_bitninja": false,
  "is_ipv6_availed": false
}
```

All fields are optional — only present fields are used. Missing fields fall through to `$ARGUMENTS` or the user prompt.

If the file does not exist or cannot be parsed, proceed silently to Phase 1 without mentioning the file.

> **Tip for users**: Create this file once to avoid re-entering credentials every time:
> ```bash
> mkdir -p ~/.e2e && cat > ~/.e2e/config.json << 'EOF'
> {
>   "api_token": "your-token-here",
>   "api_key": "your-api-key-here",
>   "project_id": "your-project-id-here",
>   "location": "Delhi"
> }
> EOF
> ```

---

## Phase 1 — Collect All Inputs (Single Prompt)

Merge inputs in this priority order (highest wins):
1. `$ARGUMENTS` (inline user input)
2. Config file (`~/.e2e/config.json`)
3. Defaults defined below

Only prompt the user for fields that are **still missing** after merging. If all required fields are resolved, skip the prompt entirely and proceed to Phase 2.

If any required fields remain missing, ask for **only the missing ones** in a single message:

```
To set up a three-tier architecture on E2E Networks, I still need
(already-loaded values from ~/.e2e/config.json are not shown):

CREDENTIALS (required, if not in config):
  - API Token     : <your token>
  - API Key       : <your API key>
  - Project ID    : <your project ID>

FRONTEND LOAD BALANCER (if not in config or $ARGUMENTS):
  - Name          : auto-generated as {location}-frontend-lb
  - OS / Version  : <e.g., Ubuntu 24.04>
  - Plan Size     : <e.g., smallest>

FRONTEND NODE (if not in config or $ARGUMENTS):
  - Instances     : <number of frontend nodes, e.g. 2>
  - Name          : auto-generated as {location}-frontend-1, {location}-frontend-2, ...
  - OS / Version  : <e.g., Ubuntu 24.04>
  - Plan Size     : <e.g., C3.8GB, smallest, 8GB RAM>
  - Tools         : <e.g., Nginx + Node.js 20 + React>

BACKEND LOAD BALANCER (if not in config or $ARGUMENTS):
  - Name          : auto-generated as {location}-backend-lb
  - OS / Version  : <e.g., Ubuntu 24.04>
  - Plan Size     : <e.g., smallest>

BACKEND NODE (if not in config or $ARGUMENTS):
  - Instances     : <number of backend nodes, e.g. 2>
  - Name          : auto-generated as {location}-backend-1, {location}-backend-2, ...
  - OS / Version  : <e.g., Ubuntu 24.04>
  - Plan Size     : <e.g., C3.8GB>
  - Tools         : <e.g., Python 3.11 + Django 4.2>

DATABASE NODE (if not in config or $ARGUMENTS):
  - Instances     : <number of database nodes, e.g. 1>
  - Name          : auto-generated as {location}-database (single), or {location}-database-1, {location}-database-2, ... (multiple)
  - OS / Version  : <e.g., Ubuntu 24.04>
  - Plan Size     : <e.g., C3.8GB>
  - Database      : <e.g., MySQL 8, PostgreSQL 16>

INSTALLATION METHOD (must always ask — no default):
  - Install via   : SSH (post-creation) OR start scripts (cloud-init at boot)?

GENERAL (if not in config):
  - Location      : <Delhi (default), Mumbai, Delhi-NCR-2, Chennai>
  - SSH Key       : <full ssh-rsa key string to upload> OR "use existing"

OPTIONAL:
  - Backups       : true/false (default: false)
  - BitNinja      : true/false (default: false)
  - Volume        : <volume name> — e.g. "attach volume V_100GB_157" (specify which tier)
  - Start Script  : <script label> OR "create new"
```

Only include fields in the prompt that are actually missing — omit any already resolved from config or `$ARGUMENTS`.
**Exception**: Always ask Installation Method — this is intent-driven and must not be assumed from config.

### Defaults (apply silently if not specified)

| Field | Default |
|---|---|
| Frontend LB Name | `{location}-frontend-lb` (always 1, no number) |
| Frontend LB OS | Ubuntu 24.04 |
| Frontend Instances | 2 |
| Frontend Name | `{location}-frontend` if 1 instance; `{location}-frontend-1`, `{location}-frontend-2`, ... if multiple |
| Frontend OS | Ubuntu 24.04 |
| Backend LB Name | `{location}-backend-lb` (always 1, no number) |
| Backend LB OS | Ubuntu 24.04 |
| Backend Instances | 2 |
| Backend Name | `{location}-backend` if 1 instance; `{location}-backend-1`, `{location}-backend-2`, ... if multiple |
| Backend OS | Ubuntu 24.04 |
| Database Instances | 1 |
| Database Name | `{location}-database` if 1 instance; `{location}-database-1`, `{location}-database-2`, ... if multiple |
| Database OS | Ubuntu 24.04 |
| All Plan Sizes | Smallest available |
| Frontend Tools | Nginx + Node.js 20 |
| Backend Tools | Python 3.11 + Django 4.2 |
| Database | MySQL 8.0 |
| Location | Delhi |
| SSH Key | First existing key from account; if none, STOP and ask user to provide their SSH public key string |

### IMPORTANT: One question that MUST be asked (no default)

1. **Installation method**: "Should I install tools via **SSH after node creation** or via **start scripts (cloud-init at boot)**?"

---

## Phase 2 — Create All Three Nodes

Run all steps in this phase sequentially without stopping or asking for confirmation. Use the same node creation logic as the `/launch-node` skill. The API workflow is:

### API Configuration

- Base URL: `https://api.e2enetworks.com`
- Header: `Authorization: Bearer <API_TOKEN>`
- Header: `Content-Type: application/json`
- Query params: `apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}`

Use `curl` via Bash for all API calls.

### Step 2.1 — Validate OS Selection

```
GET {BASE_URL}/myaccount/api/v1/images/os-category/?active=true&apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Validate OS, version, and category for all three nodes. If all three use the same OS, one call is sufficient.

### Step 2.2 — Resolve Plans & Images

```
GET {BASE_URL}/myaccount/api/v1/images/?display_category={CATEGORY}&category={OS}&osversion={VERSION}&gpu_type=&ng_container=&os={OS}&apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Resolve plan SKU and image for each node. If all three use the same OS/version/category, one call is enough — just pick the right plan for each.

### Step 2.3 — Get Security Group & SSH Key

```
GET {BASE_URL}/myaccount/api/v1/security_group/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
GET {BASE_URL}/myaccount/api/v1/ssh_keys/?image={IMAGE}&apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Find `is_default: true` security group. Save its `id` as `SG_ID`.

**SSH Keys GET response format:**
```json
{
  "code": 200,
  "data": [
    {
      "pk": 8342,
      "label": "roniee007@rounak-Latitude",
      "ssh_key": "ssh-rsa AAAA...",
      "timestamp": "15-May-2023",
      "project_name": "default-project-23140",
      "ssh_key_type": "RSA",
      "total_attached_nodes": 2
    }
  ],
  "errors": {},
  "message": "Success"
}
```

Parse from `response.data[]`. Each key has `pk`, `label`, and `ssh_key` fields.

For SSH key:
- If user provided a raw SSH public key string → upload it first:
  ```
  POST {BASE_URL}/myaccount/api/v1/ssh_keys/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
  Body: { "label": "<any-node-name>", "ssh_key": "<SSH_PUBLIC_KEY_STRING>" }
  ```
  The response returns the created key object; save its `ssh_key` string as `SSH_KEY_STRING`. Use this key string for all nodes.
- If user named a key (e.g. "rounak's key") → fuzzy-match on `label` (case-insensitive) from `response.data[]`; use that entry's `ssh_key` string.
- If "use existing" or no preference → use the `ssh_key` field of the first entry in `response.data[]`; if the array is empty, STOP and ask the user to provide their SSH public key string.
- **Always use the `ssh_key` string value** (not `pk` or any integer) when passing to node creation.
- All nodes share the same SSH key string (`SSH_KEY_STRING`).

### Step 2.4 — Resolve Volume (only if user wants to attach a volume)

**Trigger:** Run only if the user mentioned attaching a volume/block storage. Skip otherwise.
**Trigger examples:** "attach volume V_100GB_157 to backend", "add a volume to the database node"

```
GET {BASE_URL}/myaccount/api/v1/block_storage/?per_page=100&page_no=1&apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Response format — read from `response.data`:
```json
{
  "code": 200,
  "data": [
    { "block_id": 40252, "name": "V_100GB_157", "size": 95368, "vm_detail": {}, "status": "Available", "size_string": "100 GB" }
  ],
  "errors": {},
  "message": "Success"
}
```

Logic:
1. Filter volumes where `status` is `"Available"`
2. If user named a specific volume → fuzzy-match on `name` (case-insensitive, partial)
3. If no name given and only one available → use it automatically, inform user
4. If no name given and multiple available → list them (`name`, `size_string`, `block_id`) and ask user to pick one
5. If matched volume is not `"Available"` → STOP, show attached node from `vm_detail`, ask for another
6. If no available volumes → STOP, inform user, ask if they want to continue without a volume
7. Ask user which tier node (frontend/backend/database) this volume should attach to
ON SUCCESS: Save `block_id` as `VOLUME_BLOCK_ID` and note the target tier

### Step 2.5 — Resolve Start Script (only if user explicitly named a custom boot script)

**Trigger:** Run only if the user explicitly named or requested a custom start/boot script (e.g., "run start script test on boot", "create new start script", "use my script labeled deploy"). Skip if the installation method is **SSH** and no custom script was mentioned — in that case `start_scripts` stays `[]`. If the user chose **start scripts** as the installation method in Phase 1, the tier-specific installation commands are generated inline during Step 2.6 (node creation) — do NOT run this step for that case.

#### Using an existing script

```
GET {BASE_URL}/myaccount/api/v1/start-script/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Response format (direct array):
```json
[
  { "id": 687, "label": "test", "script_content": "#!/bin/bash\nmkdir test" },
  { "id": 933, "label": "new",  "script_content": "#!/bin/bash\necho \"qqqq\"" }
]
```

Logic:
1. If user named a specific script → fuzzy-match on `label` (case-insensitive, partial); use it if found
2. If no name given and only one script exists → use it, inform user
3. If no name given and multiple scripts → list them (`id`, `label`) and ask user to pick
4. If no scripts exist → inform user, offer to create a new one
5. If named script not found → show available list, ask to pick or create new
ON SUCCESS: Save `script_content` as `START_SCRIPT_CONTENT`

#### Creating a new script

```
POST {BASE_URL}/myaccount/api/v1/start-script/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
Body: { "label": "<SCRIPT_LABEL>", "script_content": "<SCRIPT_CONTENT>" }
```

- Ask user for `label` and `script_content` if not already provided
- ON SUCCESS: Save the `script_content` as `START_SCRIPT_CONTENT`, inform user the script was created
- ON ERROR: STOP, show error

### Step 2.6 — Create All Nodes

For the two load balancers, make one POST each with `number_of_instances: 1`. For each application tier, make **one POST request** with `number_of_instances` set to the user's chosen count. The API will create all instances in a single call and return all IDs in the response.

**Creation order**: frontend-lb → frontend nodes → backend-lb → backend nodes → database nodes.

**Naming convention**: `{location}-{role}` — use the full lowercase location name (e.g. `chennai`, `delhi`, `mumbai`). Append a number only when there are multiple instances of the same role.

| Node | 1 instance | 2+ instances |
|---|---|---|
| Frontend LB | `{location}-frontend-lb` | _(always 1)_ |
| Frontend nodes | `{location}-frontend` | `{location}-frontend-1`, `{location}-frontend-2`, ... |
| Backend LB | `{location}-backend-lb` | _(always 1)_ |
| Backend nodes | `{location}-backend` | `{location}-backend-1`, `{location}-backend-2`, ... |
| Database nodes | `{location}-database` | `{location}-database-1`, `{location}-database-2`, ... |

Because the API creates all instances of a tier in one call, pass the **base name without the number** in the request body (e.g. `chennai-frontend`) — the server appends `-1`, `-2`, ... automatically when `number_of_instances > 1`. For tiers with exactly 1 instance, pass the base name as-is (no suffix).

```
POST {BASE_URL}/myaccount/api/v1/nodes/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Body:
```json
{
  "label": "default",
  "name": "<NODE_NAME>",
  "region": "ncr",
  "plan": "<PLAN_SKU>",
  "image": "<IMAGE>",
  "is_private": false,
  "ssh_keys": ["<SSH_KEY_STRING>"],
  "start_scripts": [],
  "backups": false,
  "enable_bitninja": false,
  "disable_password": true,
  "is_saved_image": false,
  "reserve_ip": "",
  "is_ipv6_availed": false,
  "vpc_id": null,
  "subnet_id": null,
  "default_public_ip": false,
  "ngc_container_id": null,
  "number_of_instances": <INSTANCE_COUNT_FOR_THIS_TIER>,
  "security_group_id": <SG_ID>,
  "is_encryption_required": false
}
```

**Field notes**:
- `ssh_keys`: array containing the SSH public key string (`SSH_KEY_STRING`) resolved in Step 2.3 — use the `ssh_key` field from the GET/POST response, NOT an integer ID.
- `security_group_id`: integer ID of the default security group (`SG_ID`) resolved in Step 2.3.

Set `number_of_instances` per node group:
- Frontend LB: always `1`
- Frontend:    `FRONTEND_INSTANCE_COUNT` (from user input or default 2)
- Backend LB:  always `1`
- Backend:     `BACKEND_INSTANCE_COUNT`  (from user input or default 2)
- Database:    `DATABASE_INSTANCE_COUNT` (from user input or default 1)

Volume attachment (if resolved in Step 2.4) is performed as a **separate API call after node creation** — it is not a field in the node creation body. After the node is Running, call:
```
POST {BASE_URL}/myaccount/api/v1/block_storage/<VOLUME_BLOCK_ID>/attach/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
Body: { "node_id": <TARGET_NODE_ID> }
```
Note: a volume can only be attached to **one** instance; warn the user if that tier has more than 1 instance, and attach to instance #1 only.

For `start_scripts` (use the actual script content string, not the script ID):
- If the user chose **start scripts** as the installation method → populate with the tier-specific installation commands (see Phase 3 scripts)
- If the user provided a custom start script via Step 2.5 → populate with `START_SCRIPT_CONTENT`
- Otherwise leave as `[]`

### Step 2.7 — Poll All Nodes Until Running

Collect all node IDs returned from the creation calls in Step 2.6 and poll each one until it reaches `Running` state. If a node enters a failed state (`Failed`, `Failed recreate`, `Error`, `Failure`), automatically create a replacement node with the same config and poll the replacement instead. Use the Python script below — do NOT use bash arrays for this.

```python
import subprocess, json, time

BASE_URL   = "https://api.e2enetworks.com"
API_KEY    = "<API_KEY>"
PROJECT_ID = "<PROJECT_ID>"
LOCATION   = "<LOCATION>"
API_TOKEN  = "<API_TOKEN>"

# Node creation params per tier — used to create replacements
node_create_params = {
    "frontend-lb": {"plan": "<PLAN_SKU>", "image": "<IMAGE>", "ssh_keys": ["<SSH_KEY_STRING>"], "security_group_id": <SG_ID>},
    "frontend":    {"plan": "<PLAN_SKU>", "image": "<IMAGE>", "ssh_keys": ["<SSH_KEY_STRING>"], "security_group_id": <SG_ID>},
    "backend-lb":  {"plan": "<PLAN_SKU>", "image": "<IMAGE>", "ssh_keys": ["<SSH_KEY_STRING>"], "security_group_id": <SG_ID>},
    "backend":     {"plan": "<PLAN_SKU>", "image": "<IMAGE>", "ssh_keys": ["<SSH_KEY_STRING>"], "security_group_id": <SG_ID>},
    "database":    {"plan": "<PLAN_SKU>", "image": "<IMAGE>", "ssh_keys": ["<SSH_KEY_STRING>"], "security_group_id": <SG_ID>},
}

# All node IDs grouped by tier — mutable lists so replacements can be swapped in
all_nodes = {
    "frontend-lb": [<FRONTEND_LB_ID>],
    "frontend":    [<FRONTEND_NODE_IDS...>],
    "backend-lb":  [<BACKEND_LB_ID>],
    "backend":     [<BACKEND_NODE_IDS...>],
    "database":    [<DB_NODE_IDS...>],
}

FAILED_STATES    = {"Error", "Failed", "Failed recreate", "Failure"}
MAX_POLLS        = 30    # 30 × 20 s = 10 minutes per node
POLL_INTERVAL    = 20
MAX_REPLACEMENTS = 2     # max auto-replacements per node slot

node_info = {}  # final node_id -> API response data

def create_replacement(tier, failed_name, replacement_num):
    params = node_create_params[tier]
    new_name = f"{failed_name}-r{replacement_num}"
    body = {
        "label": "default",
        "name": new_name,
        "region": "ncr",
        "plan": params["plan"],
        "image": params["image"],
        "is_private": False,
        "ssh_keys": params["ssh_keys"],
        "start_scripts": [],
        "backups": False,
        "enable_bitninja": False,
        "disable_password": True,
        "is_saved_image": False,
        "reserve_ip": "",
        "is_ipv6_availed": False,
        "vpc_id": None,
        "subnet_id": None,
        "default_public_ip": False,
        "number_of_instances": 1,
        "security_group_id": params["security_group_id"],
        "is_encryption_required": False,
    }
    url = (f"{BASE_URL}/myaccount/api/v1/nodes/"
           f"?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}")
    result = subprocess.run(
        ["curl", "-s", "-X", "POST",
         "-H", f"Authorization: Bearer {API_TOKEN}",
         "-H", "Content-Type: application/json",
         url, "-d", json.dumps(body)],
        capture_output=True, text=True
    )
    resp = json.loads(result.stdout)
    if resp.get("code") == 200:
        node = resp["data"]["node_create_response"][0]
        print(f"  Replacement created: {node['name']} | ID: {node['id']} | Public: {node['public_ip_address']}")
        return node["id"]
    else:
        print(f"  ERROR creating replacement: {resp}")
        raise SystemExit(1)

for tier, ids in all_nodes.items():
    i = 0
    while i < len(ids):
        node_id = ids[i]
        replacement_count = 0
        while True:
            running = False
            for attempt in range(1, MAX_POLLS + 1):
                url = (f"{BASE_URL}/myaccount/api/v1/nodes/{node_id}/"
                       f"?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}")
                result = subprocess.run(
                    ["curl", "-s", "-H", f"Authorization: Bearer {API_TOKEN}", url],
                    capture_output=True, text=True
                )
                try:
                    resp = json.loads(result.stdout)
                except Exception:
                    print(f"  [{tier}] node {node_id} — parse error, retrying...")
                    time.sleep(POLL_INTERVAL)
                    continue
                state = (resp.get("data", {}).get("status", "")
                         or resp.get("data", {}).get("state", ""))
                print(f"  [{tier}] node {node_id} — attempt {attempt}/{MAX_POLLS} — state: {state}")
                if state == "Running":
                    node_info[node_id] = resp["data"]
                    running = True
                    break
                elif state in FAILED_STATES:
                    print(f"  [{tier}] node {node_id} failed with state '{state}'.")
                    if replacement_count >= MAX_REPLACEMENTS:
                        print(f"  ERROR: max replacements ({MAX_REPLACEMENTS}) reached for [{tier}]. Stopping.")
                        raise SystemExit(1)
                    replacement_count += 1
                    failed_name = resp.get("data", {}).get("name", f"{tier}-node")
                    print(f"  Creating replacement #{replacement_count} for {failed_name}...")
                    node_id = create_replacement(tier, failed_name, replacement_count)
                    ids[i] = node_id   # swap failed ID with replacement ID
                    break              # restart poll loop for the new node
                if attempt == MAX_POLLS:
                    print(f"  ERROR: node {node_id} did not reach Running after {MAX_POLLS} polls.")
                    raise SystemExit(1)
                time.sleep(POLL_INTERVAL)
            if running:
                break   # node confirmed Running; move to next slot
        i += 1

print("All nodes are Running.")
```

Run this script via `python3 -c "$(cat <<'PYEOF'\n...\nPYEOF\n)"` or write it to a temp file and execute it. After all nodes are running, extract `public_ip` for each from `node_info`.

Report a summary table after all nodes are up, grouped by tier:

```
All <total> nodes are Running!

  FRONTEND LOAD BALANCER:
    [1] <name>  | ID: <id> | Public: <ip>

  FRONTEND (<count> instance(s)):
    [1] <name>  | ID: <id> | Public: <ip>
    [2] <name>  | ID: <id> | Public: <ip>
    ...

  BACKEND LOAD BALANCER:
    [1] <name>  | ID: <id> | Public: <ip>

  BACKEND (<count> instance(s)):
    [1] <name>  | ID: <id> | Public: <ip>
    ...

  DATABASE (<count> instance(s)):
    [1] <name>  | ID: <id> | Public: <ip>
    ...
```

---

## Phase 3 — Install Tools & Configure Tiers

If the user chose **SSH installation**, SSH into each node and run the installation commands.
If the user chose **start scripts**, this phase was already handled in Phase 2 (scripts run at boot). Skip to Phase 4.

For SSH, use: `ssh -o StrictHostKeyChecking=accept-new root@<PUBLIC_IP> '<commands>'`

**IMPORTANT**: After node shows `Running`, wait 30 seconds before first SSH attempt. If SSH fails, retry up to 3 times with 15-second gaps (the node may still be booting).

**Installation order** (must follow — each step depends on the previous):
1. Step 3.1 — Install Database (all instances)
2. Step 3.2 — Install Backend nodes (all instances) — needs DB IP & password
3. Step 3.0a — Configure Backend Load Balancer — needs backend node IPs
4. Step 3.3 — Install Frontend nodes (all instances) — needs backend LB IP
5. Step 3.0b — Configure Frontend Load Balancer — needs frontend node IPs

**Multi-instance behaviour**:
- **Database tier**: Install and configure on **all** database instances. The first instance is the primary; additional instances are standbys (inform the user that replication must be configured separately if needed). The backend nodes point to the **first** (primary) database instance's IP.
- **Backend tier**: Install on **all** backend instances with identical configuration. All instances point to the same database IP.
- **Frontend tier**: Install on **all** frontend instances with identical configuration. All instances use the **backend LB IP** as `BACKEND_HOST` (not a direct backend node IP).
- **Load balancers**: Configure after their respective app-tier nodes are known, so IPs can be added to the HAProxy backend pool.

### Step 3.1 — Install Database

SSH into the **database node** FIRST (other tiers depend on it).

**MySQL 8.0 (default):**
```bash
ssh -o StrictHostKeyChecking=no root@<DB_PUBLIC_IP> 'bash -s' <<'DBSCRIPT'
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y mysql-server

# Start and enable MySQL
systemctl start mysql
systemctl enable mysql

# On Ubuntu 24.04, fresh MySQL uses unix socket auth for root — must use sudo mysql for initial setup
MYSQL_ROOT_PASSWORD="E2E_three_tier_$(openssl rand -hex 8)"
sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '${MYSQL_ROOT_PASSWORD}';"
sudo mysql -e "CREATE USER 'appuser'@'%' IDENTIFIED WITH mysql_native_password BY '${MYSQL_ROOT_PASSWORD}';"
sudo mysql -e "CREATE DATABASE app_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
sudo mysql -e "GRANT ALL PRIVILEGES ON app_db.* TO 'appuser'@'%';"
sudo mysql -e "FLUSH PRIVILEGES;"

# Allow remote connections
sed -i 's/bind-address\s*=.*/bind-address = 0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
systemctl restart mysql

echo "MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}"
echo "DATABASE_SETUP_COMPLETE"
DBSCRIPT
```

**PostgreSQL (if chosen):**
```bash
ssh -o StrictHostKeyChecking=no root@<DB_PUBLIC_IP> 'bash -s' <<'DBSCRIPT'
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y postgresql postgresql-contrib

PG_PASSWORD="E2E_three_tier_$(openssl rand -hex 8)"
sudo -u postgres psql -c "ALTER USER postgres PASSWORD '${PG_PASSWORD}';"
sudo -u postgres psql -c "CREATE USER appuser WITH PASSWORD '${PG_PASSWORD}';"
sudo -u postgres psql -c "CREATE DATABASE app_db OWNER appuser;"

# Allow remote connections
PG_VERSION=$(ls /etc/postgresql/)
echo "host all all 0.0.0.0/0 md5" >> /etc/postgresql/${PG_VERSION}/main/pg_hba.conf
sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/${PG_VERSION}/main/postgresql.conf
systemctl restart postgresql

echo "PG_PASSWORD=${PG_PASSWORD}"
echo "DATABASE_SETUP_COMPLETE"
DBSCRIPT
```

**CRITICAL**: Capture the generated password from the SSH output before proceeding to Step 3.2. Run the DB script so its output is captured into a variable:

**MySQL:**
```bash
DB_PASSWORD=$(ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o LogLevel=ERROR root@<DB_PUBLIC_IP> 'bash -s' <<'DBSCRIPT'
... (MySQL install commands)
DBSCRIPT
)
DB_PASSWORD=$(echo "$DB_PASSWORD" | grep "MYSQL_ROOT_PASSWORD=" | cut -d= -f2)
```

**PostgreSQL:**
```bash
DB_OUTPUT=$(ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o LogLevel=ERROR root@<DB_PUBLIC_IP> 'bash -s' <<'DBSCRIPT'
... (PostgreSQL install commands)
DBSCRIPT
)
DB_PASSWORD=$(echo "$DB_OUTPUT" | grep "PG_PASSWORD=" | cut -d= -f2)
```

Save `DB_PASSWORD` and `DB_HOST` (the database node's public IP) as local shell variables — they will be substituted into the backend install script in Step 3.2.

### Step 3.2 — Install Backend

SSH into the **backend node** after database is ready.

Always set `DB_HOST` to the database node's **public IP**. Before running the SSH command, define these variables in your local shell — they will be substituted into the heredoc:

```bash
DB_HOST="<database_node_public_ip>"   # from Step 3.1
DB_PASSWORD="<captured_db_password>"  # captured from Step 3.1 output
```

**Python + Django (default):**
```bash
ssh -o StrictHostKeyChecking=no root@<BACKEND_PUBLIC_IP> 'bash -s' <<BACKENDSCRIPT
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y python3 python3-pip python3-venv pkg-config default-libmysqlclient-dev build-essential

# Create Django project
mkdir -p /opt/backend && cd /opt/backend
python3 -m venv venv
source venv/bin/activate

pip install django djangorestframework gunicorn mysqlclient django-cors-headers

django-admin startproject app_backend .
python manage.py startapp api

python3 <<'PYEOF'
import re

with open('/opt/backend/app_backend/settings.py', 'r') as f:
    content = f.read()

# Add imports
if "import os" not in content:
    content = "import os\n" + content

# Replace DATABASES
content = re.sub(
    r"DATABASES = \{[^}]*\{[^}]*\}[^}]*\}",
    """DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'app_db',
        'USER': 'appuser',
        'PASSWORD': os.environ.get('DB_PASSWORD', '${DB_PASSWORD}'),
        'HOST': os.environ.get('DB_HOST', '${DB_HOST}'),
        'PORT': '3306',
    }
}""",
    content,
    flags=re.DOTALL
)

# Add apps
content = content.replace(
    "'django.contrib.staticfiles',",
    "'django.contrib.staticfiles',\n    'rest_framework',\n    'corsheaders',\n    'api',"
)

# Add CORS middleware
content = content.replace(
    "'django.middleware.security.SecurityMiddleware',",
    "'corsheaders.middleware.CorsMiddleware',\n    'django.middleware.security.SecurityMiddleware',"
)

# Add CORS and ALLOWED_HOSTS settings
content += "\nCORS_ALLOW_ALL_ORIGINS = True\n"
content = content.replace("ALLOWED_HOSTS = []", "ALLOWED_HOSTS = ['*']")

with open('/opt/backend/app_backend/settings.py', 'w') as f:
    f.write(content)
PYEOF

# Create a sample API model, serializer, view, and URL
cat > /opt/backend/api/models.py <<'MODELS'
from django.db import models

class Item(models.Model):
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name
MODELS

cat > /opt/backend/api/serializers.py <<'SERIALIZERS'
from rest_framework import serializers
from .models import Item

class ItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = Item
        fields = '__all__'
SERIALIZERS

cat > /opt/backend/api/views.py <<'VIEWS'
from rest_framework import viewsets
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import Item
from .serializers import ItemSerializer

class ItemViewSet(viewsets.ModelViewSet):
    queryset = Item.objects.all()
    serializer_class = ItemSerializer

@api_view(['GET'])
def health_check(request):
    return Response({"status": "ok", "service": "backend"})
VIEWS

cat > /opt/backend/api/urls.py <<'APIURLS'
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'items', views.ItemViewSet)

urlpatterns = [
    path('', include(router.urls)),
    path('health/', views.health_check),
]
APIURLS

# Patch project urls.py
cat > /opt/backend/app_backend/urls.py <<'PROJURLS'
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
]
PROJURLS

# Run migrations
cd /opt/backend
source venv/bin/activate
python manage.py makemigrations api
python manage.py migrate

# Create systemd service for gunicorn
cat > /etc/systemd/system/backend.service <<'SERVICE'
[Unit]
Description=Django Backend (Gunicorn)
After=network.target

[Service]
User=root
WorkingDirectory=/opt/backend
Environment="DB_PASSWORD=${DB_PASSWORD}"
Environment="DB_HOST=${DB_HOST}"
ExecStart=/opt/backend/venv/bin/gunicorn app_backend.wsgi:application --bind 0.0.0.0:8000 --workers 3
Restart=always

[Install]
WantedBy=multi-user.target
SERVICE

systemctl daemon-reload
systemctl enable backend

# Wait for database to be reachable before starting
echo "Waiting for database to be ready..."
for i in $(seq 1 15); do
  if python3 -c "
import socket, sys
try:
    s = socket.create_connection(('${DB_HOST}', ${DB_PORT:-5432}), timeout=2)
    s.close()
    sys.exit(0)
except:
    sys.exit(1)
" 2>/dev/null; then
    echo "Database is reachable."
    break
  fi
  echo "  attempt $i/15 — not ready yet, waiting 2s..."
  sleep 2
done

systemctl start backend

echo "BACKEND_SETUP_COMPLETE"
BACKENDSCRIPT
```

**If the user chose PostgreSQL** (instead of MySQL), make these substitutions in the backend script above before running it:
1. In `apt-get install`: remove `default-libmysqlclient-dev`, add `libpq-dev`
2. In `pip install`: replace `mysqlclient` with `psycopg2-binary`
3. In the DATABASES patch: change `ENGINE` to `django.db.backends.postgresql` and `PORT` to `5432`
4. In the DB readiness check: change the port from `5432` to `5432` (already correct for PostgreSQL); for MySQL use `3306`
5. In the systemd service `Environment`: no changes needed — `DB_PASSWORD` and `DB_HOST` stay the same

### Step 3.0a — Configure Backend Load Balancer

SSH into the **backend-lb node** and install HAProxy. Configure it to round-robin across all backend nodes on port 8000.

Always use each backend node's **public IP** in the HAProxy backend pool.

The HAProxy frontend always binds to `0.0.0.0:8000` (all interfaces) so it can accept traffic from frontend nodes.

```bash
ssh -o StrictHostKeyChecking=no root@<BACKEND_LB_PUBLIC_IP> 'bash -s' <<BLBSCRIPT
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y haproxy

cat > /etc/haproxy/haproxy.cfg <<'HAPROXYCFG'
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4096
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend backend_lb_front
    bind *:8000
    default_backend backend_nodes

backend backend_nodes
    balance roundrobin
    option httpchk GET /api/health/
    http-check expect status 200
    server {location}-backend-1 <BACKEND_NODE_1_IP>:8000 check
    server {location}-backend-2 <BACKEND_NODE_2_IP>:8000 check
    # ... repeat for each backend node
HAPROXYCFG

systemctl enable haproxy
systemctl restart haproxy
echo "BACKEND_LB_SETUP_COMPLETE"
BLBSCRIPT
```

Replace `<BACKEND_NODE_N_IP>` with each backend node's **public IP**. Add one `server` line per backend instance.

Save `BACKEND_LB_IP` as the backend-lb's **public IP** (frontend nodes use this to reach the backend LB over the internet).

### Step 3.3 — Install Frontend

SSH into **every frontend node** after backend-lb is configured.

Always set `BACKEND_HOST` to the **backend LB's public IP** (`BACKEND_LB_IP`) — never a direct backend node IP.

**Nginx + Node.js (default):**
```bash
ssh -o StrictHostKeyChecking=no root@<FRONTEND_PUBLIC_IP> 'bash -s' <<FRONTENDSCRIPT
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y nginx curl

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs

# Create a simple frontend app
mkdir -p /opt/frontend
cd /opt/frontend

cat > /opt/frontend/package.json <<'PKGJSON'
{
  "name": "three-tier-frontend",
  "version": "1.0.0",
  "scripts": {
    "build": "echo 'Static build complete'"
  }
}
PKGJSON

# Create static HTML + JS that fetches from backend API
mkdir -p /var/www/html
cat > /var/www/html/index.html <<'HTMLEOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three-Tier App</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; background: #f5f5f5; padding: 2rem; }
        .container { max-width: 800px; margin: 0 auto; }
        h1 { color: #333; margin-bottom: 1rem; }
        .status { padding: 1rem; border-radius: 8px; margin-bottom: 1rem; }
        .status.ok { background: #d4edda; color: #155724; }
        .status.error { background: #f8d7da; color: #721c24; }
        .items { display: grid; gap: 1rem; margin-top: 1rem; }
        .item { background: white; padding: 1rem; border-radius: 8px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
        .item h3 { color: #333; } .item p { color: #666; margin-top: 0.5rem; }
        .item .price { color: #28a745; font-weight: bold; font-size: 1.1rem; }
        #loading { color: #666; font-style: italic; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Three-Tier Architecture</h1>
        <div id="backend-status" class="status">Checking backend...</div>
        <h2>Items from Database</h2>
        <div id="loading">Loading items...</div>
        <div id="items" class="items"></div>
    </div>
    <script>
        const BACKEND_URL = 'http://${BACKEND_HOST}:8000/api';

        async function checkHealth() {
            try {
                const res = await fetch(BACKEND_URL + '/health/');
                const data = await res.json();
                document.getElementById('backend-status').className = 'status ok';
                document.getElementById('backend-status').textContent = 'Backend: Connected (' + data.status + ')';
            } catch (e) {
                document.getElementById('backend-status').className = 'status error';
                document.getElementById('backend-status').textContent = 'Backend: Disconnected - ' + e.message;
            }
        }

        async function loadItems() {
            try {
                const res = await fetch(BACKEND_URL + '/items/');
                const items = await res.json();
                document.getElementById('loading').style.display = 'none';
                const container = document.getElementById('items');
                if (items.length === 0) {
                    container.innerHTML = '<p style="color:#666">No items yet. Run "add dummy data" to populate.</p>';
                    return;
                }
                container.innerHTML = items.map(item =>
                    '<div class="item"><h3>' + item.name + '</h3><p>' + item.description + '</p><p class="price">Rs. ' + item.price + '</p></div>'
                ).join('');
            } catch (e) {
                document.getElementById('loading').textContent = 'Failed to load items: ' + e.message;
            }
        }

        checkHealth();
        loadItems();
    </script>
</body>
</html>
HTMLEOF

# Configure Nginx as reverse proxy + static server
cat > /etc/nginx/sites-available/default <<'NGINXCONF'
server {
    listen 80 default_server;
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files \$uri \$uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://${BACKEND_HOST}:8000/api/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}
NGINXCONF

nginx -t && systemctl restart nginx
systemctl enable nginx

echo "FRONTEND_SETUP_COMPLETE"
FRONTENDSCRIPT
```

### Step 3.0b — Configure Frontend Load Balancer

SSH into the **frontend-lb node** after backend-lb is configured.

```bash
ssh -o StrictHostKeyChecking=no root@<FRONTEND_LB_PUBLIC_IP> 'bash -s' <<FLBSCRIPT
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y haproxy

cat > /etc/haproxy/haproxy.cfg <<'HAPROXYCFG'
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4096
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend frontend_lb_front
    bind *:80
    default_backend frontend_nodes

backend frontend_nodes
    balance roundrobin
    option httpchk GET /
    http-check expect status 200
    server {location}-frontend-1 <FRONTEND_NODE_1_IP>:80 check
    server {location}-frontend-2 <FRONTEND_NODE_2_IP>:80 check
    # ... repeat for each frontend node
HAPROXYCFG

systemctl enable haproxy
systemctl restart haproxy
echo "FRONTEND_LB_SETUP_COMPLETE"
FLBSCRIPT
```

Replace `<FRONTEND_NODE_N_IP>` with each frontend node's **public IP**. Add one `server` line per frontend instance.

---

## Phase 4 — Report Final Summary

After all tiers are installed and configured:

```
Three-Tier Architecture Deployed Successfully!

INFRASTRUCTURE:
  Frontend Load Balancer | HAProxy (port 80):
    <frontend-lb-name> | <frontend_lb_public_ip>  ← entry point for users

  Frontend  (<N> instance(s)) | Nginx + Node.js 20:
    [1] <name> | <public_ip>
    [2] <name> | <public_ip>   (if applicable)
    ...

  Backend Load Balancer | HAProxy (port 8000):
    <backend-lb-name> | <backend_lb_public_ip>  ← receives traffic from frontend nodes

  Backend   (<N> instance(s)) | Python + Django + Gunicorn (port 8000):
    [1] <name> | <public_ip>
    [2] <name> | <public_ip>   (if applicable)
    ...

  Database  (<N> instance(s)) | MySQL 8.0:
    [1] <name> | <public_ip>  ← primary (backend connects here)
    [2] <name> | <public_ip>  ← standby (if applicable)
    ...

TRAFFIC FLOW:
  Internet → Frontend LB (<frontend_lb_public_ip>:80)
           → Frontend nodes (round-robin)
           → Backend LB (<backend_lb_ip>:8000)
           → Backend nodes (round-robin)
           → Database (<first_db_ip>:3306)

ACCESS:
  App URL          : http://<frontend_lb_public_ip>        ← use this URL
  Backend API      : http://<backend_lb_public_ip>:8000/api/
  SSH Frontend LB  : ssh root@<frontend_lb_public_ip>
  SSH Backend LB   : ssh root@<backend_lb_public_ip>
  SSH Frontend     : ssh root@<frontend_public_ip>  (repeat for each instance)
  SSH Backend      : ssh root@<backend_public_ip>   (repeat for each instance)
  SSH Database     : ssh root@<db_public_ip>        (repeat for each instance)

DATABASE CREDENTIALS:
  Host     : <first_db_ip>
  Database : app_db
  User     : appuser
  Password : <generated_password>

NOTE: Additional database instances are standbys only —
replication must be configured separately if required.

To add sample data, say: "add dummy data"
```

---

## Phase 5 — Add Dummy Data (On-Demand)

When the user says **"add dummy data"**, SSH into the **backend node** and seed framework-specific data.

**Django + MySQL/PostgreSQL:**
```bash
ssh -o StrictHostKeyChecking=no root@<BACKEND_PUBLIC_IP> 'bash -s' <<'SEEDSCRIPT'
cd /opt/backend
source venv/bin/activate

python manage.py shell <<'PYEOF'
from api.models import Item

items = [
    Item(name="Cloud Server - Basic", description="2 vCPUs, 4GB RAM, 50GB SSD. Ideal for small web apps and dev environments.", price=1200.00),
    Item(name="Cloud Server - Standard", description="4 vCPUs, 8GB RAM, 100GB SSD. Perfect for production workloads and APIs.", price=2263.00),
    Item(name="Cloud Server - Premium", description="8 vCPUs, 16GB RAM, 200GB SSD. High-performance computing and databases.", price=4500.00),
    Item(name="Managed MySQL Database", description="Fully managed MySQL 8.0 instance with automated backups and monitoring.", price=3500.00),
    Item(name="Object Storage - 100GB", description="S3-compatible object storage for static assets, backups, and media files.", price=500.00),
    Item(name="CDN Bandwidth - 1TB", description="Content delivery network with global edge locations for fast content serving.", price=750.00),
    Item(name="Load Balancer", description="Layer 4/7 load balancer with health checks and SSL termination.", price=1500.00),
    Item(name="Kubernetes Cluster - Starter", description="Managed K8s with 3 worker nodes. Container orchestration made simple.", price=5000.00),
    Item(name="SSL Certificate - Wildcard", description="Wildcard SSL certificate for all subdomains. Auto-renewal included.", price=200.00),
    Item(name="Disaster Recovery Plan", description="Automated DR with hourly snapshots, cross-region replication, and 99.9% SLA.", price=8000.00),
]

Item.objects.bulk_create(items)
print(f"Seeded {len(items)} items into the database.")
PYEOF

echo "DUMMY_DATA_ADDED"
SEEDSCRIPT
```

After seeding, confirm:
```
Dummy data added! 10 items seeded into the database.

Visit http://<frontend_lb_public_ip> to see the data served through all tiers:
  Frontend LB --> Frontend nodes (Nginx) --> Backend LB --> Backend nodes (Django API) --> Database (MySQL)

API endpoint: http://<backend_lb_public_ip>:8000/api/items/
```

---

## Adapting to Other Stacks

The scripts above are defaults. If the user specifies different tools, adapt accordingly:

### Frontend Alternatives
- **React**: Install Node.js, `npx create-react-app`, build, serve via Nginx
- **Vue**: Install Node.js, `npm create vue@latest`, build, serve via Nginx
- **Angular**: Install Node.js, `npm install -g @angular/cli`, `ng new`, build, serve via Nginx
- **Plain HTML**: Just write HTML to `/var/www/html/`, configure Nginx

### Backend Alternatives
- **Node.js + Express**: Install Node.js, create Express app, use `mysql2` or `pg` driver, run with PM2
- **Flask**: Install Python, Flask, flask-cors, gunicorn; similar DB config
- **FastAPI**: Install Python, FastAPI, uvicorn, SQLAlchemy; adapt DB connection
- **Go + Gin**: Install Go, create Gin app, use `go-sql-driver/mysql`

### Database Alternatives
- **PostgreSQL**: Use `psycopg2-binary` in Django, change engine to `django.db.backends.postgresql`, port `5432`
- **MariaDB**: Same as MySQL install, use `mariadb-server` package
- **MongoDB**: Install `mongod`, use `djongo` or `pymongo` in backend

### Dummy Data Adaptation
When seeding data for non-Django backends, adapt the seed script to match the ORM/framework:
- **Express + Sequelize**: Create a `seed.js` script
- **Flask + SQLAlchemy**: Create a `seed.py` script
- **FastAPI + SQLAlchemy**: Similar to Flask
- Always create the same conceptual data: a list of product/service items with name, description, and price

---

## Error Handling

| Error | Fix |
|---|---|
| Node creation fails | Report API error, stop |
| SSH key upload fails | Show error, ask user for an existing key |
| Volume not available | Show attached node from `vm_detail`, ask user to pick another |
| No available volumes | Inform user, ask if continue without volume |
| Start script not found | Show available scripts, ask to pick or create new |
| Start script create fails | Show error, ask user to try again or skip |
| SSH connection refused | Wait 30s, retry up to 3 times |
| Package install fails | Report error, suggest manual fix |
| DB connection refused from backend | Check security group allows port 3306/5432, check bind-address |
| Frontend can't reach backend LB | Check security group allows port 8000 on backend-lb, verify `BACKEND_HOST` = backend LB IP |
| Backend LB can't reach backend nodes | Check security group allows port 8000 on backend nodes, verify HAProxy server IPs are correct |
| Frontend LB can't reach frontend nodes | Check security group allows port 80 on frontend nodes, verify HAProxy server IPs are correct |
| HAProxy not balancing | SSH into LB, run `haproxy -f /etc/haproxy/haproxy.cfg -c` to validate config |
| Migration fails | Check DB credentials, verify DB is running |
