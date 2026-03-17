---
name: launch-node
description: Launch a VM/node on E2E Networks. Use when the user wants to create, spin up, or launch a node/VM/instance on E2E Networks cloud.
disable-model-invocation: true
allowed-tools: Bash, WebFetch, ToolSearch, Read, Glob, Grep, AskUserQuestion
argument-hint: "[node specs e.g. 'ubuntu 24.04 8GB RAM named my-server']"
---

# Launch a Node on E2E Networks

You are executing the E2E Networks node launch workflow. Follow the **collect-then-execute** pattern strictly.

## Phase 1 — Collect All Inputs

If credentials or node specs are missing from `$ARGUMENTS`, ask the user for everything needed in a **single prompt**:

```
To launch a node on E2E Networks, I need the following:

CREDENTIALS (required):
  1. API Token     : <your token>
  2. API Key       : <your API key - different from token>
  3. Project ID    : <your project ID>

NODE CONFIGURATION:
  4. Node Name     : <name using only A-Z, a-z, 0-9, _, - (max 50 chars)>
  5. OS            : <e.g., Ubuntu, AlmaLinux, RockyLinux, Debian, CentOS, Windows>
  6. OS Version    : <e.g., 24.04, 22.04, 9, 12>
  7. Category      : <e.g., Linux Virtual Node, Linux Smart Dedicated Compute>
  8. Plan Size     : <e.g., "smallest", "8GB RAM", "C3.8GB", "16 vCPUs">
  9. SSH Public Key: <full ssh-rsa key string, or "use existing" if already uploaded>
  10. Location     : <Delhi (default), Mumbai, Delhi-NCR-2, Chennai>

OPTIONAL (defaults will be used if not specified):
  11. Backups           : true/false  (default: false)
  12. Enable BitNinja   : true/false  (default: false)
  13. Disable Password  : true/false  (default: true)
  14. Number of Nodes   : 1-100       (default: 1)
  15. IPv6              : true/false  (default: false)
```

### Input Sanitization (auto-fix, do NOT ask)

| Issue | Auto-fix |
|---|---|
| Node name has spaces | Replace with `-` |
| Node name has special chars | Remove non `A-Za-z0-9_-` |
| Node name exceeds 50 chars | Truncate to 50 |
| Node name empty after sanitization | Auto-generate: `{OS}-{Plan}-{random 3 digits}` |
| SSH key reference is a name/label | Fetch keys from API, fuzzy-match on label (case-insensitive, partial). If no match, STOP and show available keys |
| OS name wrong casing | Match case-insensitively |

### Defaults for Missing Fields

| Field | Default |
|---|---|
| Location | `Delhi` |
| OS | `Ubuntu` |
| OS Version | Latest LTS (e.g., `24.04`) |
| Category | `Linux Virtual Node` |
| Plan Size | Smallest available |
| Node Name | `{OS}-{Plan}-{random 3 digits}` |
| SSH Key | First existing key from account; if none, STOP and ask |
| Security Group | Auto-select `is_default: true` |

### Interpreting User Input

| User says | Interpreted as |
|---|---|
| "launch a node" / "create a VM" | Use all defaults |
| "Ubuntu 24.04" | OS=Ubuntu, Version=24.04 |
| "8GB RAM" or "8GB" | Match plan with ~8GB RAM |
| "4 vCPUs" | Match plan with 4 vCPUs |
| "C3.8GB" or "C3" | Match plan by name/series |
| "20GB RAM and 250GB storage" | Match closest to both RAM and disk |
| "smallest" / "cheapest" | Lowest specs plan |
| "largest" / "most powerful" | Highest specs plan |
| "dedicated" | Category=Linux Smart Dedicated Compute |
| "2 nodes" / "3 instances" | number_of_instances=2 or 3 |

---

## Phase 2 — Execute Without Stopping

Once inputs are collected, run all steps sequentially. **Do NOT pause for confirmation between steps.** Only stop on API errors.

### API Configuration

All requests use:
- Header: `Authorization: Bearer <API_TOKEN>`
- Header: `Content-Type: application/json`
- Query params: `apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}`
- Base URL: `https://api.e2enetworks.com`

Use `curl` via Bash for all API calls (WebFetch cannot send auth headers).

---

### Step 1 — Validate OS Selection

```
GET {BASE_URL}/myaccount/api/v1/images/os-category/?active=true&apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Logic:
1. Find OS family matching user's OS (case-insensitive on `OS` field)
2. Find matching version in that family's `version` array
3. Verify category exists in that family's `category` array
4. ON ERROR: STOP, show available options
5. ON SUCCESS: Save `os`, `version`, `display_category`, proceed silently

---

### Step 2 — Resolve Plan & Image

```
GET {BASE_URL}/myaccount/api/v1/images/?display_category={DISPLAY_CATEGORY}&category={OS}&osversion={VERSION}&gpu_type=&ng_container=&os={OS}&apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Plan matching logic:
- "C3.8GB" -> exact match on `name`
- "8GB RAM" -> match on `specs.ram` closest to 8
- "4 vCPUs" -> match on `specs.cpu` closest to 4
- "20GB RAM and 250GB storage" -> smallest plan where `specs.ram` >= 20 AND `specs.disk_space` >= 250
- "smallest" -> lowest `specs.cpu`
- No preference -> first plan (smallest)

ON ERROR: STOP, show available plans
ON SUCCESS: Save `plan` and `image` values, proceed silently

---

### Step 3 — Auto-Select Security Group

```
GET {BASE_URL}/myaccount/api/v1/security_group/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Find security group where `is_default` is `true`.
ON ERROR: STOP, list available groups
ON SUCCESS: Save `security_group_id`, proceed

---

### Step 4 — Resolve SSH Key

```
GET {BASE_URL}/myaccount/api/v1/ssh_keys/?image={IMAGE}&apikey={API_KEY}&location={LOCATION}
```

Logic:
1. If user provided full SSH public key string -> use directly, skip API call
2. If user referenced key by name/label -> fetch keys, fuzzy-match (case-insensitive, partial)
3. If "use existing" or no preference -> use first key from API
4. If no keys and no raw key -> STOP, ask user for SSH public key
ON SUCCESS: Save SSH key string(s) as list

---

### Step 5 — Create the Node

```
POST {BASE_URL}/myaccount/api/v1/nodes/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Request body:
```json
{
  "name": "<NODE_NAME>",
  "plan": "<PLAN_SKU from Step 2>",
  "image": "<IMAGE from Step 2>",
  "region": "ncr",
  "ssh_keys": ["<SSH_KEY from Step 4>"],
  "security_group_id": "<ID from Step 3>",
  "label": "default",
  "is_private": false,
  "disable_password": true,
  "backups": false,
  "enable_bitninja": false,
  "is_saved_image": false,
  "saved_image_template_id": null,
  "start_scripts": [],
  "reserve_ip": "",
  "is_ipv6_availed": false,
  "subnet_id": "",
  "default_public_ip": false,
  "ngc_container_id": null,
  "number_of_instances": 1
}
```

Override defaults with user-provided optional values.
ON ERROR: Report error, STOP
ON SUCCESS: Save `id`, proceed to polling

---

### Step 6 — Poll Until Running

```
GET {BASE_URL}/myaccount/api/v1/nodes/{NODE_ID}/?apikey={API_KEY}&project_id={PROJECT_ID}&location={LOCATION}
```

Poll every 20 seconds. Max 10 polls (~3.5 min). Stop when `status` is `Running` or `Failed`.

---

## Phase 3 — Report Results

Once node is `Running`, report:

```
Node launched successfully!

  Name       : <name>
  Node ID    : <id>
  Status     : Running
  Public IP  : <ip>
  Private IP : <ip>
  OS         : <os> <version>
  Plan       : <plan_name> (<vcpus> vCPUs, <ram> RAM, <disk> Disk)
  Price      : Rs. <per_hr>/hr (Rs. <per_mo>/mo)
  Location   : <location>

  SSH Access : ssh root@<public_ip>
```

---

## Error Handling

| Error | Cause | Fix |
|---|---|---|
| 401 Unauthorized | Invalid/expired token | Ask for valid token |
| 412 Precondition Failed | Validation error | Check `errors` field |
| 403 Forbidden | Account not approved / limit reached | Contact E2E support |
| 400 Bad Request | Malformed request | Verify JSON structure |
| OS/version not found | Unavailable OS | Show available options |
| Plan not found | Unavailable plan | Show available plans |
| No SSH keys | None on account | Ask user for key |
| No default security group | None with `is_default: true` | Show groups, ask user |
| Node stuck Creating | Provisioning delay | Wait 3.5 min, report timeout |
