# Explicit Path-Based Routing (Non-Proxy Approach)

This document explains how to implement **explicit path-based routing** where each endpoint is defined separately in API Gateway, as opposed to the `{proxy+}` approach.

---

## Proxy vs Explicit: Quick Comparison

| Aspect | Proxy (`{proxy+}`) | Explicit Paths |
|--------|-------------------|----------------|
| **API Gateway config** | One resource catches all | Each endpoint defined |
| **Routing logic** | Lambda handles routing | API Gateway routes |
| **Per-path auth** | ❌ Not possible | ✅ Yes |
| **Per-path throttling** | ❌ Not possible | ✅ Yes |
| **Request validation** | ❌ Not possible | ✅ Yes |
| **Setup effort** | Low | Higher |

---

## When to Use Explicit Paths

Use explicit paths when you need:
- Different authorization per endpoint
- Per-endpoint rate limiting
- Request/response validation
- Caching on specific endpoints
- Clear API documentation (OpenAPI)

---

## Example: Multi-Endpoint Lambda

Let's create a Lambda that handles specific endpoints explicitly.

### Lambda Code: `users-service-lambda`

```python
import json

def lambda_handler(event, context):
    """
    Handles explicit paths:
    - GET  /users          → list all users
    - POST /users          → create user
    - GET  /users/{id}     → get user by ID
    - PUT  /users/{id}     → update user
    - DELETE /users/{id}   → delete user
    """
    
    http_method = event.get('httpMethod', 'GET')
    resource = event.get('resource', '')
    path_params = event.get('pathParameters', {}) or {}
    body = event.get('body')
    
    # Parse body if present
    if body:
        try:
            body = json.loads(body)
        except:
            body = body
    
    user_id = path_params.get('id')
    
    # Route based on resource and method
    if resource == '/users' and http_method == 'GET':
        return respond(200, {
            "action": "LIST_USERS",
            "users": [
                {"id": "1", "name": "Alice"},
                {"id": "2", "name": "Bob"}
            ]
        })
    
    elif resource == '/users' and http_method == 'POST':
        return respond(201, {
            "action": "CREATE_USER",
            "message": "User created",
            "data": body
        })
    
    elif resource == '/users/{id}' and http_method == 'GET':
        return respond(200, {
            "action": "GET_USER",
            "user": {"id": user_id, "name": f"User {user_id}"}
        })
    
    elif resource == '/users/{id}' and http_method == 'PUT':
        return respond(200, {
            "action": "UPDATE_USER",
            "message": f"User {user_id} updated",
            "data": body
        })
    
    elif resource == '/users/{id}' and http_method == 'DELETE':
        return respond(200, {
            "action": "DELETE_USER",
            "message": f"User {user_id} deleted"
        })
    
    else:
        return respond(404, {
            "error": "Not Found",
            "resource": resource,
            "method": http_method
        })

def respond(status_code, body):
    return {
        "statusCode": status_code,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps(body, indent=2)
    }
```

---

## API Gateway Configuration

### Resource Structure

```
/
└── /users
    ├── GET      → users-service-lambda (List users)
    ├── POST     → users-service-lambda (Create user)
    └── /{id}
        ├── GET      → users-service-lambda (Get user)
        ├── PUT      → users-service-lambda (Update user)
        └── DELETE   → users-service-lambda (Delete user)
```

### Step-by-Step Setup

#### 1. Create `/users` Resource
1. Select root `/`
2. Create resource → Name: `users`

#### 2. Create GET Method on `/users`
1. Select `/users`
2. Actions → Create Method → GET
3. Integration type: Lambda Function
4. ✅ Lambda Proxy Integration
5. Lambda: `users-service-lambda`

#### 3. Create POST Method on `/users`
1. Select `/users`
2. Actions → Create Method → POST
3. Same Lambda configuration

#### 4. Create `/{id}` Resource under `/users`
1. Select `/users`
2. Create resource → Name: `{id}`
3. Resource path: `{id}` (curly braces = path parameter)

#### 5. Create GET, PUT, DELETE Methods on `/users/{id}`
1. Select `/users/{id}`
2. Create each method (GET, PUT, DELETE)
3. All point to same Lambda

---

## Comparison: Number of API Gateway Resources

### Proxy Approach
```
/users/{proxy+}     →  1 resource, 1 method (ANY)
```
- **Total:** 1 resource, 1 method

### Explicit Approach
```
/users              →  1 resource, 2 methods (GET, POST)
/users/{id}         →  1 resource, 3 methods (GET, PUT, DELETE)
```
- **Total:** 2 resources, 5 methods

---

## Benefits of Explicit Approach

### 1. Per-Endpoint Authorization

```
/users          GET   → Authorization: NONE (public list)
/users          POST  → Authorization: AWS_IAM (admin only)
/users/{id}     GET   → Authorization: NONE
/users/{id}     PUT   → Authorization: AWS_IAM
/users/{id}     DELETE→ Authorization: AWS_IAM
```

### 2. Per-Endpoint Throttling

In Usage Plan → Configure Method Throttling:
```
/users          GET   → 1000 req/s (high traffic)
/users          POST  → 10 req/s (rate limit creates)
/users/{id}     DELETE→ 5 req/s (very limited)
```

### 3. Request Validation

```json
// Model for POST /users
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "required": ["name", "email"],
  "properties": {
    "name": {"type": "string", "minLength": 1},
    "email": {"type": "string", "format": "email"}
  }
}
```

---

## Testing Explicit Endpoints

```bash
# List users
curl -X GET https://API_URL/dev/users

# Create user
curl -X POST https://API_URL/dev/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Charlie", "email": "charlie@example.com"}'

# Get specific user
curl -X GET https://API_URL/dev/users/123

# Update user
curl -X PUT https://API_URL/dev/users/123 \
  -H "Content-Type: application/json" \
  -d '{"name": "Charlie Updated"}'

# Delete user
curl -X DELETE https://API_URL/dev/users/123
```

---

## Hybrid Approach: Combining Both

You can use both approaches in the same API:

```
/
├── /users                    ← Explicit (fine control)
│   ├── GET, POST
│   └── /{id}
│       ├── GET, PUT, DELETE
│
├── /admin/{proxy+}           ← Proxy (catch-all for admin)
│   └── ANY → admin-lambda
│
└── /public/{proxy+}          ← Proxy (catch-all for public)
    └── ANY → public-lambda
```

---

## Decision Guide: When to Use Which

| Scenario | Recommendation |
|----------|----------------|
| POC / rapid development | `{proxy+}` |
| Stable, well-defined API | Explicit paths |
| Different auth per endpoint | Explicit paths |
| Migrating from ALB | `{proxy+}` initially |
| Microservices with single Lambda | `{proxy+}` per service |
| Public API with documentation | Explicit paths |

---

**← Back to:** [09-team-decision-guide.md](09-team-decision-guide.md) | [00-overview.md](00-overview.md)
