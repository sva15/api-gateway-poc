# Kong Architecture in Existing ALB Setup

This document explains how to deploy Kong API Gateway alongside your existing UI/ALB architecture.

---

## Current Architecture (Unchanged)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EC2 Instance                                    │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                         Nginx (Port 80)                          │   │
│   │                                                                  │   │
│   │  /ui/*              → localhost:3000 (Angular UI 1)             │   │
│   │  /dev/ui/*          → localhost:3001 (Angular UI 2)             │   │
│   │  /dev/configurator/*→ localhost:3002 (Configurator UI)          │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                     ↑                                    │
└─────────────────────────────────────│────────────────────────────────────┘
                                      │
                    ┌─────────────────┴──────────────────┐
                    │      Nginx Target Group (Port 80)   │
                    └─────────────────┬──────────────────┘
                                      │
┌─────────────────────────────────────┴───────────────────────────────────┐
│                          Internal ALB                                    │
│                                                                          │
│  Path Rules:                                                             │
│  • /ui/*               → Nginx Target Group                             │
│  • /dev/ui/*           → Nginx Target Group                             │
│  • /dev/configurator/* → Nginx Target Group                             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Target Architecture (With Kong Added)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EC2 Instance                                    │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                         Nginx (Port 80)                          │   │
│   │  /ui/*, /dev/ui/*, /dev/configurator/* → Angular Apps            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    Kong Gateway (Port 8000)                      │   │
│   │                                                                  │   │
│   │  /dev/api/service1/* → Lambda / EC2 Backend                     │   │
│   │  /dev/api/service2/* → Lambda / EC2 Backend                     │   │
│   │  /dev/api/config     → Config Service                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                     ↑                                    │
└─────────────────────────────────────│────────────────────────────────────┘
                                      │
          ┌───────────────────────────┴────────────────────────────┐
          │                                                        │
┌─────────┴─────────┐                                ┌─────────────┴─────────┐
│ Nginx Target Group│                                │  Kong Target Group    │
│    (Port 80)      │                                │    (Port 8000)        │
└─────────┬─────────┘                                └─────────────┬─────────┘
          │                                                        │
┌─────────┴────────────────────────────────────────────────────────┴─────────┐
│                              Internal ALB                                   │
│                                                                             │
│  Path Rules:                                                                │
│  • /ui/*               → Nginx Target Group                                │
│  • /dev/ui/*           → Nginx Target Group                                │
│  • /dev/configurator/* → Nginx Target Group                                │
│  • /dev/api/*          → Kong Target Group  [NEW]                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation Steps

### Step 1: Create Kong Target Group

1. Go to EC2 Console → Target Groups → Create
2. Configure:

| Setting | Value |
|---------|-------|
| Target type | IP |
| Target group name | `kong-api-gateway-tg` |
| Protocol | HTTP |
| Port | 8000 |
| VPC | Your VPC |
| Health check path | `/status` |

3. Register the EC2 private IP as target

### Step 2: Add ALB Listener Rule

1. Go to ALB → Listeners → View/edit rules
2. Add rule:

| Condition | Action |
|-----------|--------|
| Path pattern: `/dev/api/*` | Forward to `kong-api-gateway-tg` |

3. Set priority above the default rule

### Step 3: Kong Deployment Options

#### Option A: Docker on Same EC2 (Recommended for POC)

```yaml
# docker-compose.yaml
version: '3.8'
services:
  kong-database:
    image: postgres:13
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kongpass
    volumes:
      - kong-data:/var/lib/postgresql/data

  kong-migrations:
    image: kong:3.5
    command: kong migrations bootstrap
    depends_on:
      - kong-database
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_PASSWORD: kongpass
    restart: on-failure

  kong:
    image: kong:3.5
    depends_on:
      - kong-database
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_PASSWORD: kongpass
      KONG_PROXY_LISTEN: 0.0.0.0:8000
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
    ports:
      - "8000:8000"   # Proxy
      - "8001:8001"   # Admin API
    restart: always

volumes:
  kong-data:
```

#### Option B: DB-less Mode (Simpler)

```yaml
# docker-compose.yaml
version: '3.8'
services:
  kong:
    image: kong:3.5
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /kong/declarative/kong.yml
      KONG_PROXY_LISTEN: 0.0.0.0:8000
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    volumes:
      - ./kong.yml:/kong/declarative/kong.yml
    ports:
      - "8000:8000"
      - "8001:8001"
```

### Step 4: Path Stripping Configuration

Since ALB routes `/dev/api/*` to Kong, configure Kong to handle accordingly:

```yaml
# kong.yml (DB-less config)
_format_version: "3.0"

services:
  - name: service1
    url: http://lambda-url-or-backend:port
    routes:
      - name: service1-route
        paths:
          - /dev/api/service1
        strip_path: false  # Keep full path
```

---

## Traffic Flow

```
Browser Request: http://alb-url/dev/api/service1/users/123
                           │
                           ▼
                    ┌─────────────┐
                    │     ALB     │
                    │             │
                    │ Rule: /dev/api/* → Kong TG
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │    Kong     │
                    │  (Port 8000)│
                    │             │
                    │ Route: /dev/api/service1/* → service1
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Backend    │
                    │ (Lambda/EC2)│
                    └─────────────┘
```

---

## Port Configuration

| Component | Port | Purpose |
|-----------|------|---------|
| Nginx | 80 | UI serving |
| Kong Proxy | 8000 | API Gateway |
| Kong Admin | 8001 | Configuration API |
| PostgreSQL | 5432 | Kong datastore (if using DB mode) |

---

## Security Considerations

### EC2 Security Group

| Type | Port | Source | Description |
|------|------|--------|-------------|
| Inbound | 80 | ALB SG | Nginx for UI |
| Inbound | 8000 | ALB SG | Kong proxy |
| Inbound | 8001 | Your IP only | Kong admin (restrict!) |

> ⚠️ **Critical:** Never expose port 8001 (Kong Admin) to the ALB or public.

---

## Health Checks

### ALB Health Check for Kong

| Setting | Value |
|---------|-------|
| Protocol | HTTP |
| Path | `/status` |
| Port | 8000 |
| Healthy threshold | 2 |
| Unhealthy threshold | 3 |
| Timeout | 5s |
| Interval | 30s |

---

## Verification

```bash
# Test Kong is running
curl http://localhost:8000/status

# Test via ALB
curl http://alb-url/dev/api/service1/test

# Check Kong admin (from EC2 only)
curl http://localhost:8001/services
```

---

**← Previous:** [01-kong-vs-aws-apigw.md](01-kong-vs-aws-apigw.md) | **Next:** [03-installation.md](03-installation.md) →
