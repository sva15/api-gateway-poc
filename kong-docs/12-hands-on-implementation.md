# Kong Hands-On Implementation Guide

Complete step-by-step guide with prerequisites, Lambda function, Nginx configuration, and Kong setup.

---

## Table of Contents

1. [Prerequisites & Infrastructure](#1-prerequisites--infrastructure)
2. [URL Translation Flow](#2-url-translation-flow)
3. [Lambda Function Setup](#3-lambda-function-setup)
4. [Nginx Configuration](#4-nginx-configuration)
5. [Kong Configuration](#5-kong-configuration)
6. [Testing All Functionality](#6-testing-all-functionality)

---

## 1. Prerequisites & Infrastructure

### 1.1 AWS Infrastructure Checklist

```
□ VPC with private subnets
□ EC2 instance in VPC
  ├── Security Group: Allow 80 from ALB
  ├── IAM Role: Lambda invoke permission
  └── Docker & Docker Compose installed

□ Internal ALB
  ├── Listener: Port 80
  └── Target Group: IP-based → EC2 private IP

□ Lambda Function
  ├── VPC attached (same VPC as EC2)
  ├── Python 3.11+ runtime
  └── Execution role with VPC access

□ ALB Lambda Target Group (existing)
  └── Routes: /dev/api1/*, /dev/api2/*
```

### 1.2 EC2 IAM Role Policy

Attach to EC2 instance role for Kong Lambda plugin:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "KongLambdaInvoke",
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": [
                "arn:aws:lambda:ap-south-1:ACCOUNT_ID:function:kong-poc-lambda",
                "arn:aws:lambda:ap-south-1:ACCOUNT_ID:function:kong-poc-*"
            ]
        }
    ]
}
```

### 1.3 Security Group Rules

**EC2 Security Group:**

| Type | Port | Source | Description |
|------|------|--------|-------------|
| Inbound | 80 | ALB SG | HTTP from ALB |
| Inbound | 22 | Your IP | SSH access |
| Outbound | 443 | 0.0.0.0/0 | Lambda API calls |
| Outbound | 80 | VPC CIDR | Internal traffic |

---

## 2. URL Translation Flow

### 2.1 Complete Request Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           URL TRANSLATION FLOW                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Browser Request:                                                           │
│   http://alb-url/api-gateway/users/123                                      │
│   Headers: x-api-key: abc123, Content-Type: application/json                │
│                        │                                                     │
│                        ▼                                                     │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │  ALB (Port 80)                                                        │  │
│   │  Rule: /* → IP Target Group (EC2:80)                                 │  │
│   │  Passes: Full path, all headers                                       │  │
│   └───────────────────────────┬──────────────────────────────────────────┘  │
│                               │                                              │
│                               ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │  NGINX (Port 80)                                                      │  │
│   │  location /api-gateway/ {                                             │  │
│   │      proxy_pass http://127.0.0.1:8000/;   ← NOTE: trailing /          │  │
│   │  }                                                                    │  │
│   │                                                                        │  │
│   │  Transforms: /api-gateway/users/123 → /users/123                     │  │
│   └───────────────────────────┬──────────────────────────────────────────┘  │
│                               │                                              │
│                               ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │  Kong Proxy (Port 8000)                                               │  │
│   │  Route: path="/users" matches "/users/123"                           │  │
│   │  Service: points to Lambda or ALB                                    │  │
│   │                                                                        │  │
│   │  Kong receives: GET /users/123                                        │  │
│   │  Kong passes to backend: /users/123 (if strip_path=false)            │  │
│   └───────────────────────────┬──────────────────────────────────────────┘  │
│                               │                                              │
│                               ▼                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │  Lambda Function                                                      │  │
│   │  Receives event with path: /users/123                                │  │
│   │  Flask routes to: GET /users/<id>                                    │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Port Mapping Summary

| External URL | Nginx Location | Kong Port | Purpose |
|--------------|----------------|-----------|---------|
| `/api-gateway/*` | proxy_pass :8000/ | 8000 | **API Proxy** (main traffic) |
| `/kong/*` | proxy_pass :8002/ | 8002 | Kong Manager GUI |
| `/kong-admin/*` | proxy_pass :8001/ | 8001 | Admin API (restrict!) |

### 2.3 Key Points

1. **Port 8000** = Kong Proxy (where APIs are accessed)
2. **Port 8001** = Kong Admin API (configuration, NOT for browsers)
3. **Port 8002** = Kong Manager (Admin GUI)

**Your UI calls:** `http://alb-url/api-gateway/users` → Nginx strips `/api-gateway` → Kong receives `/users`

---

## 3. Lambda Function Setup

### 3.1 Lambda Function Code (Flask-based)

Create Lambda function: `kong-poc-lambda`

```python
"""
Kong POC Lambda Function
Tests all HTTP methods and request handling

Deploy with: 
- Runtime: Python 3.11
- Handler: lambda_function.lambda_handler
- VPC: Your private VPC
"""

import json
from datetime import datetime

def lambda_handler(event, context):
    """
    Universal Lambda handler supporting both:
    - AWS API Gateway event format
    - Kong aws-lambda plugin event format
    - ALB event format
    """
    
    # Detect event source and normalize
    if 'requestContext' in event and 'elb' in event.get('requestContext', {}):
        # ALB event format
        method = event.get('httpMethod', 'GET')
        path = event.get('path', '/')
        headers = event.get('headers', {}) or {}
        query = event.get('queryStringParameters', {}) or {}
        body = event.get('body', '')
        source = 'ALB'
    elif 'httpMethod' in event:
        # AWS API Gateway / Kong Lambda plugin format
        method = event.get('httpMethod', 'GET')
        path = event.get('path', event.get('request_uri', '/'))
        headers = event.get('headers', {}) or {}
        query = event.get('queryStringParameters', {}) or {}
        body = event.get('body', '')
        source = 'API_GATEWAY_OR_KONG'
    elif 'request_method' in event:
        # Kong aws-lambda plugin raw format
        method = event.get('request_method', 'GET')
        path = event.get('request_uri', '/')
        headers = event.get('request_headers', {}) or {}
        query = {}  # Parse from URI if needed
        body = event.get('request_body', '')
        source = 'KONG_LAMBDA_PLUGIN'
    else:
        # Direct invocation
        method = 'POST'
        path = '/'
        headers = {}
        query = {}
        body = json.dumps(event)
        source = 'DIRECT'
    
    # Parse body if JSON
    if body:
        try:
            body = json.loads(body) if isinstance(body, str) else body
        except:
            pass
    
    # Route based on path and method
    response = route_request(method, path, headers, query, body, source)
    
    return response


def route_request(method, path, headers, query, body, source):
    """Simple Flask-like routing"""
    
    # Remove trailing slashes for consistency
    path = path.rstrip('/')
    
    # Extract path parameters (simple parsing)
    parts = path.split('/')
    
    # Routes
    routes = {
        # Health check
        ('GET', '/'): lambda: health_check(),
        ('GET', '/health'): lambda: health_check(),
        
        # Users CRUD
        ('GET', '/users'): lambda: list_users(query),
        ('POST', '/users'): lambda: create_user(body),
        ('GET', '/users/{id}'): lambda: get_user(parts[-1]),
        ('PUT', '/users/{id}'): lambda: update_user(parts[-1], body),
        ('DELETE', '/users/{id}'): lambda: delete_user(parts[-1]),
        
        # Orders CRUD
        ('GET', '/orders'): lambda: list_orders(query),
        ('POST', '/orders'): lambda: create_order(body),
        ('GET', '/orders/{id}'): lambda: get_order(parts[-1]),
        
        # Config endpoint
        ('GET', '/config'): lambda: get_config(),
        ('POST', '/config'): lambda: update_config(body),
    }
    
    # Match route
    route_key = (method, path)
    
    # Check exact match first
    if route_key in routes:
        result = routes[route_key]()
    # Check pattern match (e.g., /users/123)
    elif len(parts) >= 2:
        pattern_key = (method, f"/{parts[1]}/{{id}}")
        if pattern_key in routes:
            result = routes[pattern_key]()
        else:
            result = not_found(method, path)
    else:
        result = not_found(method, path)
    
    # Add metadata
    result['_metadata'] = {
        'source': source,
        'method': method,
        'path': path,
        'timestamp': datetime.utcnow().isoformat(),
        'headers_received': list(headers.keys()) if headers else []
    }
    
    return format_response(200, result)


# ============ Handler Functions ============

def health_check():
    return {
        'status': 'healthy',
        'service': 'kong-poc-lambda',
        'version': '1.0.0'
    }

def list_users(query):
    limit = int(query.get('limit', 10)) if query else 10
    return {
        'action': 'LIST_USERS',
        'users': [
            {'id': '1', 'name': 'Alice', 'email': 'alice@example.com'},
            {'id': '2', 'name': 'Bob', 'email': 'bob@example.com'},
            {'id': '3', 'name': 'Charlie', 'email': 'charlie@example.com'},
        ][:limit],
        'total': 3,
        'limit': limit
    }

def create_user(body):
    return {
        'action': 'CREATE_USER',
        'message': 'User created successfully',
        'user': {
            'id': '4',
            'name': body.get('name', 'Unknown') if body else 'Unknown',
            'email': body.get('email', 'unknown@example.com') if body else 'unknown@example.com'
        }
    }

def get_user(user_id):
    return {
        'action': 'GET_USER',
        'user': {
            'id': user_id,
            'name': f'User {user_id}',
            'email': f'user{user_id}@example.com',
            'created_at': '2026-01-01T00:00:00Z'
        }
    }

def update_user(user_id, body):
    return {
        'action': 'UPDATE_USER',
        'message': f'User {user_id} updated successfully',
        'user': {
            'id': user_id,
            'name': body.get('name', 'Updated') if body else 'Updated',
            'email': body.get('email', f'user{user_id}@example.com') if body else f'user{user_id}@example.com'
        }
    }

def delete_user(user_id):
    return {
        'action': 'DELETE_USER',
        'message': f'User {user_id} deleted successfully'
    }

def list_orders(query):
    return {
        'action': 'LIST_ORDERS',
        'orders': [
            {'id': 'ORD-001', 'user_id': '1', 'total': 99.99},
            {'id': 'ORD-002', 'user_id': '2', 'total': 149.99},
        ]
    }

def create_order(body):
    return {
        'action': 'CREATE_ORDER',
        'order': {
            'id': 'ORD-003',
            'user_id': body.get('user_id') if body else None,
            'items': body.get('items', []) if body else [],
            'total': 199.99
        }
    }

def get_order(order_id):
    return {
        'action': 'GET_ORDER',
        'order': {
            'id': order_id,
            'user_id': '1',
            'items': [{'name': 'Widget', 'qty': 2}],
            'total': 99.99
        }
    }

def get_config():
    return {
        'action': 'GET_CONFIG',
        'config': {
            'feature_flags': {
                'new_ui': True,
                'beta_api': False
            },
            'rate_limits': {
                'default': 100,
                'premium': 1000
            }
        }
    }

def update_config(body):
    return {
        'action': 'UPDATE_CONFIG',
        'message': 'Configuration updated',
        'config': body
    }

def not_found(method, path):
    return {
        'error': 'NOT_FOUND',
        'message': f'No handler for {method} {path}',
        'available_endpoints': [
            'GET /',
            'GET /health',
            'GET /users',
            'POST /users',
            'GET /users/{id}',
            'PUT /users/{id}',
            'DELETE /users/{id}',
            'GET /orders',
            'POST /orders',
            'GET /orders/{id}',
            'GET /config',
            'POST /config'
        ]
    }


def format_response(status_code, body):
    """Format response for ALB/API Gateway/Kong"""
    return {
        'statusCode': status_code,
        'headers': {
            'Content-Type': 'application/json',
            'X-Lambda-Function': 'kong-poc-lambda',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps(body, indent=2)
    }
```

### 3.2 Lambda Deployment

1. Create Lambda function in AWS Console
2. Copy the code above
3. Set handler: `lambda_function.lambda_handler`
4. Set runtime: Python 3.11
5. Attach to VPC (same as EC2)
6. Test with sample event

### 3.3 Test Events

**GET /users:**
```json
{
  "httpMethod": "GET",
  "path": "/users",
  "headers": {},
  "queryStringParameters": {"limit": "5"}
}
```

**POST /users:**
```json
{
  "httpMethod": "POST",
  "path": "/users",
  "headers": {"Content-Type": "application/json"},
  "body": "{\"name\": \"David\", \"email\": \"david@example.com\"}"
}
```

---

## 4. Nginx Configuration

### 4.1 Complete nginx.conf

```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    sendfile on;
    keepalive_timeout 65;
    client_max_body_size 10M;

    server {
        listen 80;
        server_name _;

        # ============================================
        # Kong API Proxy (Port 8000)
        # URL: http://alb-url/api-gateway/*
        # Translates to: http://127.0.0.1:8000/*
        # ============================================
        location /api-gateway/ {
            proxy_pass http://127.0.0.1:8000/;
            
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Prefix /api-gateway;
            
            # Pass through all headers
            proxy_pass_request_headers on;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # ============================================
        # Kong Manager GUI (Port 8002)
        # URL: http://alb-url/kong/
        # ============================================
        location /kong/ {
            proxy_pass http://127.0.0.1:8002/;
            
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # WebSocket support (for Kong Manager)
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # ============================================
        # Kong Admin API (Port 8001) - RESTRICTED!
        # Only allow from localhost or specific IPs
        # ============================================
        location /kong-admin/ {
            # Security: Restrict access
            allow 127.0.0.1;
            allow 10.0.0.0/8;  # Adjust to your VPC CIDR
            deny all;
            
            proxy_pass http://127.0.0.1:8001/;
            
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # ============================================
        # Angular UI (Port 4302) - If applicable
        # ============================================
        location /dev/configurator/ {
            proxy_pass http://127.0.0.1:4302/;
            
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Health check for ALB
        location /health {
            return 200 'OK';
            add_header Content-Type text/plain;
        }
    }
}
```

### 4.2 Reload Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 4.3 Verify Nginx is Working

```bash
# Test Kong proxy
curl http://localhost/api-gateway/

# Test Kong Manager
curl http://localhost/kong/

# Test health
curl http://localhost/health
```

---

## 5. Kong Configuration

### 5.1 Start Kong with Docker Compose

Use your existing `docker-compose.yaml`:

```bash
cd /path/to/kong
docker-compose up -d
```

### 5.2 Verify Kong is Running

```bash
# Check containers
docker ps

# Check Kong status
curl http://localhost:8001/status
```

### 5.3 Configure via Kong Manager UI

Access: `http://alb-url/kong/` or `http://localhost:8002`

---

### Step 1: Create Gateway Service

**Kong Manager → Gateway Services → New Gateway Service**

#### Option A: Direct Lambda (Recommended)

| Field | Value |
|-------|-------|
| Name | `poc-lambda-service` |
| URL | `http://localhost` |
| Retries | `3` |
| Connect Timeout | `60000` |

*(URL is placeholder, Lambda plugin will handle invocation)*

#### Option B: Via ALB Lambda Target

| Field | Value |
|-------|-------|
| Name | `poc-lambda-service` |
| URL | `http://internal-alb-xxx.region.elb.amazonaws.com` |
| Path | `/dev/api1` |
| Retries | `3` |

---

### Step 2: Create Routes

**Kong Manager → Routes → Create Route**

#### Route 1: Catch-All (Proxy Style)

| Field | Value |
|-------|-------|
| Name | `poc-all-routes` |
| Service | `poc-lambda-service` |
| Protocols | HTTP, HTTPS |
| Paths | `/` |
| Strip Path | `OFF` (unchecked) |
| Methods | (leave empty = all) |

#### Route 2: Specific Users Route (Fine-Grained)

| Field | Value |
|-------|-------|
| Name | `users-route` |
| Service | `poc-lambda-service` |
| Paths | `/users` |
| Methods | GET, POST, PUT, DELETE |
| Strip Path | `OFF` |

---

### Step 3: Create Consumer

**Kong Manager → Consumers → New Consumer**

| Field | Value |
|-------|-------|
| Username | `frontend-app` |
| Custom ID | (optional) |

**Add Credentials:**
Consumers → frontend-app → Credentials → New Key Auth Credential

Note the generated key (e.g., `abc123xyz890...`)

---

### Step 4: Enable Plugins

#### 4.1 Key Authentication (On Route)

**Routes → poc-all-routes → Plugins → Add Plugin → Key Authentication**

| Field | Value |
|-------|-------|
| Key Names | `x-api-key` |
| Key In Header | ON |
| Key In Query | OFF |
| Key In Body | OFF |
| Hide Credentials | ON |
| Run On Preflight | OFF |

#### 4.2 CORS (Global)

**Plugins → New Plugin → CORS → Global**

| Field | Value |
|-------|-------|
| Origins | `*` |
| Methods | `GET,POST,PUT,DELETE,OPTIONS` |
| Headers | `Content-Type,x-api-key,Authorization` |
| Max Age | `3600` |

#### 4.3 AWS Lambda Plugin (On Route - if using Option A)

**Routes → poc-all-routes → Plugins → Add Plugin → AWS Lambda**

| Field | Value |
|-------|-------|
| **Aws Region** | `ap-south-1` |
| **Function Name** | `kong-poc-lambda` |
| **Aws Key** | **LEAVE EMPTY** |
| **Aws Secret** | **LEAVE EMPTY** |
| Aws Imds Protocol Version | `v2` |
| Invocation Type | `RequestResponse` |
| Timeout | `60000` |
| **Forward Request Body** | ON |
| **Forward Request Headers** | ON |
| **Forward Request Method** | ON |
| **Forward Request Uri** | ON |
| **Is Proxy Integration** | ON |
| **Awsgateway Compatible** | ON |

---

## 6. Testing All Functionality

### 6.1 Test Commands

**Replace `YOUR_API_KEY` with your generated key.**

```bash
# Set variables
ALB_URL="http://your-alb-url"
API_KEY="your-api-key-here"

# ============================================
# Health Check
# ============================================
curl -H "x-api-key: $API_KEY" \
  "$ALB_URL/api-gateway/health"

# ============================================
# GET /users - List Users
# ============================================
curl -H "x-api-key: $API_KEY" \
  "$ALB_URL/api-gateway/users"

# ============================================
# GET /users?limit=2 - With Query Param
# ============================================
curl -H "x-api-key: $API_KEY" \
  "$ALB_URL/api-gateway/users?limit=2"

# ============================================
# POST /users - Create User
# ============================================
curl -X POST \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "New User", "email": "new@example.com"}' \
  "$ALB_URL/api-gateway/users"

# ============================================
# GET /users/123 - Get Specific User
# ============================================
curl -H "x-api-key: $API_KEY" \
  "$ALB_URL/api-gateway/users/123"

# ============================================
# PUT /users/123 - Update User
# ============================================
curl -X PUT \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Name"}' \
  "$ALB_URL/api-gateway/users/123"

# ============================================
# DELETE /users/123 - Delete User
# ============================================
curl -X DELETE \
  -H "x-api-key: $API_KEY" \
  "$ALB_URL/api-gateway/users/123"

# ============================================
# GET /orders - List Orders
# ============================================
curl -H "x-api-key: $API_KEY" \
  "$ALB_URL/api-gateway/orders"

# ============================================
# POST /orders - Create Order
# ============================================
curl -X POST \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "1", "items": [{"name": "Widget", "qty": 3}]}' \
  "$ALB_URL/api-gateway/orders"

# ============================================
# GET /config - Get Config
# ============================================
curl -H "x-api-key: $API_KEY" \
  "$ALB_URL/api-gateway/config"

# ============================================
# Test WITHOUT API Key (Should fail with 401)
# ============================================
curl "$ALB_URL/api-gateway/users"
# Expected: {"message":"No API key found in request"}
```

### 6.2 Expected Responses

**GET /users:**
```json
{
  "action": "LIST_USERS",
  "users": [
    {"id": "1", "name": "Alice", "email": "alice@example.com"},
    {"id": "2", "name": "Bob", "email": "bob@example.com"}
  ],
  "_metadata": {
    "source": "KONG_LAMBDA_PLUGIN",
    "method": "GET",
    "path": "/users"
  }
}
```

**POST /users:**
```json
{
  "action": "CREATE_USER",
  "message": "User created successfully",
  "user": {
    "id": "4",
    "name": "New User",
    "email": "new@example.com"
  }
}
```

### 6.3 Troubleshooting

| Issue | Check |
|-------|-------|
| 404 Not Found | Route path in Kong |
| 401 Unauthorized | API key correct? Key-auth plugin enabled? |
| 502 Bad Gateway | Lambda function accessible? IAM role? |
| 504 Timeout | Lambda timeout, Kong timeout |
| CORS errors | CORS plugin enabled with correct headers |

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         IMPLEMENTATION COMPLETE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. ✅ Lambda Function: kong-poc-lambda                                      │
│     └── 12 endpoints, all HTTP methods                                      │
│                                                                              │
│  2. ✅ Nginx Configuration                                                   │
│     ├── /api-gateway/* → Kong Proxy (8000)                                 │
│     ├── /kong/*        → Kong Manager (8002)                               │
│     └── /kong-admin/*  → Kong Admin (8001) RESTRICTED                      │
│                                                                              │
│  3. ✅ Kong Configuration                                                    │
│     ├── Gateway Service: poc-lambda-service                                 │
│     ├── Route: / (catch-all)                                                │
│     ├── Consumer: frontend-app + API key                                    │
│     └── Plugins: Key Auth, CORS, AWS Lambda                                 │
│                                                                              │
│  4. ✅ URL Translation                                                       │
│     http://alb/api-gateway/users/123                                        │
│       → Nginx strips /api-gateway                                            │
│       → Kong receives /users/123                                            │
│       → Lambda processes GET /users/123                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```
