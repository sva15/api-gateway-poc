# Troubleshooting & Common Issues

This document covers **real-world issues encountered** during implementation with their root causes and resolutions.

---

## Issue 1: VPC Endpoint Creation – Subnets Not Visible

### Symptom
While creating an Interface VPC Endpoint for `execute-api`, subnets from a newly created VPC were **not visible**, even though they were in the same region.

### Root Cause
New VPC subnets were **not eligible for interface endpoint ENI creation** due to configuration constraints. Default VPC subnets work because they are preconfigured and fully compatible.

### Resolution

Interface endpoints require:
- ✅ Valid IPv4 CIDR
- ✅ Available IPv4 addresses (not exhausted)
- ✅ **DNS hostnames enabled** at VPC level
- ✅ **DNS resolution enabled** at VPC level
- ✅ Standard AZs (not Local Zones)

**To fix:**
1. Go to VPC Console → Your VPC → Edit VPC settings
2. Enable **DNS hostnames** ✅
3. Enable **DNS resolution** ✅
4. Ensure subnets have available IP addresses

**Workaround:** Use default VPC for POC, revisit custom VPC with correct settings later.

---

## Issue 2: Stage and Resource Path Duplication (`/dev/dev/...`)

### Symptom
Created both a **stage named `dev`** and a **resource path `/dev`**, resulting in URLs like:
```
/dev/dev/api1/test
```

### Root Cause
API Gateway **automatically injects the stage name** into the URL. Creating a `/dev` resource duplicates it.

### URL Structure
```
https://{api-id}.execute-api.{region}.amazonaws.com/{stage}/{resource-path}
                                                    ↑
                                              Auto-injected
```

### Resolution
- ❌ Do NOT create a `/dev` resource
- ✅ Create resources under root directly: `/api1`, `/api2`
- ✅ Use stage name `dev` for environment separation

**Correct structure:**
```
/                      ← Root
├── /api1/{proxy+}     ← Resource
└── /api2/{proxy+}     ← Resource

Stage: dev

Final URL: /dev/api1/test  ← Correct!
```

---

## Issue 3: 403 Errors – "Explicit Deny in Resource-Based Policy"

### Symptom
Requests from EC2 inside the VPC returned:
```
explicit deny in a resource-based policy
```

### Root Cause

**Wrong condition used:**
```json
"aws:SourceVpc": "vpc-xxx"  ← WRONG for private APIs
```

For **private API Gateway**, requests originate from the **VPC Endpoint ENI**, not directly from the VPC. The `aws:SourceVpc` condition doesn't match.

Also, explicit `Deny` always overrides `Allow`.

### Resolution

**Correct approach – use `aws:SourceVpce`:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:REGION:ACCOUNT:API_ID/*",
            "Condition": {
                "StringEquals": {
                    "aws:SourceVpce": "vpce-xxxx"
                }
            }
        }
    ]
}
```

> **Key Learning:** For private APIs, use `aws:SourceVpce`, NOT `aws:SourceVpc`

---

## Issue 4: Incorrect Resource ARN in Resource Policy

### Symptom
Resource policy not working as expected.

### Root Cause
Used incomplete or overly broad ARN.

### Resolution

**Correct ARN format:**
```
arn:aws:execute-api:{region}:{account}:{api-id}/{stage}/{method}/{resource-path}
```

**Examples:**
| ARN Pattern | Meaning |
|-------------|---------|
| `arn:...:api-id/*` | All stages, methods, resources |
| `arn:...:api-id/dev/*/*` | Only `dev` stage, all methods/resources |
| `arn:...:api-id/dev/GET/*` | Only `dev` stage, only GET |

---

## Issue 5: EC2 IAM Role Shows "User: anonymous"

### Symptom
EC2 instance had IAM role with `execute-api:Invoke` permission, but API Gateway showed:
```
User: anonymous
```

### Root Cause
API methods had **Authorization = NONE**.

IAM roles are **ignored** unless:
1. Method authorization = **AWS_IAM**
2. Request is **SigV4-signed**

### Resolution
This is **expected behavior**. For private APIs with `Authorization = NONE`:
- Resource policy + VPC endpoint controls access
- EC2 IAM role is irrelevant (not evaluated)

Only enable IAM auth if you need identity-based access control.

---

## Issue 6: API Key Not Enforced Even When Enabled

### Symptom
After enabling **"API Key Required"** on methods, APIs were still invoked **without an API key**.

### Root Cause

API key enforcement requires **ALL** of these:
1. ✅ Method setting: API Key Required = true
2. ✅ Usage Plan exists
3. ✅ API stage is attached to Usage Plan
4. ✅ API Key is associated with Usage Plan
5. ✅ Request includes valid `x-api-key` header

If any step is missing, API Gateway **ignores** the API key requirement.

### Resolution

**Complete setup:**
1. Create Usage Plan
2. Attach your API + stage to Usage Plan
3. Create API Key
4. Associate API Key with Usage Plan
5. Redeploy the API

> **Key Learning:** "API Key Required" checkbox only marks the method as *eligible*. **Usage Plans** enforce the actual requirement.

---

## Issue 7: Browser UI Calls Failed – CORS Error

### Symptom
Browser-based UI (running on EC2 behind internal ALB) calling private API Gateway failed with CORS errors, even though both are internal.

### Root Cause
CORS is enforced by **web browsers**, not by AWS networking.

- UI origin: `http://alb-internal-url`
- API origin: `https://xxx.execute-api.region.amazonaws.com`
- **Different origins = CORS required**

VPC isolation does NOT bypass CORS.

### Resolution

Enable CORS on callable resources (`/api1/{proxy+}`, `/api2/{proxy+}`):

1. Select resource → Actions → Enable CORS
2. Configure:
   - `Access-Control-Allow-Origin`: `*` (or specific origin)
   - `Access-Control-Allow-Headers`: `Content-Type, Authorization, X-Api-Key`
   - `Access-Control-Allow-Methods`: `GET, POST, PUT, DELETE, OPTIONS`
3. Also configure CORS for Gateway Responses (4XX, 5XX)
4. **Redeploy API**

> **Key Learning:** If a browser is involved, CORS is mandatory—regardless of VPC isolation.

---

## Issue 8: IAM Auth Doesn't Work for Browser Calls

### Symptom
Enabled IAM authorization, but browser calls fail even with valid EC2 IAM role.

### Root Cause
- IAM auth requires **SigV4-signed requests**
- Browsers do NOT automatically sign requests
- EC2 IAM role is used by server-side code, NOT browser JavaScript

### Resolution

For browser-based internal UIs:
- ❌ Don't use IAM authorization
- ✅ Use: VPC Endpoint restriction + Resource Policy + Optional API Keys

If IAM auth is required:
- Use Backend-for-Frontend (BFF) pattern
- Backend signs requests using SDK
- Or use Cognito for browser auth

> **Key Learning:** IAM auth is for service-to-service calls, not browser clients.

---

## Issue 9: Lambda-to-Lambda Communication Design

### Question
Should Lambda-to-Lambda calls go through API Gateway?

### Answer
**No.** API Gateway is an entry point for consumers, not internal communication.

**Recommended patterns:**

| Scenario | Pattern |
|----------|---------|
| Synchronous Lambda-to-Lambda | Direct invoke via boto3 `lambda_client.invoke()` |
| Asynchronous/decoupled | SQS, SNS, EventBridge |
| Orchestration | Step Functions |

**Why direct invoke is better:**
- Lower latency
- Lower cost
- IAM-based security
- No API Gateway overhead

> **Key Learning:** API Gateway doesn't affect Lambda behavior. After invocation, Lambda works the same as before (can access VPC resources, databases, etc.)

---

## Issue 10: Proxy vs Explicit Resources – When to Use Which

### Guidance

| Use Case | Approach |
|----------|----------|
| POC / rapid development | `{proxy+}` |
| ALB migration | `{proxy+}` |
| Fine-grained auth per path | Explicit resources |
| Per-method throttling | Explicit resources |
| Request validation | Explicit resources |
| Production-stable APIs | Explicit resources |

**Hybrid approach:**
- Start with `{proxy+}` for speed
- Add explicit resources as APIs stabilize
- Keep `{proxy+}` as fallback

> **Key Learning:** Start broad with proxy, tighten with explicit paths as needed.

---

## Quick Troubleshooting Guide

### 403 Forbidden Errors

```
┌─────────────────────────────────────────────────────────────┐
│                   403 TROUBLESHOOTING                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Is resource policy using aws:SourceVpce (not SourceVpc)?│
│     └── Private APIs need VPC Endpoint ID, not VPC ID       │
│                                                              │
│  2. Is the API deployed after changes?                       │
│     └── Resource policy changes need redeployment           │
│                                                              │
│  3. Is VPC Endpoint private DNS enabled?                     │
│     └── DNS must resolve to private IPs                      │
│                                                              │
│  4. Is API Key required but missing Usage Plan?              │
│     └── API Keys need Usage Plan association                 │
│                                                              │
│  5. Is IAM auth enabled but request unsigned?                │
│     └── IAM auth requires SigV4 signing                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 500 Internal Server Errors

```
┌─────────────────────────────────────────────────────────────┐
│                   500 TROUBLESHOOTING                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Does Lambda have correct response format?                │
│     └── Must return: statusCode, headers, body (as string)  │
│                                                              │
│  2. Is Lambda timing out?                                    │
│     └── Check CloudWatch logs for timeout                    │
│                                                              │
│  3. Does API Gateway have permission to invoke Lambda?       │
│     └── Check Lambda resource-based policy                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Validation Checklist

```
□ VPC Endpoint
  ├── □ DNS hostnames enabled on VPC
  ├── □ DNS resolution enabled on VPC
  ├── □ Private DNS enabled on endpoint
  └── □ Subnets have available IPs

□ API Structure
  ├── □ No duplicate /stage resource (e.g., no /dev if stage is dev)
  └── □ Resources under root: /api1, /api2

□ Resource Policy
  ├── □ Using aws:SourceVpce (not aws:SourceVpc)
  ├── □ Correct VPC Endpoint ID
  └── □ Correct ARN format

□ API Keys (if used)
  ├── □ Usage Plan exists
  ├── □ API + stage attached to Usage Plan
  └── □ API Key associated with Usage Plan

□ CORS (if browser clients)
  ├── □ CORS enabled on {proxy+} resources
  └── □ Gateway Responses configured for 4XX/5XX

□ Deployment
  └── □ API redeployed after ALL changes
```

---

**← Back to:** [05-hands-on-implementation.md](05-hands-on-implementation.md) | [00-overview.md](00-overview.md)
