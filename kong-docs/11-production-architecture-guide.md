# Kong Gateway Production POC - Complete Architecture Guide

## Overview

This document provides a complete architecture and implementation guide for **Kong Gateway** as an API management layer in front of **AWS Lambda–based APIs**, based on the requirements for a production-grade POC.

---

## Table of Contents

1. [Architecture Diagram](#1-architecture-diagram)
2. [Lambda Invocation Strategy](#2-lambda-invocation-strategy)
3. [Kong Configuration](#3-kong-configuration)
4. [Routing Strategy](#4-routing-strategy)
5. [Request Validation](#5-request-validation)
6. [End-to-End Request Flow](#6-end-to-end-request-flow)
7. [Production Best Practices](#7-production-best-practices)
8. [Security Considerations](#8-security-considerations)
9. [Pros & Cons Analysis](#9-pros--cons-analysis)

---

## 1. Architecture Diagram

### Current Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          CURRENT STATE                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│   Browser                                                                 │
│      │                                                                    │
│      ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐    │
│   │                     Internal ALB                                 │    │
│   │                                                                  │    │
│   │  /dev/api1/*           → Lambda Target Group → Lambda 1 (VPC)   │    │
│   │  /dev/api2/*           → Lambda Target Group → Lambda 2 (VPC)   │    │
│   │  /dev/configurator/*   → IP Target Group → EC2:80 → Angular     │    │
│   └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│   EC2 Instance:                                                           │
│   ┌─────────────────────────────────────────────────────────────────┐    │
│   │  NGINX (Port 80) → Angular App (Port 4302)                      │    │
│   └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### Proposed Architecture (With Kong)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          PROPOSED STATE                                   │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│   Browser                                                                 │
│      │                                                                    │
│      ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐    │
│   │                     Internal ALB                                 │    │
│   │                                                                  │    │
│   │  /dev/api1/*           → IP Target Group → NGINX → Kong Proxy   │    │
│   │  /dev/api2/*           → IP Target Group → NGINX → Kong Proxy   │    │
│   │  /kong/*               → IP Target Group → NGINX → Kong Manager │    │
│   │  /dev/configurator/*   → IP Target Group → NGINX → Angular      │    │
│   └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│   EC2 Instance:                                                           │
│   ┌─────────────────────────────────────────────────────────────────┐    │
│   │                                                                  │    │
│   │   NGINX (Port 80)                                                │    │
│   │   ├── /dev/api1/*       → proxy_pass http://127.0.0.1:8000      │    │
│   │   ├── /dev/api2/*       → proxy_pass http://127.0.0.1:8000      │    │
│   │   ├── /kong/*           → proxy_pass http://127.0.0.1:8002      │    │
│   │   └── /dev/configurator → proxy_pass http://127.0.0.1:4302      │    │
│   │                                                                  │    │
│   │   Kong Gateway (Docker)                                          │    │
│   │   ├── Port 8000: Proxy (API requests)                           │    │
│   │   ├── Port 8001: Admin API (internal only)                      │    │
│   │   └── Port 8002: Kong Manager (Admin GUI)                       │    │
│   │                            │                                     │    │
│   │                            ▼                                     │    │
│   │                   ┌────────────────────┐                        │    │
│   │                   │   INVOCATION       │                        │    │
│   │                   │   CHOICE POINT     │                        │    │
│   │                   └─────────┬──────────┘                        │    │
│   │                             │                                    │    │
│   │            ┌────────────────┼────────────────┐                  │    │
│   │            ▼                ▼                ▼                   │    │
│   │      ┌──────────┐    ┌──────────┐    ┌──────────────┐           │    │
│   │      │  Option  │    │  Option  │    │    Option    │           │    │
│   │      │    A     │    │    B     │    │      C       │           │    │
│   │      │  Lambda  │    │   ALB    │    │  Function    │           │    │
│   │      │  Plugin  │    │  Lambda  │    │    URLs      │           │    │
│   │      │          │    │  Target  │    │  (IAM Auth)  │           │    │
│   │      └────┬─────┘    └────┬─────┘    └──────┬───────┘           │    │
│   │           │               │                 │                    │    │
│   └───────────┼───────────────┼─────────────────┼────────────────────┘    │
│               │               │                 │                         │
│               ▼               ▼                 ▼                         │
│         ┌──────────────────────────────────────────────────────┐         │
│         │              Lambda Functions (VPC Private)           │         │
│         │              Flask-based, handles subpaths            │         │
│         └──────────────────────────────────────────────────────┘         │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Lambda Invocation Strategy

### Available Options

| Option | Description | Security | Latency | Complexity |
|--------|-------------|----------|---------|------------|
| **A. Kong Lambda Plugin** | Kong directly invokes Lambda via AWS SDK | IAM role on EC2 | Low (~50ms) | Medium |
| **B. ALB Lambda Target** | Kong → ALB → Lambda Target Group | VPC internal | Medium (~100ms) | Low |
| **C. Function URLs + IAM** | Kong → Lambda Function URL with SigV4 | IAM auth | Low (~30ms) | High |

---

### Option A: Kong AWS Lambda Plugin (RECOMMENDED)

```
Kong → AWS Lambda Plugin → Lambda (VPC)
```

**How it works:**
- Kong uses AWS SDK to directly invoke Lambda
- EC2 IAM instance role provides credentials
- No additional network hops

**Configuration:**
```yaml
plugins:
  - name: aws-lambda
    config:
      aws_region: ap-south-1
      function_name: my-lambda
      # Leave aws_key and aws_secret EMPTY
      # Uses EC2 instance IAM role automatically
      forward_request_body: true
      forward_request_headers: true
      forward_request_method: true
      forward_request_uri: true
      is_proxy_integration: true
      awsgateway_compatible: true
```

**IAM Role Policy (on EC2):**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:ap-south-1:ACCOUNT:function:*"
        }
    ]
}
```

**Pros:**
- ✅ Direct invocation, lowest latency
- ✅ No additional infrastructure
- ✅ Uses secure IAM role credentials
- ✅ Full request context passed to Lambda

**Cons:**
- ❌ Requires IAM role configuration
- ❌ Kong must be in VPC or have Lambda access

---

### Option B: ALB Lambda Target Group

```
Kong → HTTP Request → ALB → Lambda Target Group → Lambda
```

**How it works:**
- Kong sends HTTP request to internal ALB endpoint
- ALB routes to Lambda target group
- Uses existing infrastructure

**Kong Configuration:**
```yaml
services:
  - name: lambda-via-alb
    url: http://internal-alb-dns.region.elb.amazonaws.com
    routes:
      - name: api1-route
        paths: ["/dev/api1"]
        strip_path: false
```

**Pros:**
- ✅ Uses existing ALB infrastructure
- ✅ No Kong Lambda plugin needed
- ✅ Simple HTTP routing

**Cons:**
- ❌ Additional network hop (Kong → ALB → Lambda)
- ❌ Higher latency (~50-100ms extra)
- ❌ Kong cannot validate requests fully before ALB

---

### Option C: Lambda Function URLs + IAM Auth

```
Kong → HTTPS → Lambda Function URL (IAM auth) → Lambda
```

**How it works:**
- Enable Function URL on Lambda with IAM auth type
- Kong signs requests with SigV4
- Requires custom plugin or AWS Lambda plugin with URL mode

**Pros:**
- ✅ Direct HTTPS to Lambda
- ✅ No ALB needed for new Lambdas

**Cons:**
- ❌ Must enable Function URLs on each Lambda
- ❌ Requires SigV4 signing in Kong
- ❌ More complex configuration

---

### RECOMMENDATION

| Scenario | Recommended Option |
|----------|-------------------|
| **New setup, full control** | **Option A: Kong Lambda Plugin** |
| **Existing ALB with Lambda TGs** | **Option B: ALB Lambda Target** (start here) |
| **Gradual migration** | Start with B, migrate to A |

**For your setup:** Start with **Option B** (ALB Lambda Target) since you already have working ALB → Lambda routing. This allows you to:
1. Add Kong as security layer immediately
2. No Lambda changes needed
3. Migrate to direct Lambda invocation later if needed

---

## 3. Kong Configuration

### 3.1 Gateway Service Configuration

#### For ALB Lambda Target (Option B)

```yaml
Service:
  name: api1-service
  url: http://internal-alb-1234.ap-south-1.elb.amazonaws.com
  retries: 3
  connect_timeout: 60000
  read_timeout: 60000
  write_timeout: 60000
```

| Field | Value | Description |
|-------|-------|-------------|
| **Name** | `api1-service` | Unique identifier |
| **URL** | ALB DNS name | Your internal ALB endpoint |
| **Retries** | `3` | Retry on failure |
| **Timeouts** | `60000` | 60 seconds |

#### For Direct Lambda (Option A)

```yaml
Service:
  name: api1-lambda-service
  url: http://localhost  # Placeholder, Lambda plugin overrides
```

---

### 3.2 Route Configuration

#### Proxy-Style Route (Like {proxy+})

```yaml
Route:
  name: api1-proxy-route
  service: api1-service
  paths: ["/dev/api1"]
  strip_path: false
  methods: []  # Empty = all methods
  protocols: [http, https]
  preserve_host: false
```

| Field | Value | Why |
|-------|-------|-----|
| **paths** | `/dev/api1` | Matches `/dev/api1/*` |
| **strip_path** | `false` | Keep full path for Lambda |
| **methods** | empty | Accept all HTTP methods |

#### Fine-Grained Route

```yaml
Route:
  name: users-get-route
  service: users-service
  paths: ["/dev/api1/users"]
  methods: [GET]
  strip_path: false
```

---

### 3.3 Consumer Configuration

```yaml
Consumer:
  username: frontend-app
  custom_id: angular-configurator
  tags: [internal, v1]

Credentials:
  key-auth:
    key: your-api-key-here
```

---

### 3.4 Plugin Configurations

#### Key Authentication

| Field | Value | Notes |
|-------|-------|-------|
| **Scope** | Per route (protected routes only) | |
| **key_names** | `x-api-key` | Header name |
| **key_in_header** | `true` | |
| **key_in_query** | `false` | Disable |
| **key_in_body** | `false` | Disable |
| **hide_credentials** | `true` | Don't pass to backend |
| **run_on_preflight** | `false` | Skip OPTIONS |

#### CORS

| Field | Value | Notes |
|-------|-------|-------|
| **Scope** | Global | |
| **origins** | `*` or specific ALB URL | |
| **methods** | `GET,POST,PUT,DELETE,OPTIONS` | |
| **headers** | `Content-Type,x-api-key` | |
| **max_age** | `3600` | Cache 1 hour |

> **Note:** Since UI and APIs share same ALB hostname, CORS may not be strictly required. Enable as a best practice.

#### AWS Lambda Plugin (If using Option A)

| Field | Value | Notes |
|-------|-------|-------|
| **aws_region** | `ap-south-1` | |
| **function_name** | `my-lambda` | Lambda name |
| **aws_key** | **EMPTY** | Uses IAM role |
| **aws_secret** | **EMPTY** | Uses IAM role |
| **aws_imds_protocol_version** | `v2` | More secure |
| **forward_request_body** | `true` | |
| **forward_request_headers** | `true` | |
| **forward_request_method** | `true` | |
| **forward_request_uri** | `true` | |
| **is_proxy_integration** | `true` | APIGW format |
| **awsgateway_compatible** | `true` | APIGW event |

#### Request Validator Plugin

| Field | Value | Notes |
|-------|-------|-------|
| **Scope** | Per route (fine-grained routes) | |
| **body_schema** | JSON Schema | Validate body |
| **allowed_content_types** | `application/json` | |
| **parameter_schema** | Schema for params | |

---

## 4. Routing Strategy

### 4.1 Proxy-Style Routing (Coarse-Grained)

**Equivalent to AWS API Gateway `{proxy+}`**

```
Route: /dev/api1
Matches: /dev/api1/*, /dev/api1/users, /dev/api1/users/123, etc.
Methods: ALL (leave empty)
```

**How Kong handles it:**
- Kong uses **prefix matching** by default
- Path `/dev/api1` matches any request starting with `/dev/api1`
- All HTTP methods are accepted

**Best for:**
- Lambda handles internal routing (Flask, Express)
- Rapid development
- Existing APIs being proxied

**Tradeoffs:**
| Benefit | Limitation |
|---------|------------|
| ✅ Simple setup | ❌ No per-path auth |
| ✅ One route per service | ❌ No per-path rate limiting |
| ✅ Lambda controls routing | ❌ No request validation at Kong |

---

### 4.2 Fine-Grained Routing (Per Method)

**Explicit routes with validation**

```yaml
# Route 1: GET /dev/api1/users
Route:
  name: users-list
  paths: ["/dev/api1/users"]
  methods: [GET]
  plugins:
    - rate-limiting: {minute: 100}

# Route 2: POST /dev/api1/users
Route:
  name: users-create
  paths: ["/dev/api1/users"]
  methods: [POST]
  plugins:
    - key-auth: {}
    - request-validator: {body_schema: ...}
    - rate-limiting: {minute: 10}

# Route 3: DELETE /dev/api1/users/{id}
Route:
  name: users-delete
  paths: ["/dev/api1/users"]  # Prefix match
  methods: [DELETE]
  plugins:
    - key-auth: {}
    - acl: {allow: [admin]}
```

**Best for:**
- Different auth per endpoint
- Per-method rate limiting
- Request body validation
- Compliance requirements

---

### 4.3 Hybrid Approach (RECOMMENDED)

Mix both styles for maximum flexibility.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        HYBRID ROUTING                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Fine-Grained Routes (Higher Priority):                                  │
│  ├── POST /dev/api1/users     → users-create (with validation)         │
│  ├── DELETE /dev/api1/admin/* → admin-delete (with ACL)                │
│  └── GET /dev/api1/config     → config-get (public)                    │
│                                                                          │
│  Proxy Route (Lower Priority - Catch-All):                              │
│  └── /dev/api1/*              → api1-proxy (all other requests)        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Kong Route Priority:**
1. Exact path + method match (highest)
2. Longer prefix match
3. Shorter prefix match (lowest)

---

## 5. Request Validation

### 5.1 Where Validation Should Happen

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      VALIDATION LAYERS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Layer 1: Kong (Gateway)                                                 │
│  ├── API Key validation                                                  │
│  ├── Rate limiting                                                       │
│  ├── Request body schema (if fine-grained routes)                       │
│  ├── Required headers check                                              │
│  └── IP restrictions                                                     │
│                                                                          │
│  Layer 2: Lambda (Application)                                           │
│  ├── Business logic validation                                           │
│  ├── Authorization (user permissions)                                    │
│  ├── Data integrity checks                                               │
│  └── Complex conditional validation                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Validation Options in Kong

| Validation Type | Kong Plugin | Notes |
|-----------------|-------------|-------|
| Required headers | Request Validator | |
| Required query params | Request Validator | |
| Body JSON schema | Request Validator | |
| Content-Type | Request Validator | |
| Request size | Request Size Limiting | |
| IP whitelist | IP Restriction | |

### 5.3 Request Validator Plugin Example

```yaml
plugins:
  - name: request-validator
    route: users-create
    config:
      body_schema: |
        {
          "type": "object",
          "required": ["name", "email"],
          "properties": {
            "name": {
              "type": "string",
              "minLength": 1,
              "maxLength": 100
            },
            "email": {
              "type": "string",
              "format": "email"
            }
          }
        }
      allowed_content_types:
        - application/json
```

### 5.4 Limitation with ALB-Based Lambda Routing

**Problem:**
If Kong → ALB → Lambda (Option B), Kong validates request, then ALB routes to Lambda. ALB adds its own headers but doesn't modify the request based on Kong validation.

**Solution:**
- Validation at Kong layer blocks invalid requests before ALB
- Kong returns 400 Bad Request for invalid bodies
- Lambda never receives invalid requests

**Full Request Governance:**
- Use **fine-grained routes** for sensitive endpoints
- Apply Request Validator plugin per route
- Let proxy routes pass through with minimal validation

---

## 6. End-to-End Request Flow

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    END-TO-END REQUEST FLOW                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Browser sends request:                                               │
│     GET http://alb-url/dev/api1/users                                   │
│     Headers: x-api-key: abc123, Content-Type: application/json          │
│                                │                                         │
│                                ▼                                         │
│  2. Internal ALB receives request                                        │
│     Rule: /dev/api1/* → IP Target Group (EC2)                           │
│                                │                                         │
│                                ▼                                         │
│  3. NGINX (Port 80) on EC2                                               │
│     location /dev/api1/ {                                                │
│       proxy_pass http://127.0.0.1:8000/dev/api1/;                       │
│       proxy_set_header X-Forwarded-For $remote_addr;                    │
│     }                                                                    │
│                                │                                         │
│                                ▼                                         │
│  4. Kong Proxy (Port 8000)                                               │
│     a. Route matching: /dev/api1 matches api1-proxy-route               │
│     b. Plugin execution (in order):                                      │
│        i.   Key Auth → Validate x-api-key header                        │
│        ii.  Rate Limiting → Check consumer limits                       │
│        iii. Request Validator → Check body schema (if configured)       │
│     c. If all pass → Forward to Service                                 │
│                                │                                         │
│                                ▼                                         │
│  5a. Option A: Lambda Plugin                    │                        │
│      Kong → AWS SDK → Lambda:InvokeFunction     │                        │
│      Uses EC2 IAM Role credentials              │                        │
│                                                 │                        │
│  5b. Option B: ALB Lambda Target                │                        │
│      Kong → HTTP → Internal ALB → Lambda TG     │                        │
│      Uses internal VPC networking               │                        │
│                                │                                         │
│                                ▼                                         │
│  6. Lambda Function (VPC Private)                                        │
│     Flask app receives event:                                            │
│     {                                                                    │
│       "httpMethod": "GET",                                               │
│       "path": "/dev/api1/users",                                        │
│       "headers": {"x-api-key": "[removed]", ...},                       │
│       "body": null                                                       │
│     }                                                                    │
│     Flask routes internally → returns response                          │
│                                │                                         │
│                                ▼                                         │
│  7. Response flows back:                                                 │
│     Lambda → Kong → NGINX → ALB → Browser                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Headers at Each Stage

| Stage | Headers Present |
|-------|-----------------|
| Browser → ALB | `x-api-key`, `Content-Type` |
| ALB → NGINX | + `X-Forwarded-For`, `X-Forwarded-Proto` |
| NGINX → Kong | + `Host: 127.0.0.1:8000` |
| Kong → Backend | - `x-api-key` (hidden), + `X-Consumer-Username` |

### IAM Role Usage

- EC2 instance has IAM role attached
- Role has `lambda:InvokeFunction` permission
- Kong Lambda plugin uses instance metadata (IMDS) to get credentials
- No hardcoded credentials needed

---

## 7. Production Best Practices

### 7.1 Deployment

| Practice | Implementation |
|----------|----------------|
| High availability | Run multiple Kong instances behind ALB |
| Database | Use PostgreSQL in RDS (not SQLite) |
| Secrets | Use Kong Vaults or AWS Secrets Manager |
| Configuration | DB mode for live updates, DB-less for immutable deploys |

### 7.2 Monitoring

| Tool | Purpose |
|------|---------|
| Prometheus plugin | Export metrics |
| File Log plugin | Access logs |
| CloudWatch | Centralized logging |
| Health checks | ALB health probes |

### 7.3 Security Checklist

```
□ Admin API (8001) NOT exposed to ALB
□ Kong Manager (8002) restricted to admin IPs
□ API keys stored securely
□ TLS for all external traffic
□ IAM role least privilege
□ Rate limiting enabled
□ Request size limits set
```

---

## 8. Security Considerations

### 8.1 Network Security

| Layer | Protection |
|-------|------------|
| ALB | Internal only, no public access |
| NGINX | Restricts /kong-admin to localhost |
| Kong | Admin API bound to 127.0.0.1 |
| Lambda | VPC private subnets, no public access |

### 8.2 Authentication Flow

```
Browser → x-api-key header → Kong Key Auth Plugin → Validate → Pass/Reject
```

### 8.3 Authorization

| Method | Kong Plugin |
|--------|-------------|
| API Key | Key Authentication |
| Role-based | ACL Plugin |
| IP-based | IP Restriction |

### 8.4 Data Protection

| Concern | Solution |
|---------|----------|
| Key in logs | Hide Credentials = true |
| Sensitive response | Response Transformer plugin |
| Request body | Request Validator plugin |

---

## 9. Pros & Cons Analysis

### Kong Gateway Pros

| Benefit | Details |
|---------|---------|
| ✅ Full API control | Security, validation, routing |
| ✅ No AWS APIGW costs | Pay only for EC2 |
| ✅ Flexible routing | Proxy + fine-grained |
| ✅ Plugin ecosystem | 100+ plugins |
| ✅ Single deployment | Same EC2 as UI |
| ✅ Portable | Can move off AWS |

### Kong Gateway Cons

| Limitation | Mitigation |
|------------|------------|
| ❌ Operational overhead | Use Docker Compose, automation |
| ❌ Lambda plugin config | Use IAM roles, document well |
| ❌ Not serverless | Accept EC2 management |
| ❌ Learning curve | Documentation, POC first |

### When to Choose Kong Over AWS APIGW

| Scenario | Recommendation |
|----------|----------------|
| High traffic, cost sensitive | Kong |
| Need header-based routing | Kong |
| Already have ALB infrastructure | Kong |
| Multi-cloud strategy | Kong |
| Prefer AWS managed | AWS APIGW |
| Lambda-heavy, low traffic | AWS APIGW |

---

## 10. Quick Start Checklist

```
□ Phase 1: Infrastructure
  ├── □ Kong Docker Compose running on EC2
  ├── □ NGINX configured with proxy_pass
  ├── □ ALB rule pointing to EC2 target group
  └── □ IAM role with Lambda invoke permission

□ Phase 2: Kong Configuration
  ├── □ Create Gateway Service (ALB URL)
  ├── □ Create Proxy Route (/dev/api1)
  ├── □ Enable Key Auth plugin
  ├── □ Enable CORS plugin
  └── □ Create Consumer with API key

□ Phase 3: Testing
  ├── □ Test via Kong directly (localhost:8000)
  ├── □ Test via NGINX (localhost:80)
  ├── □ Test via ALB (alb-url/dev/api1)
  └── □ Verify Lambda receives correct event

□ Phase 4: Hardening
  ├── □ Enable rate limiting
  ├── □ Add request validation
  ├── □ Configure logging
  └── □ Restrict Admin API access
```

---

**This document provides the complete architecture for your Kong Gateway POC. Follow the Quick Start Checklist to implement step by step.**
