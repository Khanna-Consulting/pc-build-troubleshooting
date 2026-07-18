# Mock Environments on kc-llm

Docker-based virtualized test environments on kc-llm for safe testing before touching production.

---

## Kubota — SQL Server 2022

**Status:** Running (set up 2026-07-18)  
**Location on server:** `/opt/mock-environments/kubota/`  
**Container:** `kubota-mssql`

### Connection

| Field    | Value              |
|----------|--------------------|
| Host     | kc-llm (100.112.124.101) |
| Port     | 1433               |
| User     | sa                 |
| Password | KubotaMock#2026!   |
| Database | KubotaMock         |

### Schema (placeholder — replace with real Kubota schema)

- **Equipment** — machines tracked by serial number, model, status
- **ServiceOrders** — work orders linked to equipment
- **Parts** — inventory with part numbers and pricing

### Commands

```bash
# Start
cd /opt/mock-environments/kubota && docker compose up -d

# Stop
docker compose down

# Connect via sqlcmd
docker exec kubota-mssql /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'KubotaMock#2026!' -C

# Run a script
docker exec kubota-mssql /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'KubotaMock#2026!' -C -i /init/your-script.sql

# Full reset (wipes data)
docker compose down -v && docker compose up -d
# Then re-run init scripts
```

### Workflow

1. Script schema from Kubota production
2. Drop into `./init/` as numbered `.sql` files
3. Reset container and run scripts
4. Test dashboards/agents against mock
5. Present at Wednesday change management meeting before any production deploy

---

## Kubota — Windows Server 2022 (Full VM)

**Status:** Running (set up 2026-07-18) — installing Windows, give it ~15 min  
**Location on server:** `/opt/mock-environments/kubota-windows/`  
**Container:** `kubota-windows`  
**Method:** dockurr/windows (QEMU/KVM inside Docker — nothing installed on host)

### RDP Connection

| Field    | Value              |
|----------|--------------------|
| Host     | 100.112.124.101    |
| Port     | 3390               |
| User     | Docker             |
| Password | KubotaMock#2026!   |

Open Microsoft Remote Desktop on your Mac → Add PC → `100.112.124.101:3390`

### SQL Server (after you install it inside Windows)

| Field    | Value              |
|----------|--------------------|
| Host     | 100.112.124.101    |
| Port     | 1434               |

### Resources Allocated

- 8GB RAM, 4 CPU cores
- Persistent storage in Docker volume `win-data`

### Commands

```bash
# Start
cd /opt/mock-environments/kubota-windows && docker compose up -d

# Stop
docker compose down

# Wipe and recreate (full reinstall)
docker compose down -v && docker compose up -d

# Check logs / install progress
docker logs kubota-windows

# Check if RDP is ready
docker logs kubota-windows 2>&1 | grep -i ready
```

### What to do once Windows is up

1. RDP in from your Mac (`100.112.124.101:3390`)
2. Install SQL Server (download inside the VM or mount an ISO)
3. Install SSMS
4. Load the Kubota schema
5. Test agents/dashboards against it
6. Present at Wednesday meeting before any production deploy

---

## General Rules

- **Never modify existing production systems** — build new, test here first
- **SQL Grease is read-only** — ~40 people consuming, don't touch
- **Grafana dashboards** — only create new, never modify existing
- **Docker on kc-llm** = safe sandbox for everything before it goes live
