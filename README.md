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

## Additional Resources

- **n8n Documentation**: https://docs.n8n.io/
- **PostgreSQL Docker**: https://hub.docker.com/_/postgres
- **ngrok Documentation**: https://ngrok.com/docs

## Credits

Configuration based on n8n official documentation and community best practices.
