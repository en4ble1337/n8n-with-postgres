# n8n with PostgreSQL Docker Setup Guide

Complete Docker Compose setup for n8n workflow automation with PostgreSQL database backend and optional ngrok tunneling.

## Overview

This setup provides a production-ready n8n instance with:
- PostgreSQL database for persistent storage
- Docker Compose orchestration
- Health checks and automatic restarts
- Optional ngrok tunneling for external access

## Quick Start

### 1. Create Environment File

Create a `.env` file in your project directory:

```env
POSTGRES_USER=node
POSTGRES_PASSWORD=<yourPW>
POSTGRES_DB=n8n
POSTGRES_NON_ROOT_USER=node
POSTGRES_NON_ROOT_PASSWORD=<yourPW>
N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=false
N8N_SECURE_COOKIE=false
```

**Important:** Replace `<yourPW>` with a secure password.

### 2. Create Docker Compose File

Create a `docker-compose.yml` file:

```yaml
version: '3.3'
services:
  postgres:
    image: postgres:16
    restart: always
    environment:
      - POSTGRES_USER
      - POSTGRES_PASSWORD
      - POSTGRES_DB
    volumes:
      - db_storage:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}']
      interval: 5s
      timeout: 5s
      retries: 10

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: always
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=false
      - N8N_SECURE_COOKIE=false
      - N8N_HOST=10.1.20.63 
      - N8N_PROTOCOL=http    
    ports:
      - 5678:5678
    links:
      - postgres
    volumes:
      - n8n_storage:/home/node/.n8n
    depends_on:
      - postgres

volumes:
  db_storage:
  n8n_storage:
```

### 3. Start the Services

```bash
docker-compose up -d
```

### 4. Access n8n

Open your browser and navigate to:
- Local: `http://10.1.20.63:5678`
- Or: `http://localhost:5678`

### 5. Add Postgress plugin in n8n with credentials:
Host: postgres (this is the Docker service name for internal container communication)

Database: n8n (as specified by POSTGRES_DB=n8n in your .env file)

User: node (as specified by POSTGRES_USER=node in your .env file)

Password: <yourPW> (whatever secure password you replaced <yourPW> with in your .env file)

Port: 5432 (standard PostgreSQL port)

## ngrok Integration (Optional)

For external access and OAuth callbacks, you can use ngrok tunneling.

### 1. Install ngrok

Follow the setup guide at: https://dashboard.ngrok.com/get-started/setup/linux

### 2. Start ngrok Tunnel

```bash
# Run in screen session for persistence
screen -S ngrok
ngrok http --url=wrongly.app 10.1.20.63:5678
```

### 3. Update Docker Compose for ngrok

Modify your `docker-compose.yml` n8n service environment:

```yaml
n8n:
  # ... other configurations
  environment:
    # ... existing environment variables
    # Keep internal host as localhost/IP for MCP communication
    - N8N_HOST=10.1.20.63
    - N8N_PROTOCOL=http
    # Use separate webhook URL for OAuth callbacks
    - WEBHOOK_URL=https://wrongly.ngrok-free.app
    - N8N_EDITOR_BASE_URL=https://wrongly.ngrok-free.app
```

### 4. Restart Services

```bash
docker-compose down
docker-compose up -d
```

## Configuration Details

### Environment Variables

| Variable | Description |
|----------|-------------|
| `POSTGRES_USER` | PostgreSQL username |
| `POSTGRES_PASSWORD` | PostgreSQL password |
| `POSTGRES_DB` | Database name for n8n |
| `N8N_HOST` | Host IP for n8n server |
| `N8N_PROTOCOL` | Protocol (http/https) |
| `WEBHOOK_URL` | External webhook URL for OAuth |
| `N8N_EDITOR_BASE_URL` | Base URL for n8n editor |

### Network Configuration

- **Internal Communication**: Uses Docker network for postgres ↔ n8n
- **External Access**: Port 5678 exposed for web interface
- **ngrok Tunnel**: Optional external access via HTTPS

## Management Commands

### Start Services

```bash
docker-compose up -d
```

### Stop Services

```bash
docker-compose down
```

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f n8n
docker-compose logs -f postgres
```

### Update Services

```bash
docker-compose pull
docker-compose up -d
```

### Database Backup

```bash
docker-compose exec postgres pg_dump -U node n8n > n8n_backup.sql
```

### Database Restore

```bash
cat n8n_backup.sql | docker-compose exec -T postgres psql -U node -d n8n
```

## Troubleshooting

### Connection Issues

```bash
# Check service status
docker-compose ps

# Check network connectivity
docker-compose exec n8n ping postgres
```

### Database Issues

```bash
# Check PostgreSQL logs
docker-compose logs postgres

# Connect to database
docker-compose exec postgres psql -U node -d n8n
```

### ngrok Issues

```bash
# Check ngrok status
screen -r ngrok

# Verify tunnel URL
curl https://wrongly.ngrok-free.app
```

## Security Considerations

- Change default passwords in `.env` file
- Use HTTPS in production (consider reverse proxy)
- Restrict database access to n8n service only
- Keep Docker images updated
- Use strong authentication for n8n

## File Structure

```
project/
├── .env
├── docker-compose.yml
└── README.md
```

```markdown
# n8n Self-Hosting Best Practices: Internal DNS, NGINX Proxy, and Cloudflared

A step-by-step quick tip for exposing n8n securely and reliably on your local network and the internet—using Cloudflared, pfSense/OPNsense DNS host overrides, and Nginx Proxy Manager.

_Author: @en4ble1337_

[Full guide here (nginx/ngnix + pfSense host overrides)](https://github.com/en4ble1337/ngnix-opnsense-cloudflare)

---

## Scenario

You want to:
- Run n8n in Docker on your local server
- Access it via a user-friendly domain (e.g. n8n.get1337.xyz) both **locally** (LAN) and **remotely** (WAN)
- Use secure HTTPS externally (via Cloudflared or NGINX Proxy Manager)
- Avoid SSL errors, browser HSTS issues, and network confusion

---

## Key Tips and Lessons Learned

### 1. Always Use Universal Bind

In docker-compose.yml for n8n:

```yaml
environment:
  - N8N_HOST=0.0.0.0  # Must-use for multi-network access!
  - N8N_PROTOCOL=http  # n8n speaks HTTP to proxy
  - WEBHOOK_URL=https://n8n.get1337.xyz/  # Tells n8n to advertise external HTTPS
  # other variables...
```

- **Why:** Ensures n8n listens on all interfaces (LAN, localhost, Docker bridge, etc.), avoiding "can't access by domain" errors[web:1].

---

### 2. Internal DNS Overrides for Seamless LAN Routing

**On pfSense/OPNsense:**
- Use DNS Host Overrides to map n8n.get1337.xyz ➔ Your n8n server's LAN IP (e.g. 10.1.20.148)
- Ensures LAN clients hit the local server directly and avoid "hairpin NAT"/double-NAT loopbacks

**Sample DNS Host Override Table:**

| Host | Parent Domain of Host | IP Return For Host | Description |
|------|----------------------|-------------------|---------------------|
| n8n | get1337.xyz | 10.1.20.148 | n8n internal DNS |

- See real-world config screenshots and more guidance at [github.com/en4ble1337/ngnix-opnsense-cloudflare](https://github.com/en4ble1337/ngnix-opnsense-cloudflare)[web:1][attached_image:1].

---

### 3. Nginx Proxy Manager for Local HTTPS

- Forward port 443 HTTPS traffic on n8n.get1337.xyz to n8n's HTTP port (e.g., 10.1.20.148:5678)
- Use Let's Encrypt, wildcard, or your preferred cert for SSL in NPM
- **Do NOT use Cloudflare-embedded certs unless subdomain matches**
- "Force SSL" on in the proxy, let NPM handle certificate refresh and upgrades

**Minimal Proxy Host Configuration:**

| Domain Names | Scheme | Forward Host/IP | Forward Port | Cert | Force SSL | HSTS | Notes |
|-------------------|--------|-----------------|-------------|----------|-----------|------|--------------|
| n8n.get1337.xyz | http | 10.1.20.148 | 5678 | Let's Encrypt (or wildcard) | Yes | No | Do not use "Cloudflare Origin" unless wildcard! |

*Screenshots in linked repo guide.*

---

### 4. Bonus: Troubleshooting Guide (Browser/Cache Issues)

- If HTTPS fails **only on one device**:
  - Clear your browser SSL state, cookies, and **HSTS settings** (see "chrome://net-internals/#hsts")
  - Flush OS DNS and certificate caches
  - Try incognito/private window
  - Temporarily disable antivirus/proxy/VPN for further isolation

---

## Full Reference & More Examples

- See [https://github.com/en4ble1337/ngnix-opnsense-cloudflare](https://github.com/en4ble1337/ngnix-opnsense-cloudflare) for exhaustive screenshots, troubleshooting, and custom NGINX advanced tips.

---

### Key Takeaways

- **Always set** `N8N_HOST=0.0.0.0` for Dockerized workloads where multi-network or proxy access is required
- Leverage local DNS host overrides for hassle-free, hairpin-NAT-free local access
- Use Nginx Proxy Manager or a similar reverse proxy for local HTTPS, configured with a matching certificate for every subdomain
- Remember: Most "can't connect" problems are either proxy/cert mismatch, DNS confusion, or browser side-caching (including HSTS!)
```


## Additional Resources

- **n8n Documentation**: https://docs.n8n.io/
- **PostgreSQL Docker**: https://hub.docker.com/_/postgres
- **ngrok Documentation**: https://ngrok.com/docs

## Credits

Configuration based on n8n official documentation and community best practices.
