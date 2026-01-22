# API Gateway Configuration Options - Team Decision Guide

**Purpose:** This document presents configuration options for routing and security in AWS API Gateway Private REST APIs. Use this to align with the team before implementation.

---

## How to Use This Document

1. Review each section with your team
2. Discuss trade-offs for your specific use case
3. Mark your decisions in the checkboxes
4. Use decisions to guide implementation

---

# Part 1: Routing Options

## Option A: Proxy-Based Routing (`{proxy+}`)

**How it works:** Single `{proxy+}` resource catches all subpaths and forwards to one Lambda.

```
/api1/{proxy+}  →  Lambda handles: /api1/users, /api1/orders, /api1/anything
```

### Pros
| Benefit | Description |
|---------|-------------|
| ✅ Simple setup | One resource, one integration |
| ✅ ALB migration | Direct equivalent of ALB `/*` routing |
| ✅ Flexible | Add new paths without API Gateway changes |
| ✅ Fast iteration | Backend controls all routing logic |

### Cons
| Limitation | Description |
|------------|-------------|
| ❌ No per-path auth | Can't have `/admin/*` require IAM while `/public/*` is open |
| ❌ No per-path throttling | Same rate limits for all paths |
| ❌ No request validation | API Gateway can't validate schemas |
| ❌ Poor documentation | OpenAPI shows only `{proxy+}`, not actual endpoints |
| ❌ All paths hit Lambda | Invalid paths still invoke Lambda (cost) |

### Best For
- POC / rapid development
- Migrating from ALB
- Backend frameworks with built-in routing (Flask, Express, FastAPI)
- APIs with frequently changing paths

---

## Option B: Explicit Path-Based Routing

**How it works:** Each endpoint is defined as a separate resource with specific methods.

```
/users          GET, POST        →  usersHandler
/users/{id}     GET, PUT, DELETE →  userDetailHandler
/orders         GET, POST        →  ordersHandler
```

### Pros
| Benefit | Description |
|---------|-------------|
| ✅ Per-path auth | Different authorization per endpoint |
| ✅ Per-path throttling | Custom rate limits per operation |
| ✅ Request validation | Schema validation before Lambda |
| ✅ Better documentation | Auto-generated OpenAPI specs |
| ✅ 404 at gateway | Invalid paths rejected before Lambda |
| ✅ Caching | Per-endpoint cache configuration |

### Cons
| Limitation | Description |
|------------|-------------|
| ❌ More configuration | Must define each resource/method |
| ❌ Deployment overhead | API Gateway changes needed for new paths |
| ❌ Slower iteration | More coordination between gateway and backend |

### Best For
- Production APIs with stable endpoints
- APIs requiring fine-grained security
- Public or partner-facing APIs
- APIs with compliance requirements

---

## Option C: Hybrid Approach (Recommended)

**How it works:** Combine explicit paths for sensitive endpoints + proxy for flexibility.

```
/admin/{proxy+}     →  adminLambda    (IAM auth, strict throttling)
/public/{proxy+}    →  publicLambda   (No auth, higher limits)
/health             →  healthLambda   (Explicit, no auth)
```

### Why Hybrid Works
| Aspect | Approach |
|--------|----------|
| Sensitive endpoints | Explicit paths with specific auth |
| General CRUD operations | `{proxy+}` for flexibility |
| Health/status endpoints | Explicit, always accessible |
| New features | Start with proxy, convert to explicit when stable |

---

## Routing Decision Matrix

| Criteria | Proxy | Explicit | Hybrid |
|----------|-------|----------|--------|
| Setup speed | ⭐⭐⭐ | ⭐ | ⭐⭐ |
| Per-path security | ❌ | ⭐⭐⭐ | ⭐⭐⭐ |
| Per-path throttling | ❌ | ⭐⭐⭐ | ⭐⭐ |
| Flexibility | ⭐⭐⭐ | ⭐ | ⭐⭐ |
| API documentation | ⭐ | ⭐⭐⭐ | ⭐⭐ |
| Maintenance | ⭐⭐⭐ | ⭐ | ⭐⭐ |

### Team Decision: Routing
```
□ Option A: Proxy-based (all {proxy+})
□ Option B: Explicit paths (all defined)
□ Option C: Hybrid (recommended)
```

**Notes/Rationale:**
_______________________________________

---

# Part 2: Security Options

## Layer 1: Network Security (Required for Private APIs)

### VPC Endpoint + Resource Policy

**What it does:** Restricts API access to requests from specific VPC endpoints only.

```json
{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "execute-api:Invoke",
    "Resource": "arn:aws:execute-api:region:account:api-id/*",
    "Condition": {
        "StringEquals": {
            "aws:SourceVpce": "vpce-xxxx"
        }
    }
}
```

| Aspect | Details |
|--------|---------|
| **Protection** | Only VPC traffic can reach API |
| **Overhead** | None (policy check only) |
| **Maintenance** | Update when VPC endpoints change |

> ✅ **Recommendation:** Always enable for private APIs. This is baseline security.

---

## Layer 2: Authentication Options

### Option 2A: No Authentication (Authorization = NONE)

**How it works:** Anyone in the allowed VPC can call the API.

| Pros | Cons |
|------|------|
| ✅ Simplest setup | ❌ No identity tracking |
| ✅ No client changes | ❌ No per-client access control |
| ✅ Works with browsers | ❌ Anyone in VPC has full access |

**Best for:** Internal tools, health checks, truly public internal APIs

---

### Option 2B: API Keys

**How it works:** Clients must pass `x-api-key` header.

| Pros | Cons |
|------|------|
| ✅ Client identification | ❌ NOT authentication (keys can be shared) |
| ✅ Rate limiting per client | ❌ Requires Usage Plan setup |
| ✅ Usage tracking | ❌ Key management overhead |
| ✅ Works with browsers | ❌ Keys visible in code/logs |

**Requirements:**
1. API Key Required = true on methods
2. Usage Plan created
3. API stage attached to Usage Plan
4. API Key associated with Usage Plan

**Best for:** Throttling, client-specific quotas, usage tracking

---

### Option 2C: IAM Authorization

**How it works:** Requests must be signed with AWS SigV4 using IAM credentials.

| Pros | Cons |
|------|------|
| ✅ Strong identity verification | ❌ Doesn't work for browsers |
| ✅ Fine-grained IAM policies | ❌ Requires SDK or SigV4 signing |
| ✅ CloudTrail audit trail | ❌ More complex client setup |
| ✅ Cross-account support | ❌ Not for UI applications |

**Best for:** Service-to-service calls, Lambda-to-API calls, backend integrations

---

### Option 2D: Lambda Authorizer

**How it works:** Custom Lambda validates tokens/credentials before request proceeds.

| Pros | Cons |
|------|------|
| ✅ Custom auth logic | ❌ Additional Lambda cost |
| ✅ Any token format | ❌ Adds latency |
| ✅ External IdP integration | ❌ More components to maintain |
| ✅ Works with browsers | ❌ Must implement caching |

**Best for:** Custom tokens, OAuth (non-Cognito), legacy auth systems

---

## Security Combinations for Internal APIs

### Combination 1: VPC Only (Simplest)
```
VPC Endpoint + Resource Policy
Authorization = NONE
No API Keys
```
**Use when:** Trust all VPC traffic, simplest internal tools

---

### Combination 2: VPC + API Keys (Recommended for most internal APIs)
```
VPC Endpoint + Resource Policy
Authorization = NONE
API Keys Required = true + Usage Plan
```
**Use when:** Need throttling, client identification, but no strict auth

---

### Combination 3: VPC + IAM (Service-to-Service)
```
VPC Endpoint + Resource Policy
Authorization = AWS_IAM
```
**Use when:** Only backend services call the API, need identity audit

---

### Combination 4: VPC + API Keys + IAM (Maximum Control)
```
VPC Endpoint + Resource Policy
Authorization = AWS_IAM
API Keys Required = true + Usage Plan
```
**Use when:** Need both identity verification AND rate limiting

---

## Security Decision Matrix

| Criteria | VPC Only | VPC + Keys | VPC + IAM | VPC + Keys + IAM |
|----------|----------|------------|-----------|------------------|
| Setup complexity | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| Identity tracking | ❌ | By key | By IAM role | Both |
| Rate limiting | ❌ | ✅ | ❌ | ✅ |
| Browser support | ✅ | ✅ | ❌ | ❌ |
| Audit trail | Basic | By key | CloudTrail | Full |
| Best for | Dev/POC | Most internal | Service-to-service | High security |

### Team Decision: Security
```
□ VPC Only (simplest, for POC)
□ VPC + API Keys (throttling, client tracking)
□ VPC + IAM (service-to-service)
□ VPC + API Keys + IAM (maximum control)
```

**Notes/Rationale:**
_______________________________________

---

# Part 3: Additional Considerations

## Browser-Based UI Applications

If any browser-based UI will call the API:

| Requirement | Configuration |
|-------------|---------------|
| **CORS** | Must enable on `{proxy+}` resources |
| **API Keys** | Works (pass via `x-api-key` header) |
| **IAM Auth** | ❌ Does NOT work (browsers can't sign SigV4) |
| **OPTIONS** | Must allow without auth for preflight |

### Team Decision: Browser Support
```
□ No browser clients (backend only)
□ Browser clients will call API (need CORS)
```

---

## Lambda-to-Lambda Communication

For internal Lambda-to-Lambda calls:

| Pattern | When to Use |
|---------|-------------|
| **Direct invoke** (`boto3 lambda.invoke`) | Synchronous calls, lowest latency |
| **Through API Gateway** | Only if Lambda is consumer-facing API |
| **SQS/SNS** | Async, decoupled communication |
| **Step Functions** | Orchestration, complex workflows |

> **Recommendation:** Do NOT route Lambda-to-Lambda through API Gateway unless it's a consumer-facing endpoint.

### Team Decision: Internal Communication
```
□ Direct Lambda invoke for internal calls
□ Event-driven (SQS/SNS) for async
□ Through API Gateway (only for specific reasons)
```

---

## Request Validation

| Level | What's Validated | Overhead |
|-------|------------------|----------|
| None | Nothing at gateway | Lowest |
| Query/Headers | Required params | Low |
| Body Schema | JSON schema | Medium |

> **Note:** Validation only works with explicit paths, not `{proxy+}`

### Team Decision: Validation
```
□ No gateway validation (Lambda validates)
□ Query/header validation only
□ Full body schema validation
```

---

# Summary: Decision Checklist

| Category | Decision | Owner |
|----------|----------|-------|
| **Routing Model** | □ Proxy / □ Explicit / □ Hybrid | |
| **Security Layer** | □ VPC Only / □ +Keys / □ +IAM / □ +Both | |
| **Browser Support** | □ Yes (CORS) / □ No | |
| **Internal Comms** | □ Direct / □ Events / □ API Gateway | |
| **Validation** | □ None / □ Params / □ Schema | |

---

## Next Steps After Team Alignment

1. Finalize decisions from this document
2. Update implementation based on choices
3. Review [05-hands-on-implementation.md](05-hands-on-implementation.md) for step-by-step guide
4. Reference [08-troubleshooting.md](08-troubleshooting.md) for common issues

---

**Document Version:** 1.0  
**Last Updated:** January 2026
