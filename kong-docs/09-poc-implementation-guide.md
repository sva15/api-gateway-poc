# Kong POC Implementation Guide

Complete configuration guide for Kong Gateway behind Nginx/ALB with plugin setup.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Internal ALB                                       │
│                                                                              │
│  http://aws-alb-url/api-gateway/*  →  EC2 (Nginx:80)                        │
│  http://aws-alb-url/ui/*           →  EC2 (Nginx:80)                        │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            EC2 Instance                                      │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        Nginx (Port 80)                               │   │
│   │                                                                      │   │
│   │  /api-gateway/      → proxy_pass http://127.0.0.1:8000/             │   │
│   │  /kong-manager/     → proxy_pass http://127.0.0.1:8002/             │   │
│   │  /ui/*              → Static files / Angular apps                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                  Kong Gateway (Docker)                               │   │
│   │                                                                      │   │
│   │  Port 8000 (Proxy)   → API requests                                 │   │
│   │  Port 8001 (Admin)   → Admin API (internal only)                    │   │
│   │  Port 8002 (Manager) → Kong Manager UI                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                    ┌─────────────────┴──────────────────┐                   │
│                    ▼                                    ▼                    │
│           ┌──────────────┐                    ┌──────────────┐              │
│           │ Lambda (VPC) │                    │ Internal ALB │              │
│           │ via IAM Role │                    │ Lambda Target│              │
│           └──────────────┘                    └──────────────┘              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Nginx Configuration

### nginx.conf

```nginx
server {
    listen 80;
    server_name _;

    # Kong Proxy (API Gateway)
    location /api-gateway/ {
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Prefix /api-gateway;
    }

    # Kong Manager UI (Optional - for admin access)
    location /kong-manager/ {
        proxy_pass http://127.0.0.1:8002/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Kong Admin API (Restrict to internal only!)
    location /kong-admin/ {
        # Only allow from localhost or specific IPs
        allow 127.0.0.1;
        deny all;
        proxy_pass http://127.0.0.1:8001/;
    }

    # Your existing UI routes
    location /ui/ {
        # Your existing Angular app config
    }
}
```

### URL Mapping

| External URL | Nginx Routes To | Kong Sees |
|--------------|-----------------|-----------|
| `http://alb/api-gateway/service1/users` | `http://127.0.0.1:8000/service1/users` | `/service1/users` |
| `http://alb/api-gateway/service2/orders` | `http://127.0.0.1:8000/service2/orders` | `/service2/orders` |
| `http://alb/kong-manager/` | `http://127.0.0.1:8002/` | Kong Manager UI |

---

## Part 2: Kong Routes Configuration

### Option A: Proxy-Based Routing (Like `{proxy+}`)

Catches all subpaths and forwards to single backend.

#### Via Kong Manager UI

1. **Create Service**
   - Name: `service1-backend`
   - URL: `http://internal-alb-url/lambda-target` OR direct Lambda

2. **Create Route**
   - Name: `service1-proxy-route`
   - Paths: `/service1` (this catches `/service1/*`)
   - Strip Path: `OFF` (keep full path)
   - Methods: Leave empty (all methods)

#### Behavior

| Request | Matches | Forwarded Path |
|---------|---------|----------------|
| `GET /service1/users` | ✅ | `/service1/users` |
| `POST /service1/users/123` | ✅ | `/service1/users/123` |
| `DELETE /service1/orders/456` | ✅ | `/service1/orders/456` |

---

### Option B: Explicit Method-Based Routing

Different configuration per HTTP method.

#### Via Kong Manager UI

1. **Create Service**
   - Name: `users-service`
   - URL: `http://backend:8080`

2. **Create Multiple Routes for Same Path**

   **Route 1: GET /users**
   - Name: `users-get`
   - Paths: `/users`
   - Methods: `GET`

   **Route 2: POST /users**
   - Name: `users-post`
   - Paths: `/users`
   - Methods: `POST`
   - (Apply stricter plugins here)

   **Route 3: DELETE /users/{id}**
   - Name: `users-delete`
   - Paths: `/users` (Kong uses prefix match)
   - Methods: `DELETE`
   - (Apply stricter plugins here)

3. **Apply Different Plugins Per Route**
   - `users-get`: No key-auth (public read)
   - `users-post`: Key-auth + Rate limiting
   - `users-delete`: Key-auth + ACL (admin only)

---

## Part 3: Plugin Configurations

### 3.1 Key Authentication Plugin

**Scope:** Apply to specific routes that need protection

#### Configuration Values

| Field | Value | Notes |
|-------|-------|-------|
| **Scope** | Scoped (specific routes) | Not global |
| **Protocols** | http, https | ✅ |
| **Hide Credentials** | ✅ ON | Remove key from upstream request |
| **Key In Header** | ✅ ON | Accept key in header |
| **Key In Query** | OFF | Disable for security |
| **Key In Body** | OFF | Disable |
| **Key Names** | `x-api-key` | Match AWS API Gateway |
| **Run On Preflight** | OFF | Don't check OPTIONS |

#### Create Consumer and API Key

1. **Consumers** → New Consumer
   - Username: `internal-service`

2. **Credentials** → Key Auth
   - Key: (auto-generate or enter your key)

---

### 3.2 CORS Plugin

**Scope:** Apply globally or to routes with browser access

#### Configuration Values

| Field | Value | Notes |
|-------|-------|-------|
| **Scope** | Global or Routes with browser access | |
| **Protocols** | http, https | ✅ |
| **Origins** | `*` (POC) or specific domains | `http://alb-url` for prod |
| **Methods** | `GET,POST,PUT,DELETE,OPTIONS` | |
| **Headers** | `Content-Type,Authorization,x-api-key` | Include x-api-key! |
| **Exposed Headers** | (leave empty) | |
| **Credentials** | OFF | Unless using cookies |
| **Max Age** | `3600` | Cache preflight for 1 hour |
| **Preflight Continue** | OFF | |

---

### 3.3 AWS Lambda Plugin

**Scope:** Apply to specific routes calling Lambda

#### For IAM Instance Role (No credentials needed!)

| Field | Value | Notes |
|-------|-------|-------|
| **Scope** | Scoped to Lambda routes | |
| **Aws Region** | `ap-south-1` | Your region |
| **Function Name** | `my-lambda-function` | Lambda function name |
| **Aws Key** | **LEAVE EMPTY** | Uses instance role! |
| **Aws Secret** | **LEAVE EMPTY** | Uses instance role! |
| **Aws Imds Protocol Version** | `v2` | Recommended for security |
| **Invocation Type** | `RequestResponse` | Synchronous |
| **Timeout** | `60000` | 60 seconds |
| **Forward Request Body** | ✅ ON | Pass request body |
| **Forward Request Headers** | ✅ ON | Pass headers |
| **Forward Request Method** | ✅ ON | Pass HTTP method |
| **Forward Request Uri** | ✅ ON | Pass path |
| **Is Proxy Integration** | ✅ ON | AWS API Gateway compatible |
| **Awsgateway Compatible** | ✅ ON | Event format like APIGW |

#### IAM Role Required on EC2

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": [
                "arn:aws:lambda:ap-south-1:ACCOUNT:function:*"
            ]
        }
    ]
}
```

---

## Part 4: Lambda Integration Options

### Option 1: Direct Lambda Invocation (Recommended)

```
Kong → AWS Lambda Plugin → Lambda (VPC)
```

**Pros:**
- Lower latency
- Uses IAM instance role
- No additional infrastructure

**Kong Route Config:**
- Service URL: `http://localhost` (placeholder, plugin overrides)
- Apply AWS Lambda plugin to route

---

### Option 2: Via Internal ALB (Existing Lambda Target Groups)

```
Kong → HTTP Service → Internal ALB → Lambda Target Group
```

**Pros:**
- Uses existing infrastructure
- No plugin needed

**Kong Route Config:**
- Service URL: `http://internal-alb-dns/lambda-path`
- No Lambda plugin needed

---

## Part 5: Complete Example Setup

### Service: user-service (Lambda)

```yaml
# Conceptual config (configure via Kong Manager UI)

Service:
  name: user-service
  url: http://localhost  # Placeholder for Lambda

Route:
  name: user-service-route
  paths: ["/users"]
  strip_path: false

Plugins on Route:
  - key-auth:
      key_names: [x-api-key]
      hide_credentials: true
  
  - cors:
      origins: ["*"]
      methods: [GET, POST, PUT, DELETE, OPTIONS]
      headers: [Content-Type, x-api-key]
  
  - aws-lambda:
      aws_region: ap-south-1
      function_name: user-service-lambda
      forward_request_body: true
      forward_request_headers: true
      forward_request_method: true
      forward_request_uri: true
      is_proxy_integration: true
      awsgateway_compatible: true
```

---

## Part 6: Testing

### Test via ALB URL

```bash
# Without API key (should fail if key-auth enabled)
curl http://aws-alb-url/api-gateway/users

# With API key
curl -H "x-api-key: your-api-key" \
  http://aws-alb-url/api-gateway/users

# POST request
curl -X POST \
  -H "x-api-key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{"name": "John"}' \
  http://aws-alb-url/api-gateway/users
```

---

## Summary: AWS API Gateway vs Kong Mapping

| AWS API Gateway | Kong Equivalent |
|-----------------|-----------------|
| `{proxy+}` | Prefix path in Route |
| API Key Required | Key Auth plugin |
| Usage Plan throttling | Rate Limiting plugin |
| CORS | CORS plugin |
| Lambda Integration | AWS Lambda plugin or HTTP to ALB |
| Stage (`/dev`) | Part of route path |
| Resource Policy | ACL + IP Restriction plugins |

---

## Recommended Implementation Order

1. ✅ Set up Nginx reverse proxy
2. ✅ Create Services in Kong
3. ✅ Create Routes (proxy-based first)
4. ✅ Enable CORS plugin globally
5. ✅ Enable Key Auth on protected routes
6. ✅ Configure AWS Lambda plugin (or HTTP to ALB)
7. ✅ Create Consumers and API keys
8. ✅ Test end-to-end via ALB URL
