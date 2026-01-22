# Dev-Ready Operational Features

This section implements and explains operational features for a production-ready internal API.

---

## 1. Logging and Monitoring (CloudWatch)

### Enable API Gateway Logging

#### Step 1: Create CloudWatch Log Role

1. Go to IAM Console → Roles → Create role
2. Select **API Gateway** as trusted entity
3. Attach policy: `AmazonAPIGatewayPushToCloudWatchLogs`
4. Name: `api-gateway-cloudwatch-role`

#### Step 2: Configure API Gateway Settings

1. API Gateway Console → Settings (left sidebar)
2. Enter the CloudWatch log role ARN
3. Save

#### Step 3: Enable Stage Logging

1. Go to your API → Stages → `dev`
2. Logs/Tracing tab:

| Setting | Value |
|---------|-------|
| CloudWatch Logs | ✅ Enable |
| Log Level | INFO or ERROR |
| Log full requests/responses | ✅ (dev only, disable in prod) |
| Enable X-Ray Tracing | ✅ (optional) |

### Log Locations

| Service | Log Group Pattern |
|---------|------------------|
| API Gateway | `API-Gateway-Execution-Logs_<api-id>/<stage>` |
| Lambda | `/aws/lambda/<function-name>` |

### Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `Count` | Total API calls | Baseline + 50% |
| `4XXError` | Client errors | > 5% of requests |
| `5XXError` | Server errors | > 1% of requests |
| `Latency` | End-to-end time | > 1000ms (p95) |
| `IntegrationLatency` | Backend time | > 500ms (p95) |

### Sample CloudWatch Dashboard Widgets

```
┌─────────────────────────────────────────────────────────────┐
│                    API GATEWAY DASHBOARD                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐    ┌──────────────────┐              │
│  │  Request Count   │    │   Error Rates    │              │
│  │   (per minute)   │    │   (4XX / 5XX)    │              │
│  │    ▄▅▆▇█▇▆▅▄     │    │    ▁▁▂▁▁▁▁▁▁    │              │
│  └──────────────────┘    └──────────────────┘              │
│                                                              │
│  ┌──────────────────┐    ┌──────────────────┐              │
│  │   Latency p50    │    │   Latency p99    │              │
│  │     120ms        │    │     450ms        │              │
│  └──────────────────┘    └──────────────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Request Validation

### Types of Validation

| Type | What It Validates | Benefit |
|------|------------------|---------|
| **Query Parameters** | Required params, types | Reject bad requests early |
| **Headers** | Required headers | Consistent API usage |
| **Request Body** | Schema validation | Data integrity |

### Implementation Steps

#### Create Request Validator

1. API Gateway Console → Your API → Models
2. Create model for expected request body:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
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
  },
  "required": ["name", "email"]
}
```

3. Go to Resources → Select method → Method Request
4. Set **Request Validator**: `Validate body, query string parameters, and headers`
5. Add model under **Request Body**
6. **Deploy the API**

### Validation Error Responses

```json
{
    "message": "Invalid request body"
}
```

---

## 3. Throttling and Rate Limits

### Throttling Levels

```
┌─────────────────────────────────────────────────────────────┐
│                   THROTTLING HIERARCHY                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Level 1: Account Level                                      │
│  ├── Default: 10,000 req/sec across all APIs                │
│  └── Can request increase                                   │
│                                                              │
│  Level 2: Stage Level                                        │
│  ├── Configure in Stage settings                            │
│  └── Applies to all methods in stage                        │
│                                                              │
│  Level 3: Method Level                                       │
│  ├── Override stage settings per method                     │
│  └── Set in Usage Plan → API Stages → Configure Method      │
│                                                              │
│  Level 4: API Key Level                                      │
│  ├── Per-client throttling via Usage Plans                  │
│  └── Most granular control                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Configure Stage Throttling

1. Stages → `dev` → Stage Editor → Default Method Throttling
2. Set:
   - Rate: 1000 requests/second
   - Burst: 2000 requests

---

## 4. Stage-Based Deployment

### Stage Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                     DEPLOYMENT STAGES                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Stage: dev                                                  │
│  ├── URL: .../dev/api1/...                                  │
│  ├── Purpose: Development and testing                       │
│  ├── Logging: Full request/response                         │
│  └── Throttling: Lower limits                               │
│                                                              │
│  Stage: staging                                              │
│  ├── URL: .../staging/api1/...                              │
│  ├── Purpose: Pre-production testing                        │
│  ├── Logging: Errors only                                   │
│  └── Throttling: Production-like limits                     │
│                                                              │
│  Stage: prod                                                 │
│  ├── URL: .../prod/api1/...                                 │
│  ├── Purpose: Production traffic                            │
│  ├── Logging: Errors only (no body logging)                 │
│  └── Throttling: Full capacity                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Stage Variables

Use stage variables for environment-specific configuration:

1. Stages → `dev` → Stage Variables
2. Add variables:

| Variable | Dev Value | Staging Value | Prod Value |
|----------|-----------|---------------|------------|
| `lambdaAlias` | `$LATEST` | `staging` | `prod` |
| `logLevel` | `DEBUG` | `INFO` | `ERROR` |

3. Reference in integration: `${stageVariables.lambdaAlias}`

---

## 5. Naming and Structure Best Practices

### API Naming

| Element | Convention | Example |
|---------|------------|---------|
| API Name | `<project>-<purpose>-api` | `payments-internal-api` |
| Resource | Lowercase, plural nouns | `/users`, `/orders` |
| Stage | Lowercase environment | `dev`, `staging`, `prod` |

### Resource Structure

```
RECOMMENDED                          AVOID
────────────                         ─────

/users                               /getUsers
  ├── GET (list)                     /createUser
  ├── POST (create)                  /getUserById
  └── /{userId}                      /user/get
      ├── GET (get one)              /user/list
      ├── PUT (update)
      └── DELETE
```

### Tagging Strategy

Apply tags for cost tracking and organization:

| Tag Key | Example Value |
|---------|---------------|
| `Project` | `api-gateway-poc` |
| `Environment` | `dev` |
| `Owner` | `platform-team` |
| `CostCenter` | `engineering` |

---

## 6. Safe Defaults for Internal APIs

### Security Defaults

```
┌─────────────────────────────────────────────────────────┐
│           INTERNAL API SAFE DEFAULTS                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ✅ Private API (no public access)                       │
│  ✅ VPC Endpoint restriction in resource policy          │
│  ✅ IAM authorization enabled                            │
│  ✅ CloudWatch logging enabled                           │
│  ✅ Throttling configured                                │
│                                                          │
│  Optional based on needs:                                │
│  △ API Keys (for client identification)                 │
│  △ Request validation (if schema known)                 │
│  △ Caching (for read-heavy APIs)                        │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Operational Defaults

| Setting | Recommended Default |
|---------|---------------------|
| Logging | Enabled (INFO level) |
| Detailed Metrics | Enabled |
| X-Ray Tracing | Enabled |
| Throttling | 100-1000 req/sec based on expected load |
| Timeout | 29 seconds (max for REST) |

---

## Operational Checklist

```
□ Logging & Monitoring
  ├── □ CloudWatch Logs enabled
  ├── □ Log role configured
  ├── □ Appropriate log level set
  └── □ Alarms configured for errors

□ Throttling
  ├── □ Stage-level throttling set
  ├── □ Usage plans for rate limiting (if needed)
  └── □ Account limits reviewed

□ Deployment
  ├── □ Stage created (dev)
  ├── □ Stage variables configured (if needed)
  └── □ Deployment history tracked

□ Best Practices
  ├── □ Consistent naming convention
  ├── □ Resources tagged
  ├── □ Documentation updated
  └── □ Team access configured
```

---

## Final Outcome Verification

After completing all sections, verify:

| Requirement | Verification |
|-------------|--------------|
| Understand Private REST APIs | ✅ Completed sections 1-3 |
| Replace ALB path routing | ✅ Completed section 4-5 |
| Use `{proxy+}` correctly | ✅ Tested in hands-on |
| Secure internal APIs | ✅ Completed section 6 |
| Operate APIs properly | ✅ Completed section 7 |

---

**← Previous:** [06-security.md](06-security.md) | **Back to:** [00-overview.md](00-overview.md)
