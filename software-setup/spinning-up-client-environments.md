# Spinning Up Client Test Environments

How to create isolated mock environments for any client on kc-llm. Each client gets their own Docker container(s) — nothing touches the host OS, nothing touches other clients, nothing touches production.

---

## How It Works

```
/opt/mock-environments/
├── kubota/                  # SQL Server on Linux
│   ├── docker-compose.yml
│   ├── init/
│   │   ├── 01-create-schema.sql
│   │   └── 02-seed-data.sql
│   └── README.md
├── kubota-windows/          # Full Windows Server 2022 VM
│   ├── docker-compose.yml
│   └── README.md
├── client-b/                # Next client goes here
│   ├── docker-compose.yml
│   └── init/
└── client-c/
    └── ...
```

Each folder is self-contained. Each has its own Docker Compose file, its own volumes, its own ports. They run independently.

---

## Option A: SQL Server on Linux (Lightweight)

Best for: testing queries, schemas, stored procedures, agents, and dashboards against a client's data. Covers 90% of use cases.

### Steps

```bash
# 1. Create the folder
sudo mkdir -p /opt/mock-environments/<client-name>/init
sudo chown -R kc-llm:kc-llm /opt/mock-environments/<client-name>

# 2. Create docker-compose.yml
cat > /opt/mock-environments/<client-name>/docker-compose.yml << 'EOF'
services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: <client-name>-mssql
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: "<pick-a-strong-password>"
      MSSQL_PID: Developer
    ports:
      - "<pick-unused-port>:1433"
    volumes:
      - data:/var/opt/mssql
      - ./init:/init
    restart: unless-stopped
    mem_limit: 4g

volumes:
  data:
EOF

# 3. Add your schema script(s) to init/
# Copy from production or write from scratch

# 4. Start it
cd /opt/mock-environments/<client-name> && docker compose up -d

# 5. Wait ~15 seconds for SQL Server to boot, then load schema
docker exec <client-name>-mssql /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P '<your-password>' -C \
  -i /init/01-create-schema.sql

# 6. Load seed data
docker exec <client-name>-mssql /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P '<your-password>' -C \
  -i /init/02-seed-data.sql
```

### Port Allocation

| Client   | Port | Container Name    |
|----------|------|-------------------|
| Kubota   | 1433 | kubota-mssql      |
| (next)   | 1435 | clientb-mssql     |
| (next)   | 1436 | clientc-mssql     |

Pick any unused port. Check what's taken:
```bash
sudo ss -tlnp | grep LISTEN
```

### Connecting

From your Mac or any machine on Tailscale:
```
Host: 100.112.124.101
Port: <the port you picked>
User: sa
Password: <your-password>
```

Works with Azure Data Studio, DBeaver, SSMS, or any agent/script that speaks SQL.

---

## Option B: Full Windows Server 2022 (RDP Desktop)

Best for: when you need the full Windows experience — SSMS, AD auth, Windows services, SSIS, or a 1:1 match of what the client actually runs.

### Steps

```bash
# 1. Create the folder
sudo mkdir -p /opt/mock-environments/<client-name>-windows
sudo chown -R kc-llm:kc-llm /opt/mock-environments/<client-name>-windows

# 2. Create docker-compose.yml
cat > /opt/mock-environments/<client-name>-windows/docker-compose.yml << 'EOF'
services:
  windows:
    image: dockurr/windows
    container_name: <client-name>-windows
    environment:
      VERSION: "2022"
      RAM_SIZE: "8G"
      CPU_CORES: "4"
      USERNAME: "Docker"
      PASSWORD: "<pick-a-strong-password>"
    devices:
      - /dev/kvm
    ports:
      - "<rdp-port>:3389"
      - "<sql-port>:1433"
      - "<web-viewer-port>:8006"
    volumes:
      - win-data:/storage
    cap_add:
      - NET_ADMIN
    restart: unless-stopped

volumes:
  win-data:
EOF

# 3. Start it (first run downloads Windows ISO + installs, ~15 min)
cd /opt/mock-environments/<client-name>-windows && docker compose up -d

# 4. Watch progress
docker logs <client-name>-windows -f

# 5. Once "Windows started successfully" appears, wait 2-3 min for OOBE, then RDP in
```

### Port Allocation

| Client   | RDP Port | SQL Port | Web Viewer | Container Name     |
|----------|----------|----------|------------|--------------------|
| Kubota   | 3390     | 1434     | 8006       | kubota-windows     |
| (next)   | 3391     | 1437     | 8007       | clientb-windows    |
| (next)   | 3392     | 1438     | 8008       | clientc-windows    |

### Connecting via RDP

- **Mac:** Microsoft Remote Desktop → Add PC → `100.112.124.101:<rdp-port>`
- **Browser:** `http://100.112.124.101:<web-viewer-port>` (noVNC web viewer)

### After RDP-ing In

1. Open Edge → download SQL Server 2022 Developer: `https://go.microsoft.com/fwlink/p/?linkid=2215158`
2. Run installer → pick "Basic"
3. Download SSMS: `https://aka.ms/ssmsfullsetup`
4. Install, open SSMS, connect to `localhost`
5. Load client schema

---

## Managing Environments

### Start/Stop

```bash
# Start a client environment
cd /opt/mock-environments/<client-name> && docker compose up -d

# Stop (preserves data)
docker compose down

# Wipe everything and start fresh
docker compose down -v && docker compose up -d
```

### See all running environments

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep -v monitoring
```

### Resource usage

```bash
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'
```

### How much headroom does kc-llm have?

- **CPU:** 32 cores (Ryzen 9 9950X)
- **RAM:** 61GB total
- **Disk:** 785GB available

Each SQL Server container uses ~1-2GB RAM. Each Windows VM uses ~8GB. You can comfortably run 5-6 client environments simultaneously.

---

## Rules

1. **Never touch production** — that's the whole point of this
2. **Never modify SQL Grease** — ~40 people consuming it, read-only for us
3. **Grafana dashboards** — only create new, never modify existing
4. **Kubota** — must present at Wednesday meeting before any production deploy
5. **Test here first** — script the schema, load it, run your agents/dashboards against mock, validate, then go live

---

## Quick Reference: Spinning Up a New Client in 5 Minutes

```bash
# Replace <client> with the client name, <port> with an unused port
CLIENT="newclient"
PORT="1435"
PASS="StrongPass#2026!"

sudo mkdir -p /opt/mock-environments/$CLIENT/init
sudo chown -R kc-llm:kc-llm /opt/mock-environments/$CLIENT

cat > /opt/mock-environments/$CLIENT/docker-compose.yml << EOF
services:
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: ${CLIENT}-mssql
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: "${PASS}"
      MSSQL_PID: Developer
    ports:
      - "${PORT}:1433"
    volumes:
      - data:/var/opt/mssql
      - ./init:/init
    restart: unless-stopped
    mem_limit: 4g

volumes:
  data:
EOF

cd /opt/mock-environments/$CLIENT && docker compose up -d
echo "Ready in ~15 seconds at 100.112.124.101:$PORT (sa / $PASS)"
```

Drop your `.sql` files in `init/`, run them with sqlcmd, and you're testing.
