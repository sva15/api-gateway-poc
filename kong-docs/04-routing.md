# Kong Routing - Proxy and Explicit Models

This document explains both routing models in Kong, equivalent to AWS API Gateway's `{proxy+}` and explicit path routing.

---

## Routing Concept Mapping

| AWS API Gateway | Kong |
|-----------------|------|
| Resource | Route path |
| `{proxy+}` | Prefix match or regex |
| Explicit path | Exact match |
| Stage (`/dev`) | Part of path or separate config |
| Method | Route methods |
| Integration | Service upstream |

---

## Model 1: Proxy-Based Routing (Like `{proxy+}`)

Catch-all routing where Kong forwards all subpaths to a single backend.

### Use Case
```
/dev/api/service1/*  → All requests to one backend
Backend handles internal routing
```

### Configuration (DB-less)

```yaml
_format_version: "3.0"

services:
  - name: service1-backend
    url: http://service1-host:8080
    routes:
      - name: service1-proxy
        paths:
          - /dev/api/service1    # Prefix match
        strip_path: false        # Keep full path
        preserve_host: false
        methods:                 # Optional: all methods
          - GET
          - POST
          - PUT
          - DELETE
          - PATCH
```

### Configuration (Admin API)

```bash
# Create service
curl -X POST http://localhost:8001/services \
  -d name=service1-backend \
  -d url=http://service1-host:8080

# Create catch-all route
curl -X POST http://localhost:8001/services/service1-backend/routes \
  -d name=service1-proxy \
  -d "paths[]=/dev/api/service1" \
  -d strip_path=false
```

### Behavior

| Request | Forwarded To | Path Received by Backend |
|---------|--------------|-------------------------|
| `/dev/api/service1/users` | service1-host:8080 | `/dev/api/service1/users` |
| `/dev/api/service1/orders/123` | service1-host:8080 | `/dev/api/service1/orders/123` |
| `/dev/api/service1/` | service1-host:8080 | `/dev/api/service1/` |

---

## Model 2: Explicit Route-Based Routing

Individual paths with per-route configuration.

### Use Case
```
GET  /dev/api/config   → config-service (read)
POST /dev/api/config   → config-service (write, different auth)
GET  /dev/api/users    → users-service
```

### Configuration (DB-less)

```yaml
_format_version: "3.0"

services:
  - name: config-service
    url: http://config-host:8080
    routes:
      - name: config-get
        paths:
          - /dev/api/config
        methods:
          - GET
        strip_path: false

      - name: config-post
        paths:
          - /dev/api/config
        methods:
          - POST
        strip_path: false

  - name: users-service
    url: http://users-host:8080
    routes:
      - name: users-route
        paths:
          - /dev/api/users
        methods:
          - GET
          - POST
        strip_path: false
```

### Per-Route Plugins

```yaml
services:
  - name: config-service
    url: http://config-host:8080
    routes:
      - name: config-get
        paths:
          - /dev/api/config
        methods:
          - GET
        plugins:
          - name: rate-limiting
            config:
              minute: 100

      - name: config-post
        paths:
          - /dev/api/config
        methods:
          - POST
        plugins:
          - name: key-auth  # Require API key for POST
          - name: rate-limiting
            config:
              minute: 10    # Lower limit for writes
```

---

## Model 3: Hybrid Approach (Recommended)

Mix proxy routes for flexible services and explicit routes for controlled endpoints.

```yaml
_format_version: "3.0"

services:
  # Proxy-based: Backend handles routing
  - name: legacy-service
    url: http://legacy-host:8080
    routes:
      - name: legacy-proxy
        paths:
          - /dev/api/legacy
        strip_path: false

  # Explicit: Kong controls routing
  - name: config-service
    url: http://config-host:8080
    routes:
      - name: config-read
        paths:
          - /dev/api/config
        methods:
          - GET

      - name: config-write
        paths:
          - /dev/api/config
        methods:
          - POST
          - PUT
        plugins:
          - name: key-auth
```

---

## Path Matching Rules

Kong uses **prefix matching** by default.

| Path Config | Matches | Does NOT Match |
|-------------|---------|----------------|
| `/api` | `/api`, `/api/users`, `/api/x/y` | `/apis`, `/ap` |
| `/api/users` | `/api/users`, `/api/users/123` | `/api/user` |
| `~/api/v[0-9]+` | `/api/v1`, `/api/v2` (regex) | `/api/vX` |

### Regex Routes

```yaml
routes:
  - name: versioned-api
    paths:
      - "~/dev/api/v[0-9]+/users"  # ~ prefix = regex
    methods:
      - GET
```

---

## strip_path Explained

| `strip_path` | Request | Backend Receives |
|--------------|---------|------------------|
| `false` | `/dev/api/service1/users` | `/dev/api/service1/users` |
| `true` | `/dev/api/service1/users` | `/users` |

### When to Use

| Scenario | strip_path |
|----------|------------|
| Backend expects full path | `false` |
| Backend has its own routing | `false` |
| Adapting different path structure | `true` |
| Backend expects relative paths | `true` |

---

## Route Priority

Kong uses route specificity for matching order:

1. **Regex routes** (lowest priority)
2. **Prefix routes** (by length, longer = higher priority)
3. **Exact routes with methods** (highest priority)

### Example Priority

```yaml
# Order of matching (high to low):
1. POST /dev/api/config          # Exact path + method
2. /dev/api/config               # Exact path
3. /dev/api                      # Shorter prefix
4. ~/dev/api/.*                  # Regex
```

---

## Comparison with AWS API Gateway

| Feature | AWS API Gateway | Kong |
|---------|-----------------|------|
| Proxy route | `{proxy+}` | Prefix path |
| Explicit route | Defined resource | Explicit path + methods |
| Per-route auth | Yes | Yes (plugins) |
| Per-route throttle | Yes (Usage Plan) | Yes (rate-limiting plugin) |
| Path parameters | `{id}` | Regex capture |
| Query routing | No | Via plugins |
| Header routing | No | Native support |

---

## Testing Routes

```bash
# Test proxy route
curl http://localhost:8000/dev/api/service1/users

# Test explicit route - GET
curl http://localhost:8000/dev/api/config

# Test explicit route - POST
curl -X POST http://localhost:8000/dev/api/config \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}'

# List all routes
curl http://localhost:8001/routes
```

---

## Decision Guide

| Scenario | Route Model |
|----------|-------------|
| Migrating from ALB wildcard | Proxy (prefix) |
| Backend framework with routing | Proxy |
| Different auth per endpoint | Explicit |
| Different rate limits per endpoint | Explicit |
| Stable, documented API | Explicit |
| Rapid development | Proxy |

---

**← Previous:** [03-installation.md](03-installation.md) | **Next:** [05-security.md](05-security.md) →
