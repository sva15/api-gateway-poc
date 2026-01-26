# Kong Installation on EC2

Step-by-step guide to install Kong on the same EC2 where Nginx is running.

---

## Prerequisites

- EC2 instance running (same as UI/Nginx)
- Docker and Docker Compose installed
- Security group allows ports 8000 (proxy) and 8001 (admin, internal only)

---

## Option 1: DB-less Mode (Recommended for POC)

DB-less mode uses a YAML config file instead of a database. Simpler but requires restart for config changes.

### Step 1: Create Directory Structure

```bash
mkdir -p /opt/kong
cd /opt/kong
```

### Step 2: Create Kong Configuration

```bash
cat > kong.yml << 'EOF'
_format_version: "3.0"
_transform: true

services:
  - name: service1
    url: http://backend-host:port
    routes:
      - name: service1-proxy-route
        paths:
          - /dev/api/service1
        strip_path: false
        methods:
          - GET
          - POST
          - PUT
          - DELETE

  - name: service2
    url: http://another-backend:port
    routes:
      - name: service2-proxy-route
        paths:
          - /dev/api/service2
        strip_path: false

plugins:
  - name: rate-limiting
    config:
      minute: 100
      policy: local
EOF
```

### Step 3: Create Docker Compose

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  kong:
    image: kong:3.5
    container_name: kong-gateway
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /kong/declarative/kong.yml
      KONG_PROXY_LISTEN: 0.0.0.0:8000
      KONG_ADMIN_LISTEN: 127.0.0.1:8001
      KONG_LOG_LEVEL: info
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
    volumes:
      - ./kong.yml:/kong/declarative/kong.yml:ro
    ports:
      - "8000:8000"
      - "127.0.0.1:8001:8001"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "kong", "health"]
      interval: 30s
      timeout: 10s
      retries: 3
EOF
```

### Step 4: Start Kong

```bash
docker-compose up -d
```

### Step 5: Verify

```bash
# Check container
docker ps | grep kong

# Check health
curl http://localhost:8000/status

# Check config loaded
curl http://localhost:8001/services
```

---

## Option 2: Database Mode (Production)

Database mode uses PostgreSQL. Supports dynamic config changes via Admin API.

### Step 1: Create Docker Compose

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  kong-database:
    image: postgres:13
    container_name: kong-postgres
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kongpass123
    volumes:
      - kong-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  kong-migrations:
    image: kong:3.5
    command: kong migrations bootstrap
    depends_on:
      kong-database:
        condition: service_healthy
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_PASSWORD: kongpass123
    restart: on-failure

  kong:
    image: kong:3.5
    container_name: kong-gateway
    depends_on:
      kong-database:
        condition: service_healthy
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_PASSWORD: kongpass123
      KONG_PROXY_LISTEN: 0.0.0.0:8000
      KONG_ADMIN_LISTEN: 127.0.0.1:8001
      KONG_LOG_LEVEL: info
    ports:
      - "8000:8000"
      - "127.0.0.1:8001:8001"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "kong", "health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  kong-data:
EOF
```

### Step 2: Start Kong

```bash
docker-compose up -d
```

### Step 3: Configure via Admin API

```bash
# Create service
curl -X POST http://localhost:8001/services \
  -d name=service1 \
  -d url=http://backend-host:port

# Create route
curl -X POST http://localhost:8001/services/service1/routes \
  -d name=service1-route \
  -d paths[]=/dev/api/service1 \
  -d strip_path=false
```

---

## Configuration Management

### DB-less: Reload Config

```bash
# Edit kong.yml, then:
docker-compose restart kong

# Or hot reload (Kong 3.x)
curl -X POST http://localhost:8001/config \
  -F config=@kong.yml
```

### Database Mode: Live Updates

```bash
# Add new route (no restart needed)
curl -X POST http://localhost:8001/services/service1/routes \
  -d name=new-route \
  -d paths[]=/dev/api/new-endpoint
```

---

## Monitoring & Logs

### View Logs

```bash
# Follow logs
docker logs -f kong-gateway

# Last 100 lines
docker logs --tail 100 kong-gateway
```

### Check Status

```bash
# Kong status
curl http://localhost:8001/status

# List all services
curl http://localhost:8001/services

# List all routes
curl http://localhost:8001/routes
```

---

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Port 8000 not accessible | Security group | Allow 8000 from ALB SG |
| Config not loading | YAML syntax error | Validate with `kong config parse kong.yml` |
| Container keeps restarting | Database not ready | Wait for postgres, check logs |
| Admin API not accessible | Bound to 127.0.0.1 | Intentional - access only from EC2 |

---

**← Previous:** [02-architecture.md](02-architecture.md) | **Next:** [04-routing.md](04-routing.md) →
