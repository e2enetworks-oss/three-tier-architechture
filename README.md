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

You can find these in the [E2E Networks console](https://myaccount.e2enetworks.com) in Sidebar under API Section.

## Skills

### `/launch-node`
Launch a single VM/node on E2E Networks.

```
/launch-node Ubuntu 24.04, 8GB RAM, Delhi
```

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

### `/create-dr-plan`
Create a Disaster Recovery plan for one or more existing nodes.

```
/create-dr-plan my-frontend-node, my-backend-node
```

### `/three-tier-dr`
Create DR plans for a full three-tier setup and configure load-balanced standby nodes via Nginx.

```
/three-tier-dr
```

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
