# Kong Manager - Step-by-Step POC Implementation

Complete walkthrough for configuring Kong Gateway via Kong Manager UI, matching AWS API Gateway functionality.

---

## Quick Reference: What You Need to Create

| Kong Component | AWS API Gateway Equivalent | Required for POC |
|----------------|---------------------------|------------------|
| **Gateway Service** | Integration (backend URL) | ✅ Yes |
| **Route** | Resource + Method | ✅ Yes |
| **Consumer** | API Key identity | ✅ Yes (if using API keys) |
| **Consumer Group** | Usage Plan group | Optional |
| **Upstream** | N/A (internal) | Optional (for load balancing) |
| **Plugins** | Features (auth, CORS, etc.) | ✅ Yes |

---

## Part 1: Understanding Kong Manager Components

### 1.1 Gateway Service

**Purpose:** Defines your backend/upstream service (where Kong forwards requests).

| Field | Description | Example |
|-------|-------------|---------|
| **Name** | Unique identifier | `user-service` |
| **Protocol** | http or https | `http` |
| **Host** | Backend hostname/IP | `internal-alb.amazonaws.com` |
| **Port** | Backend port | `80` |
| **Path** | Base path (optional) | `/api` |
| **Retries** | Retry count on failure | `5` |
| **Timeouts** | Connection/Read/Write | `60000` ms each |

**AWS Equivalent:** Integration settings (backend URL)

---

### 1.2 Route

**Purpose:** Defines which requests match this service (path, method, headers).

| Field | Description | Example |
|-------|-------------|---------|
| **Name** | Unique identifier | `user-service-route` |
| **Service** | Link to Gateway Service | `user-service` |
| **Paths** | URL paths to match | `/users`, `/users/` |
| **Methods** | HTTP methods | `GET, POST, PUT, DELETE` |
| **Strip Path** | Remove matched path | `OFF` (keep path) |
| **Hosts** | Match by hostname | (leave empty) |
| **Headers** | Match by header | (leave empty) |
| **Protocols** | http, https | `HTTP, HTTPS` |

**AWS Equivalent:** Resource + Method + `{proxy+}`

---

### 1.3 Consumer

**Purpose:** Represents an API client/user for authentication and rate limiting.

| Field | Description | Example |
|-------|-------------|---------|
| **Username** | Unique identifier | `frontend-app` |
| **Custom ID** | External system ID | `app-123` |
| **Tags** | Organizational tags | `internal, v1` |

**AWS Equivalent:** API Key owner identity

---

### 1.4 Consumer Group

**Purpose:** Group consumers for shared rate limiting (like AWS Usage Plans).

| Field | Description | Example |
|-------|-------------|---------|
| **Name** | Group identifier | `premium-tier` |
| **Consumers** | Members of group | `frontend-app, mobile-app` |

**AWS Equivalent:** Usage Plan

---

### 1.5 Upstream (Optional)

**Purpose:** Load balancing across multiple backend targets.

| Field | Description | Example |
|-------|-------------|---------|
| **Name** | Virtual hostname | `user-service-upstream` |
| **Algorithm** | Round Robin, Least Conn | `Round Robin` |
| **Health Checks** | Active/Passive probing | Configure as needed |

**AWS Equivalent:** N/A (built-in to ALB)

---

## Part 2: Step-by-Step Implementation

### Prerequisites

- Kong Gateway running (docker-compose)
- Access to Kong Manager UI: `http://localhost:8002`
- Backend service URLs ready

---

### Step 1: Create Gateway Service

**Navigation:** Kong Manager → Gateway Services → New Gateway Service

#### For Lambda via ALB (Existing Infrastructure)

| Field | Value |
|-------|-------|
| **Name** | `lambda-service-1` |
| **Full URL** | `http://internal-alb-dns.region.elb.amazonaws.com` |
| **Retries** | `3` |
| **Connection timeout** | `60000` |
| **Read timeout** | `60000` |
| **Write timeout** | `60000` |

#### For Direct Backend (EC2/ECS)

| Field | Value |
|-------|-------|
| **Protocol** | `http` |
| **Host** | `10.0.1.50` (private IP) |
| **Port** | `8080` |
| **Path** | `/api` (optional) |

**Click:** Save

---

### Step 2: Create Route

**Navigation:** Kong Manager → Routes → Create Route

#### Option A: Proxy Route (Catches All Subpaths - Like `{proxy+}`)

| Field | Value | Notes |
|-------|-------|-------|
| **Name** | `lambda-service-1-route` | |
| **Service** | `lambda-service-1` | Select from dropdown |
| **Protocols** | `HTTP, HTTPS` | ✅ Both checked |
| **Paths** | `/service1` | Matches `/service1/*` |
| **Strip Path** | `OFF` (unchecked) | Keep full path! |
| **Methods** | Leave empty | Matches ALL methods |

> **Key Point:** Kong prefix matching means `/service1` matches:
> - `/service1`
> - `/service1/users`
> - `/service1/users/123/orders`

#### Option B: Explicit Method Route

| Field | Value |
|-------|-------|
| **Name** | `users-get-route` |
| **Service** | `user-service` |
| **Paths** | `/users` |
| **Methods** | `GET` only |
| **Strip Path** | `OFF` |

Create additional routes for POST, PUT, DELETE with different settings.

**Click:** Save

---

### Step 3: Create Consumer (For API Key Auth)

**Navigation:** Kong Manager → Consumers → New Consumer

| Field | Value |
|-------|-------|
| **Username** | `frontend-app` |
| **Custom ID** | (optional) |
| **Tags** | `internal, v1` |

**Click:** Save

---

### Step 4: Add Credentials to Consumer

**Navigation:** Consumers → `frontend-app` → Credentials → New Key Auth Credential

| Field | Value |
|-------|-------|
| **Key** | Auto-generate OR enter custom key |

**Save the generated key!** You'll need it for API calls.

---

### Step 5: Enable Key Authentication Plugin

**Navigation:** Plugins → New Plugin → Key Authentication

#### Scope

Choose: **Scoped** → Select specific routes that need protection

#### Configuration

| Field | Value | Notes |
|-------|-------|-------|
| **Protocols** | `http, https` | ✅ Check both |
| **Key Names** | `x-api-key` | Match AWS APIGW header |
| **Key In Header** | ✅ ON | Accept in header |
| **Key In Query** | OFF | Disable for security |
| **Key In Body** | OFF | Disable |
| **Hide Credentials** | ✅ ON | Don't send to backend |
| **Run On Preflight** | OFF | Skip OPTIONS |

**Click:** Save

---

### Step 6: Enable CORS Plugin

**Navigation:** Plugins → New Plugin → CORS

#### Scope

Choose: **Global** (all routes) OR **Scoped** (browser-facing routes only)

#### Configuration

| Field | Value | Notes |
|-------|-------|-------|
| **Protocols** | `http, https` | |
| **Origins** | `*` | Or specific domain |
| **Methods** | `GET,POST,PUT,DELETE,OPTIONS` | |
| **Headers** | `Content-Type,Authorization,x-api-key` | Include x-api-key! |
| **Exposed Headers** | (leave empty) | |
| **Max Age** | `3600` | Cache 1 hour |
| **Credentials** | OFF | Unless using cookies |
| **Preflight Continue** | OFF | |

**Click:** Save

---

### Step 7: Enable Rate Limiting Plugin (Optional)

**Navigation:** Plugins → New Plugin → Rate Limiting

#### Configuration

| Field | Value | Notes |
|-------|-------|-------|
| **Scope** | Per route or per consumer | |
| **Minute** | `100` | 100 requests/minute |
| **Hour** | `1000` | 1000 requests/hour |
| **Policy** | `local` | In-memory (use `redis` for distributed) |
| **Hide Client Headers** | OFF | Show limits in response |

---

### Step 8: Enable AWS Lambda Plugin (If Direct Lambda)

**Navigation:** Plugins → New Plugin → AWS Lambda

#### Scope

Apply to routes that call Lambda directly.

#### Configuration

| Field | Value | Notes |
|-------|-------|-------|
| **Aws Region** | `ap-south-1` | Your region |
| **Function Name** | `my-lambda-function` | Lambda name |
| **Aws Key** | **LEAVE EMPTY** | Uses EC2 IAM role |
| **Aws Secret** | **LEAVE EMPTY** | Uses EC2 IAM role |
| **Aws Imds Protocol Version** | `v2` | More secure |
| **Invocation Type** | `RequestResponse` | Sync |
| **Timeout** | `60000` | 60 seconds |
| **Forward Request Body** | ✅ ON | |
| **Forward Request Headers** | ✅ ON | |
| **Forward Request Method** | ✅ ON | |
| **Forward Request Uri** | ✅ ON | |
| **Is Proxy Integration** | ✅ ON | APIGW format |
| **Awsgateway Compatible** | ✅ ON | APIGW event format |

---

## Part 3: Testing

### Test Without Authentication

```bash
# If key-auth not enabled on route
curl http://localhost:8000/service1/test
```

### Test With API Key

```bash
curl -H "x-api-key: YOUR_GENERATED_KEY" \
  http://localhost:8000/service1/test
```

### Test POST Request

```bash
curl -X POST \
  -H "x-api-key: YOUR_GENERATED_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "John"}' \
  http://localhost:8000/service1/users
```

### Via ALB/Nginx

```bash
curl -H "x-api-key: YOUR_KEY" \
  http://aws-alb-url/api-gateway/service1/test
```

---

## Part 4: Configuration Summary

### Minimum POC Setup

```
┌─────────────────────────────────────────────────────────────┐
│                    MINIMUM POC CONFIG                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Gateway Service: lambda-service-1                        │
│     └── URL: http://internal-alb/lambda-path                │
│                                                              │
│  2. Route: lambda-service-1-route                           │
│     └── Path: /service1 (catches all subpaths)              │
│     └── Strip Path: OFF                                      │
│     └── Methods: (empty = all)                               │
│                                                              │
│  3. Consumer: frontend-app                                   │
│     └── Credential: x-api-key = abc123...                   │
│                                                              │
│  4. Plugins:                                                 │
│     └── key-auth (on route)                                 │
│     └── cors (global)                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 5: AWS API Gateway Mapping

| AWS Step | Kong Step |
|----------|-----------|
| Create REST API | N/A (Kong is the API) |
| Create Resource `/service1` | Create Route with path `/service1` |
| Create `{proxy+}` | Prefix match (automatic with path) |
| Create ANY method | Leave Methods empty in Route |
| Create Integration | Create Gateway Service with backend URL |
| Enable API Key Required | Add Key Auth plugin to Route |
| Enable CORS | Add CORS plugin |
| Create Usage Plan | Create Rate Limiting plugin + Consumer Group |
| Create API Key | Create Consumer + Key Auth Credential |
| Deploy to Stage | N/A (Kong is live immediately) |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| 404 Not Found | Check Route path matches request |
| 401 Unauthorized | Check API key and Key Auth plugin |
| 403 Forbidden | Check ACL plugin if enabled |
| 502 Bad Gateway | Check Service URL is reachable |
| CORS error | Check CORS plugin Origins and Headers |

---

**Next Steps:**
1. Configure Nginx reverse proxy for ALB
2. Add more services and routes
3. Test end-to-end via ALB URL
