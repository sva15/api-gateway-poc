# Method Configuration & Integration Types Deep Dive

This document explains method configuration options in API Gateway, covering integration types, proxy vs non-proxy integration, and when to use each.

---

## Method Configuration Options

When you create or edit a method in API Gateway, you see these configuration options:

### Method Details

| Option | Description | Values |
|--------|-------------|--------|
| **Method type** | HTTP method to handle | GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS, **ANY** |

---

## Integration Types Explained

### 1. Lambda Proxy (Recommended for Most Cases)

```
Client → API Gateway → Lambda (receives full event)
                    ← Lambda returns formatted response
```

**What it does:**
- Passes the **entire request** to Lambda as an event object
- Lambda returns a **formatted response** with statusCode, headers, body
- No mapping templates needed

**Event passed to Lambda:**
```json
{
  "httpMethod": "GET",
  "path": "/users/123",
  "headers": {...},
  "queryStringParameters": {...},
  "pathParameters": {"id": "123"},
  "body": "{...}",
  "requestContext": {...}
}
```

**Lambda response format:**
```json
{
  "statusCode": 200,
  "headers": {"Content-Type": "application/json"},
  "body": "{\"message\": \"success\"}"
}
```

---

### 2. Lambda (Non-Proxy) Integration

```
Client → API Gateway → Mapping Template → Lambda
                    ← Mapping Template ← Lambda
```

**What it does:**
- You define **mapping templates** to transform request/response
- Lambda receives a **custom event** (only what you map)
- Lambda returns raw data, API Gateway transforms it

**Use case: Transform parameters before Lambda**

**Example: Map query parameters to Lambda event**

Integration Request Mapping Template:
```velocity
{
  "userId": "$input.params('user_id')",
  "action": "$input.params('action')",
  "timestamp": "$context.requestTimeEpoch"
}
```

Lambda receives:
```json
{
  "userId": "123",
  "action": "view",
  "timestamp": 1706000000000
}
```

**Example: Transform Lambda response**

Integration Response Mapping Template:
```velocity
#set($data = $input.path('$'))
{
  "result": {
    "status": "success",
    "user": $data.user,
    "processedAt": "$context.requestTime"
  }
}
```

---

### 3. HTTP Proxy

```
Client → API Gateway → Backend HTTP Endpoint
```

**What it does:**
- Directly proxies to an HTTP endpoint
- Full request forwarded, full response returned
- No Lambda involved

**Use case:** Proxying to existing REST services, microservices

---

### 4. HTTP (Non-Proxy)

```
Client → API Gateway → Mapping → Backend HTTP
                    ← Mapping ← Backend HTTP
```

**What it does:**
- Transform requests/responses when calling HTTP backends
- Map headers, query params, paths

**Use case:** Legacy APIs with different parameter formats

---

### 5. Mock Integration

```
Client → API Gateway → Returns configured response (no backend)
```

**What it does:**
- Returns a hardcoded response
- No backend invocation

**Use case:** API mocking, OPTIONS methods for CORS, health checks

---

### 6. AWS Service Integration

```
Client → API Gateway → AWS Service (S3, SQS, DynamoDB, etc.)
```

**What it does:**
- Directly call AWS services without Lambda
- Map request to AWS API call

**Use case:** Direct S3 uploads, SQS messages, DynamoDB operations

---

### 7. VPC Link

```
Client → API Gateway → VPC Link → Private resource (NLB, ALB)
```

**What it does:**
- Connect to resources in VPC that aren't publicly accessible
- Works with Network Load Balancer or ALB

**Use case:** Private ECS services, EC2 backends

---

## Response Transfer Mode

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Buffered** | Wait for complete response before sending | Default, most APIs |
| **Stream** | Send response chunks as received | Large responses, real-time |

---

## Method Request Settings

### Authorization Options

| Value | Description | When to Use |
|-------|-------------|-------------|
| **None** | No authorization | Public endpoints, VPC-only access |
| **AWS_IAM** | SigV4 signed requests | Service-to-service |
| **COGNITO_USER_POOLS** | Cognito JWT tokens | User authentication |
| **CUSTOM** | Lambda Authorizer | Custom auth logic |

### Request Validator

| Option | What it validates |
|--------|-------------------|
| None | No validation |
| Validate body | Request body against model schema |
| Validate parameters | Required query/header parameters |
| Validate body, parameters | Both body and parameters |

### API Key Required

| Value | Behavior |
|-------|----------|
| **false** | API key not needed |
| **true** | Requires `x-api-key` header (+ Usage Plan) |

---

## Request Parameters

### Request Paths (Path Parameters)

For resource `/users/{id}`:

| Name | Caching | Mapped To |
|------|---------|-----------|
| id | ✅ Optional | `method.request.path.id` |

### URL Query String Parameters

Define expected query parameters:

| Name | Required | Caching |
|------|----------|---------|
| status | ❌ | ✅ |
| limit | ❌ | ✅ |

### HTTP Request Headers

Define expected headers:

| Name | Required |
|------|----------|
| Authorization | ✅ |
| X-Correlation-Id | ❌ |

### Request Body (Models)

Define JSON schema for body validation:

| Content Type | Model |
|--------------|-------|
| application/json | CreateUserModel |

---

## Proxy Integration vs Non-Proxy: Decision Guide

### Use Lambda Proxy Integration When:

✅ Starting a new project
✅ Lambda handles routing internally
✅ Using frameworks (Flask, Express, FastAPI)
✅ Want full request context in Lambda
✅ Quick setup needed

### Use Non-Proxy Integration When:

✅ Need to transform request before Lambda
✅ Legacy Lambda with specific input format
✅ Want API Gateway to handle response formatting
✅ Hide Lambda implementation details
✅ Need request/response mapping

---

## Example: Non-Proxy Integration Mapping

### Scenario
UI sends: `?user_id=123&action=view`
Lambda expects: `{"userId": "123", "action": "view"}`

### Integration Request Mapping Template

```velocity
#set($inputRoot = $input.path('$'))
{
  "userId": "$input.params('user_id')",
  "action": "$input.params('action')",
  "headers": {
    "contentType": "$input.params().header.get('Content-Type')",
    "userAgent": "$input.params().header.get('User-Agent')"
  },
  "requestId": "$context.requestId"
}
```

### Integration Response Mapping Template

Lambda returns:
```json
{"status": "ok", "data": {"name": "John"}}
```

Transform to:
```velocity
#set($data = $input.path('$'))
{
  "success": true,
  "result": {
    "status": "$data.status",
    "user": $data.data
  },
  "meta": {
    "apiVersion": "1.0",
    "requestId": "$context.requestId"
  }
}
```

---

## ANY Method vs Individual Methods

### Question: Should `{proxy+}` use ANY or individual methods?

### ANY Method (Recommended for Proxy)

```
/{proxy+}
 └── ANY  →  Lambda
```

**Pros:**
- Single configuration
- Lambda receives `httpMethod` in event
- Simpler API Gateway setup

**Cons:**
- Can't have different auth per method
- Can't have different throttling per method

### Individual Methods on `{proxy+}`

```
/{proxy+}
 ├── GET     →  Lambda
 ├── POST    →  Lambda
 ├── PUT     →  Lambda
 └── DELETE  →  Lambda
```

**Pros:**
- Per-method authorization possible
- Per-method throttling possible

**Cons:**
- All point to same Lambda anyway
- More configuration
- Redundant if Lambda handles everything

### Recommendation

| Scenario | Recommendation |
|----------|----------------|
| All methods same auth/throttling | Use **ANY** |
| Different auth per method | Use **individual methods** |
| POC / internal API | Use **ANY** |
| Production with strict controls | Consider **individual methods** |

---

## Summary: Integration Type Selection

```
┌─────────────────────────────────────────────────────────────┐
│                   INTEGRATION TYPE GUIDE                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Q: Is the backend a Lambda function?                        │
│  ├── YES → Q: Need request/response transformation?          │
│  │         ├── NO  → Lambda Proxy Integration ✅              │
│  │         └── YES → Lambda (Non-Proxy) Integration          │
│  │                                                           │
│  └── NO  → Q: Is the backend an HTTP endpoint?               │
│            ├── YES → Q: Need transformation?                 │
│            │         ├── NO  → HTTP Proxy                    │
│            │         └── YES → HTTP (Non-Proxy)              │
│            │                                                 │
│            └── NO  → Q: Is it an AWS service?                │
│                      ├── YES → AWS Service Integration       │
│                      └── NO  → Q: Is it in private VPC?      │
│                               ├── YES → VPC Link             │
│                               └── NO  → Mock (for testing)   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

**← Back to:** [05-hands-on-implementation.md](05-hands-on-implementation.md) | [00-overview.md](00-overview.md)
