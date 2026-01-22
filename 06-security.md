# Security & API Protection

This section extends the POC with security features.

---

## Security Layers Overview

```
Layer 1: NETWORK
├── VPC Endpoint + Resource Policy
└── Controls WHO can reach the API (network level)

Layer 2: AUTHENTICATION
├── IAM Auth, Lambda Authorizer, or Cognito
└── Verifies IDENTITY of the caller

Layer 3: RATE LIMITING
├── API Keys + Usage Plans
└── Controls RATE and USAGE of the API

Layer 4: VALIDATION
├── Request schema validation
└── Ensures request FORMAT is correct
```

---

## 1. API Keys & Usage Plans

### What API Keys Protect vs Don't Protect

| API Keys PROVIDE | API Keys DO NOT PROVIDE |
|------------------|------------------------|
| ✅ Identification (which client) | ❌ Authentication (verify identity) |
| ✅ Rate Limiting (req/sec) | ❌ Authorization (check permissions) |
| ✅ Quota Limits (req/month) | ❌ Security (keys can be stolen) |
| ✅ Usage Tracking | ❌ Fine-grained access control |

> **Important:** API Keys are for **identification and rate limiting**, not security.

### Implementation Steps

#### Create Usage Plan

1. API Gateway Console → Usage Plans → Create
2. Configure:
   - Name: `internal-api-usage-plan`
   - Throttling Rate: 100 req/sec
   - Throttling Burst: 200 requests
   - Quota: 10000 requests/month
3. Add API Stage: Select your API + `dev` stage

#### Create API Key

1. API Gateway Console → API Keys → Create
2. Name: `client-1-api-key`
3. Auto generate: Yes
4. Copy the key value

#### Associate Key with Usage Plan

1. Usage Plans → Select plan → Associated API keys
2. Add `client-1-api-key`

#### Enable on Methods

1. Resources → Select method → Method Request
2. Set **API Key Required**: `true`
3. **Deploy the API**

#### Test

```bash
# Without API key - 403 Forbidden
curl https://<api-id>.execute-api.<region>.amazonaws.com/dev/api1/test

# With API key - Success
curl -H "x-api-key: YOUR_API_KEY" \
  https://<api-id>.execute-api.<region>.amazonaws.com/dev/api1/test
```

---

## 2. IAM Authorization

### When to Use IAM Auth

| Use Case | IAM | API Keys |
|----------|-----|----------|
| Service-to-service calls | ✅ Best | ❌ Not secure |
| Rate limiting | ❌ Not built-in | ✅ Best |
| Identity-based access | ✅ Yes | ❌ No |
| Cross-account access | ✅ Yes | ✅ Yes |

### Implementation Steps

#### Enable IAM Auth on Method

1. Resources → Select method → Method Request
2. Set **Authorization**: `AWS_IAM`
3. **Deploy the API**

#### Create IAM Policy for Caller

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "execute-api:Invoke",
            "Resource": [
                "arn:aws:execute-api:<region>:<account>:<api-id>/dev/*/api1/*",
                "arn:aws:execute-api:<region>:<account>:<api-id>/dev/*/api2/*"
            ]
        }
    ]
}
```

#### Attach Policy

Attach to Lambda execution role, EC2 instance role, or IAM user.

---

## 3. Resource Policies

### Purpose

Control **who can access the API at the network/principal level**.

### Example: Allow Only Specific VPC Endpoint

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:<region>:<account>:<api-id>/*",
            "Condition": {
                "StringNotEquals": {
                    "aws:sourceVpce": "vpce-0123456789abcdef0"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:<region>:<account>:<api-id>/*"
        }
    ]
}
```

### Example: Multiple VPC Endpoints

```json
{
    "Condition": {
        "StringNotEquals": {
            "aws:sourceVpce": [
                "vpce-0123456789abcdef0",
                "vpce-abcdef0123456789a"
            ]
        }
    }
}
```

---

## 4. Lambda Authorizers

### When Required

| Use Lambda Authorizer When | Avoid When |
|---------------------------|------------|
| Custom authentication logic | Simple IAM auth sufficient |
| External identity providers | Standard Cognito OAuth works |
| Complex authorization rules | Network restrictions enough |
| Custom token formats | Simple rate limiting needed |

### For This POC

> **Recommendation:** Lambda Authorizers are likely **not needed** for internal private APIs. Start with:
> 1. Resource Policy (VPC Endpoint restriction)
> 2. IAM Authorization (identity verification)
> 3. API Keys (if rate limiting needed)

---

## Recommended Security Stack

```
INTERNAL PRIVATE API SECURITY

┌─────────────────────────────────────┐
│ Layer 1: NETWORK (Always)           │
│ • Private API + VPC Endpoint        │
│ • Resource Policy restricts vpce-id │
├─────────────────────────────────────┤
│ Layer 2: AUTHENTICATION (Choose)    │
│ • IAM Auth (recommended)            │
│ • OR Lambda Authorizer (custom)     │
│ • OR None (if VPC restriction ok)   │
├─────────────────────────────────────┤
│ Layer 3: RATE LIMITING (Optional)   │
│ • API Keys + Usage Plans            │
└─────────────────────────────────────┘
```

---

**← Previous:** [05-hands-on-implementation.md](05-hands-on-implementation.md) | **Next:** [07-operations.md](07-operations.md) →
