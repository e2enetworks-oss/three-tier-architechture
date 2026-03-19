# E2E Networks — Claude Code Plugin

Automate E2E Networks cloud infrastructure using natural language in [Claude Code](https://claude.ai/code).

## Install

```bash
claude plugin install https://github.com/e2enetworks-oss/three-tier-architechture
```

## Setup credentials

Create `~/.e2e/config.json` with your E2E Networks credentials:

```json
{
  "api_token": "your-bearer-token",
  "api_key": "your-api-key",
  "project_id": "your-project-id",
  "location": "Delhi"
}
```

Set restrictive permissions to protect your credentials:

```bash
chmod 600 ~/.e2e/config.json
```

You can find these in the [E2E Networks console](https://myaccount.e2enetworks.com) in Sidebar under API Section.

## Skills

### `/three-tier-arch`
Deploy a complete three-tier architecture on E2E Networks with load balancers at every tier. Creates and configures all nodes automatically via SSH, wires the tiers together, and optionally seeds dummy data.

**What gets created:**
- 1 frontend load balancer (HAProxy, port 80)
- N frontend nodes (Nginx + app)
- 1 backend load balancer (HAProxy, port 8000)
- N backend nodes (Django/Flask/Express + app server)
- N database nodes (PostgreSQL/MySQL)

```
/three-tier-arch React frontend x2 with LB, Django backend x2 with LB, PostgreSQL db x1, Chennai
```

**Node naming** follows `{location}-{role}` — no number suffix when there is only one instance, numbered when there are multiple:

| Example | Name |
|---|---|
| Frontend LB (always 1) | `chennai-frontend-lb` |
| Frontend node x1 | `chennai-frontend` |
| Frontend node x2 | `chennai-frontend-1`, `chennai-frontend-2` |
| Backend LB (always 1) | `chennai-backend-lb` |
| Database x1 | `chennai-database` |

**Resilient provisioning**: if any node enters a failed state (`Failed`, `Failed recreate`, `Error`) during provisioning, a replacement node is automatically created and polled in its place — no manual intervention needed.

After deployment, say `add dummy data` to populate the database with sample records.

### `/three-tier-dr`
Create DR plans for a full three-tier setup and configure load-balanced standby nodes via Nginx.

```
/three-tier-dr
```

**What gets created:**
- DR plans for all **application nodes** (frontend, backend, database) — LB nodes are excluded since they are stateless and recreated fresh at the target location
- 1 new frontend LB at the target location — configured to proxy to the DR replica frontend nodes
- 1 new backend LB at the target location — configured to proxy to the DR replica backend nodes

**How block-level DR works — and what to fix manually after a drill or recovery:**

DR replication is block-level, meaning the entire disk of each node is synced to the target location. This preserves your application code and configuration exactly as it was — but it also means the IP addresses baked into those configs are the **source location IPs**, not the target location IPs.

After a DR drill or failover, the following connections will point to the **wrong location** and must be updated manually on each replica node:

| Node | Points to (wrong — source IP) | Should point to (target IP) |
|---|---|---|
| Frontend nodes | Source backend LB IP | Target backend LB IP |
| Backend nodes | Source database IP | Target database node IP |

The LBs created at the target location are correctly configured — they point to the DR replica frontend/backend nodes respectively. Only the app-tier nodes need manual reconfiguration.

**Steps after drill or recovery:**
1. SSH into each frontend replica node and update the backend host to the target backend LB's IP
2. SSH into each backend replica node and update the database host to the target database node's IP
3. Restart the relevant app processes (e.g. `systemctl restart gunicorn`, `systemctl restart nginx`)

## Supported stacks

| Tier | Default | Alternatives |
|---|---|---|
| Frontend | Nginx + Node.js 20 | React, Vue, Angular, plain HTML |
| Backend | Python 3 + Django + Gunicorn | Flask, FastAPI, Node.js + Express, Go + Gin |
| Database | MySQL 8.0 | PostgreSQL 16, MariaDB, MongoDB |

## Locations

| Location | Value to use |
|---|---|
| Delhi (default) | `Delhi` |
| Chennai | `Chennai` |
