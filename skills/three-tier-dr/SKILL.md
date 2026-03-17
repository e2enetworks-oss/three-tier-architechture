---
name: three-tier-dr
description: Create DR plans for a three-tier architecture (frontend, backend, database nodes), then launch new LB nodes and configure Nginx on them to load balance across their respective tier nodes.
disable-model-invocation: true
allowed-tools: Bash, ToolSearch, AskUserQuestion
argument-hint: "[frontend: fe-1,fe-2 | backend: be-1,be-2 | database: db-1 | target: Chennai]"
---

# Three-Tier Architecture DR Creation on E2E Networks

You are creating Disaster Recovery plans for a three-tier architecture on E2E Networks, then launching new load balancer nodes and configuring Nginx on them to distribute traffic across their respective tier nodes.

## Execution Rules

- **Proceed autonomously** — do NOT ask for confirmation between steps.
- **Apply all defaults silently** — use defaults without asking.
- **Only pause for user input in these two cases**:
  1. Required credentials or node inputs are completely missing and cannot be resolved
  2. A hard error occurs (API returns non-200, SSH fails after all retries, node not found)

---

## Phase 1 — Collect All Inputs

### Step 1.0 — Check for saved credentials

```bash
cat ~/.e2e/config.json 2>/dev/null
```

Load credentials using Python:
```python
import json, os
d = json.load(open(os.path.expanduser('~/.e2e/config.json')))
API_TOKEN = d['api_token']
API_KEY   = d['api_key']
PROJECT_ID = d['project_id']
LOCATION  = d.get('location', 'Delhi')
```

### Step 1.1 — Extract Target Location from arguments

Parse `TARGET_LOCATION` from the user-provided arguments (e.g. `target: Chennai`). If not provided in the arguments and not resolvable, ask for it before proceeding.

If credentials are missing, ask the user for everything in a **single message**:

```
To create the three-tier DR setup on E2E Networks, I need:

CREDENTIALS (required):
  1. API Token       : <your token>
  2. API Key         : <your API key>
  3. Project ID      : <your project ID>

TIER NODES — DR plans will be created for these:
  4. Frontend nodes  : <names or IDs, comma-separated>
  5. Backend nodes   : <names or IDs, comma-separated>
  6. Database node   : <name or ID>

DR CONFIGURATION:
  7. Target Location : <Chennai, Mumbai, Delhi-NCR-2>  ← REQUIRED, no default
  8. RPO (hours)     : <1–24, default: 3>
  9. Retention (days): <1–30, default: 7>

LB NODES — will be created at the target location automatically:
  (Optional) Frontend LB Name : <default: frontend-lb>
  (Optional) Backend LB Name  : <default: backend-lb>
```

### Defaults (apply silently)

| Field | Default |
|---|---|
| Source Location | from `~/.e2e/config.json` or Delhi |
| RPO hours | 3 |
| Retention days | 7 |
| Frontend LB name | `frontend-lb` |
| Backend LB name | `backend-lb` |
| LB OS | Same OS/plan as the first resolved frontend/backend node respectively |

---

## Phase 2 — Resolve All Tier Node IDs and IPs

Fetch all nodes from the API and resolve every name/ID the user provided:

```python
import subprocess, json

def get_all_nodes(api_key, project_id, location, token):
    r = subprocess.run([
        "curl", "-s",
        f"https://api.e2enetworks.com/myaccount/api/v1/nodes/?apikey={api_key}&project_id={project_id}&location={location}",
        "-H", f"Authorization: Bearer {token}"
    ], capture_output=True, text=True)
    return json.loads(r.stdout).get("data", [])

all_nodes = get_all_nodes(API_KEY, PROJECT_ID, LOCATION, API_TOKEN)

def resolve_node(inp, all_nodes):
    inp = inp.strip()
    if inp.isdigit():
        return next((n for n in all_nodes if str(n["id"]) == inp), None)
    return next((n for n in all_nodes if n["name"].lower() == inp.lower()), None)

frontend_nodes = [resolve_node(n, all_nodes) for n in FRONTEND_INPUTS]
backend_nodes  = [resolve_node(n, all_nodes) for n in BACKEND_INPUTS]
database_node  = resolve_node(DATABASE_INPUT, all_nodes)

# Warn on unresolved
if not database_node:
    print("ERROR: Could not resolve database node. Available nodes:")
    for n in all_nodes:
        print(f"  {n['id']:8}  {n['name']}")
    exit(1)

frontend_nodes = [n for n in frontend_nodes if n]
backend_nodes  = [n for n in backend_nodes  if n]

if not frontend_nodes:
    print("ERROR: No frontend nodes could be resolved.")
    exit(1)
if not backend_nodes:
    print("ERROR: No backend nodes could be resolved.")
    exit(1)
```

Print a resolution summary:
```
Resolved nodes:
  Frontend  : frontend-1 (299738), frontend-2 (299739)
  Backend   : backend-1  (299741), backend-2  (299742)
  Database  : database-1 (299743)
```

---

## Phase 3 — Create DR Plans for All Tier Nodes

Create one DR plan per node for **frontend, backend, and database nodes only**.

```
POST https://api.e2enetworks.com/myaccount/api/v1/draas/create-plan/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
Authorization: Bearer {API_TOKEN}
Content-Type: application/json
```

Payload per node:
```json
{
    "plan_name": "{SOURCE_LOCATION}_DR_{NodeName}_{timestamp}",
    "target_location": "{TARGET_LOCATION}",
    "resource_type": "Node",
    "source_resource_id": {NODE_ID},
    "rpo_hours": {RPO_HOURS},
    "recovery_point_retention_days": "{RETENTION_DAYS}",
    "volumes": []
}
```

**IMPORTANT**: `recovery_point_retention_days` must be a **string** (`"7"` not `7`).

```python
import time

def create_dr_plan(api_key, project_id, location, token, payload):
    r = subprocess.run([
        "curl", "-s", "-X", "POST",
        f"https://api.e2enetworks.com/myaccount/api/v1/draas/create-plan/?apikey={api_key}&project_id={project_id}&location={location}",
        "-H", f"Authorization: Bearer {token}",
        "-H", "Content-Type: application/json",
        "-d", json.dumps(payload)
    ], capture_output=True, text=True)
    return json.loads(r.stdout)

all_tier_nodes = (
    [("frontend", n) for n in frontend_nodes] +
    [("backend",  n) for n in backend_nodes] +
    [("database", database_node)]
)

dr_results = []
for tier, node in all_tier_nodes:
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
    code  = resp.get("code", "?")
    msg   = resp.get("message", "")
    dr_id = resp.get("data", {}).get("dr_id", "") if isinstance(resp.get("data"), dict) else ""
    status = "SUCCESS" if code in (200, 201) else "FAILED"
    print(f"  [{tier.upper():8}] [{status}] {node['name']} → '{plan_name}' | dr_id={dr_id} | {msg}")
    dr_results.append({
        "tier": tier, "node": node["name"], "node_id": node["id"],
        "plan_name": plan_name, "dr_id": dr_id,
        "status": status, "message": msg
    })
    time.sleep(1)
```

Print interim DR summary before moving to LB creation:
```
DR Plans Created:
  [FRONTEND] frontend-1 → Delhi_DR_frontend-1_xxx  SUCCESS  (dr_id=251)
  [FRONTEND] frontend-2 → Delhi_DR_frontend-2_xxx  SUCCESS  (dr_id=252)
  [BACKEND]  backend-1  → Delhi_DR_backend-1_xxx   SUCCESS  (dr_id=253)
  [BACKEND]  backend-2  → Delhi_DR_backend-2_xxx   SUCCESS  (dr_id=254)
  [DATABASE] database-1 → Delhi_DR_database-1_xxx  SUCCESS  (dr_id=255)

Now launching load balancer nodes...
```

---

## Phase 4 — Launch New Load Balancer Nodes at Target Location

Create two new nodes — one for frontend LB and one for backend LB — **at the TARGET location** (not the source location). Use `TARGET_LOCATION` for all API calls in this phase.

### Step 4.1 — Resolve plan and image at target location

Use the same plan/image as the first frontend node (for frontend LB) and first backend node (for backend LB). Fetch from the images API **at the target location**:

```
GET https://api.e2enetworks.com/myaccount/api/v1/images/os-category/?active=true&apikey={API_KEY}&project_id={PROJECT_ID}&location={TARGET_LOCATION}
```

Then:
```
GET https://api.e2enetworks.com/myaccount/api/v1/images/?display_category={CATEGORY}&category={OS}&osversion={VERSION}&gpu_type=&ng_container=&os={OS}&apikey={API_KEY}&project_id={PROJECT_ID}&location={TARGET_LOCATION}
```

Pick the plan SKU that matches the source node's plan. If you can't determine the exact plan, use the smallest available.

### Step 4.2 — Get security group and SSH key at target location

```
GET https://api.e2enetworks.com/myaccount/api/v1/security_group/?apikey={API_KEY}&project_id={PROJECT_ID}&location={TARGET_LOCATION}
GET https://api.e2enetworks.com/myaccount/api/v1/ssh_keys/?image={IMAGE}&apikey={API_KEY}&location={TARGET_LOCATION}
```

Use the security group where `is_default: true`. Use the first available SSH key — extract the `ssh_key` field (the full `ssh-rsa ...` string), not the `pk`.

### Step 4.3 — Create frontend LB node at target location

```python
def create_node(api_key, project_id, location, token, body):
    r = subprocess.run([
        "curl", "-s", "-X", "POST",
        f"https://api.e2enetworks.com/myaccount/api/v1/nodes/?apikey={api_key}&project_id={project_id}&location={location}",
        "-H", f"Authorization: Bearer {token}",
        "-H", "Content-Type: application/json",
        "-d", json.dumps(body)
    ], capture_output=True, text=True)
    return json.loads(r.stdout)

fe_lb_body = {
    "name": FRONTEND_LB_NAME,        # default: "frontend-lb"
    "plan": PLAN_SKU,
    "image": IMAGE,
    "region": "ncr",
    "ssh_keys": [SSH_KEY_VALUE],
    "security_group_id": SG_ID,
    "label": "default",
    "is_private": False,
    "disable_password": True,
    "backups": False,
    "enable_bitninja": False,
    "is_saved_image": False,
    "saved_image_template_id": None,
    "start_scripts": [],
    "default_public_ip": False,
    "ngc_container_id": None,
    "number_of_instances": 1
}
# NOTE: pass TARGET_LOCATION, not LOCATION
resp = create_node(API_KEY, PROJECT_ID, TARGET_LOCATION, API_TOKEN, fe_lb_body)
print(f"Frontend LB node creation: code={resp.get('code')} message={resp.get('message')}")
```

Do the same for backend LB node with `BACKEND_LB_NAME`, also passing `TARGET_LOCATION`.

### Step 4.4 — Resolve new LB node IDs at target location

The POST response may not include the node ID. Match by name from the nodes list **at the target location**:

```python
time.sleep(5)  # brief wait before listing
all_nodes_updated = get_all_nodes(API_KEY, PROJECT_ID, TARGET_LOCATION, API_TOKEN)
fe_lb_node = next((n for n in all_nodes_updated if n["name"].lower() == FRONTEND_LB_NAME.lower()), None)
be_lb_node = next((n for n in all_nodes_updated if n["name"].lower() == BACKEND_LB_NAME.lower()), None)
```

### Step 4.5 — Poll until both LB nodes are Running (using target location)

```python
def get_node(node_id, api_key, project_id, location, token):
    r = subprocess.run([
        "curl", "-s",
        f"https://api.e2enetworks.com/myaccount/api/v1/nodes/{node_id}/?apikey={api_key}&project_id={project_id}&location={location}",
        "-H", f"Authorization: Bearer {token}"
    ], capture_output=True, text=True)
    return json.loads(r.stdout).get("data", {})

lb_nodes = [("frontend-lb", fe_lb_node["id"]), ("backend-lb", be_lb_node["id"])]

for poll in range(1, 21):
    print(f"Poll {poll}/20")
    all_running = True
    for name, nid in lb_nodes:
        # NOTE: poll at TARGET_LOCATION
        n = get_node(nid, API_KEY, PROJECT_ID, TARGET_LOCATION, API_TOKEN)
        status = n.get("status", "?")
        pub  = n.get("public_ip_address") or "pending"
        priv = n.get("private_ip_address") or "pending"
        print(f"  {name}: {status} | pub={pub} | priv={priv}")
        if status != "Running":
            all_running = False
        else:
            # Capture IPs once Running
            if name == "frontend-lb":
                fe_lb_public_ip = pub
            else:
                be_lb_public_ip = pub
    if all_running:
        print("Both LB nodes are Running!")
        break
    if poll < 20:
        time.sleep(20)
```

---

## Phase 5 — Configure Nginx on LB Nodes via SSH

### Step 5.0 — Resolve DR target frontend and backend nodes at target location

The LB nodes must proxy to the **DR replica nodes at the target location**, not the source nodes. Fetch all nodes at `TARGET_LOCATION` and match by the same names as the source frontend/backend nodes:

```python
target_nodes = get_all_nodes(API_KEY, PROJECT_ID, TARGET_LOCATION, API_TOKEN)

# Match by same name as source nodes
target_frontend_nodes = [
    next((n for n in target_nodes if n["name"].lower() == src["name"].lower()), None)
    for src in frontend_nodes
]
target_backend_nodes = [
    next((n for n in target_nodes if n["name"].lower() == src["name"].lower()), None)
    for src in backend_nodes
]

# Warn and skip unresolved
for src, tgt in zip(frontend_nodes, target_frontend_nodes):
    if tgt is None:
        print(f"  WARN: No target-location node found matching frontend '{src['name']}' — skipping from upstream")
for src, tgt in zip(backend_nodes, target_backend_nodes):
    if tgt is None:
        print(f"  WARN: No target-location node found matching backend '{src['name']}' — skipping from upstream")

target_frontend_nodes = [n for n in target_frontend_nodes if n]
target_backend_nodes  = [n for n in target_backend_nodes  if n]

if not target_frontend_nodes:
    print("ERROR: No frontend DR target nodes found at target location. Cannot configure Frontend LB.")
if not target_backend_nodes:
    print("ERROR: No backend DR target nodes found at target location. Cannot configure Backend LB.")

print("DR target nodes resolved at %s:" % TARGET_LOCATION)
for n in target_frontend_nodes:
    print(f"  [FRONTEND] {n['name']} | private={n.get('private_ip_address')} | public={n.get('public_ip_address')}")
for n in target_backend_nodes:
    print(f"  [BACKEND]  {n['name']} | private={n.get('private_ip_address')} | public={n.get('public_ip_address')}")
```

### Step 5.1 — Clear stale SSH known_hosts entries

```python
for ip in [fe_lb_public_ip, be_lb_public_ip]:
    subprocess.run(["ssh-keygen", "-R", ip], capture_output=True)
```

Wait 30 seconds after Running before first SSH attempt.

### Step 5.2 — SSH retry wrapper

```python
def wait_for_ssh(ip, retries=3, wait=15):
    for attempt in range(1, retries + 1):
        r = subprocess.run([
            "ssh", "-o", "StrictHostKeyChecking=no",
            "-o", "ConnectTimeout=15",
            f"root@{ip}", "echo ok"
        ], capture_output=True, text=True)
        if r.returncode == 0:
            return True
        print(f"  SSH attempt {attempt}/{retries} to {ip} failed, waiting {wait}s...")
        time.sleep(wait)
    return False
```

### Step 5.3 — Build Nginx upstream config

```python
def build_upstream_block(pool_name, ips, port):
    servers = "\n".join(f"    server {ip}:{port};" for ip in ips)
    return f"upstream {pool_name} {{\n{servers}\n}}"
```

### Step 5.4 — Configure Frontend LB

Use public IPs of the **DR target frontend nodes at target location**. Fall back to private IP if public is unavailable:

```python
frontend_target_ips = [
    n.get("public_ip_address") or n.get("private_ip_address")
    for n in target_frontend_nodes
    if n.get("public_ip_address") or n.get("private_ip_address")
]

fe_upstream = build_upstream_block("frontend_pool", frontend_target_ips, 80)

fe_nginx_conf = f"""{fe_upstream}

server {{
    listen 80 default_server;
    server_name _;

    location / {{
        proxy_pass http://frontend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }}
}}"""

fe_script = f"""export DEBIAN_FRONTEND=noninteractive
apt-get update -y -q
apt-get install -y -q nginx
cat > /etc/nginx/sites-available/default << 'NGINXEOF'
{fe_nginx_conf}
NGINXEOF
nginx -t && systemctl restart nginx && systemctl enable nginx
echo "FRONTEND_LB_CONFIGURED"
"""

time.sleep(30)  # wait for node to fully boot after Running
if wait_for_ssh(fe_lb_public_ip):
    r = subprocess.run([
        "ssh", "-o", "StrictHostKeyChecking=no",
        f"root@{fe_lb_public_ip}", fe_script
    ], capture_output=True, text=True)
    if "FRONTEND_LB_CONFIGURED" in r.stdout:
        print(f"  [SUCCESS] Frontend LB configured → proxying to {frontend_target_ips} on port 80")
    else:
        print(f"  [FAILED]  Frontend LB Nginx config failed")
        print(r.stdout[-500:])
        print(r.stderr[-300:])
else:
    print(f"  [FAILED]  Could not SSH into Frontend LB at {fe_lb_public_ip}")
```

### Step 5.5 — Configure Backend LB

Use public IPs of the **DR target backend nodes at target location**. Fall back to private IP if public is unavailable:

```python
backend_target_ips = [
    n.get("public_ip_address") or n.get("private_ip_address")
    for n in target_backend_nodes
    if n.get("public_ip_address") or n.get("private_ip_address")
]

be_upstream = build_upstream_block("backend_pool", backend_target_ips, 8000)

be_nginx_conf = f"""{be_upstream}

server {{
    listen 80 default_server;
    server_name _;

    location / {{
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }}
}}"""

be_script = f"""export DEBIAN_FRONTEND=noninteractive
apt-get update -y -q
apt-get install -y -q nginx
cat > /etc/nginx/sites-available/default << 'NGINXEOF'
{be_nginx_conf}
NGINXEOF
nginx -t && systemctl restart nginx && systemctl enable nginx
echo "BACKEND_LB_CONFIGURED"
"""

if wait_for_ssh(be_lb_public_ip):
    r = subprocess.run([
        "ssh", "-o", "StrictHostKeyChecking=no",
        f"root@{be_lb_public_ip}", be_script
    ], capture_output=True, text=True)
    if "BACKEND_LB_CONFIGURED" in r.stdout:
        print(f"  [SUCCESS] Backend LB configured → proxying to {backend_target_ips} on port 8000")
    else:
        print(f"  [FAILED]  Backend LB Nginx config failed")
        print(r.stdout[-500:])
        print(r.stderr[-300:])
else:
    print(f"  [FAILED]  Could not SSH into Backend LB at {be_lb_public_ip}")
```

---

## Phase 6 — Final Summary

```
Three-Tier DR Setup Complete!

DR PLANS CREATED:
  Tier       | Node         | Plan Name                           | DR ID | Status
  -----------|--------------|--------------------------------------|-------|--------
  Frontend   | frontend-1   | Delhi_DR_frontend-1_1710000000      | 251   | SUCCESS
  Frontend   | frontend-2   | Delhi_DR_frontend-2_1710000001      | 252   | SUCCESS
  Backend    | backend-1    | Delhi_DR_backend-1_1710000002       | 253   | SUCCESS
  Backend    | backend-2    | Delhi_DR_backend-2_1710000003       | 254   | SUCCESS
  Database   | database-1   | Delhi_DR_database-1_1710000004      | 255   | SUCCESS

LOAD BALANCER NODES CREATED & CONFIGURED (at target location: Chennai):
  Frontend LB : frontend-lb | Public: <fe_lb_public_ip> → proxying to DR target frontend nodes [<fe-1 target_ip>, <fe-2 target_ip>] :80
  Backend LB  : backend-lb  | Public: <be_lb_public_ip> → proxying to DR target backend nodes  [<be-1 target_ip>, <be-2 target_ip>] :8000

DR CONFIGURATION:
  Source Location  : Delhi
  Target Location  : Chennai
  RPO              : 3 hours
  Retention        : 7 days
  Resource Type    : Node
```

If any step failed, include the error in the respective row.

---

## Error Handling

| Error | Fix |
|---|---|
| `code != 200/201` on DR create | Print full API response, mark as FAILED, continue with remaining nodes |
| Node name not found | Warn and list all available node names |
| LB node creation fails | Print API error and stop — LB setup cannot proceed |
| LB node stuck in non-Running after 20 polls | Report as timed out, stop |
| SSH connection refused after retries | Report FAILED for that LB's Nginx config |
| `nginx -t` fails on LB | Print nginx error, skip `systemctl restart` to avoid breaking existing config |
| Private IP missing for a tier node | Warn and skip that node from the upstream pool |
| Credentials missing | Ask user in a single prompt before proceeding |
