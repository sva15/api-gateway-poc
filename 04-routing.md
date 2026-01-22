# Path-Based Routing: ALB vs API Gateway

This is a **critical section** that explains the fundamental differences between ALB and API Gateway routing patterns.

---

## 1. How ALB Wildcard Routing Works

### ALB Path-Based Routing Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    APPLICATION LOAD BALANCER                     │
│                                                                  │
│  Listener Rules (evaluated in priority order):                   │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Rule 1: Path = /dev/api1/*                                 │ │
│  │  Action: Forward to → Target Group 1 (Lambda 1)             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Rule 2: Path = /dev/api2/*                                 │ │
│  │  Action: Forward to → Target Group 2 (Lambda 2)             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Default Rule: Any path                                     │ │
│  │  Action: Return 404 or fixed response                       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### ALB Wildcard Behavior

| Path Pattern | Matches | Does NOT Match |
|--------------|---------|----------------|
| `/dev/api1/*` | `/dev/api1/foo` | `/dev/api1` (no trailing path) |
| | `/dev/api1/foo/bar` | `/dev/api10/foo` |
| | `/dev/api1/a/b/c/d` | `/prod/api1/foo` |

### Key Characteristics of ALB Routing

```
ALB Routing Characteristics
─────────────────────────────

1. PATTERN-BASED
   └── Single rule matches entire path pattern
   └── Wildcard (*) handles any subpath depth

2. SIMPLE CONFIGURATION
   └── One rule per base path
   └── No resource tree needed

3. FULL PATH FORWARDED
   └── Lambda receives complete path
   └── /dev/api1/users/123 → path = "/dev/api1/users/123"

4. HTTP METHOD AGNOSTIC (by default)
   └── Same rule handles GET, POST, PUT, DELETE
   └── Can add conditions for methods if needed
```

### ALB Event in Lambda

When ALB forwards to Lambda, the event looks like:

```json
{
  "requestContext": {
    "elb": {
      "targetGroupArn": "arn:aws:elasticloadbalancing:..."
    }
  },
  "httpMethod": "GET",
  "path": "/dev/api1/users/123",
  "queryStringParameters": {},
  "headers": {
    "host": "internal-alb.example.com"
  },
  "body": null,
  "isBase64Encoded": false
}
```

---

## 2. How API Gateway Routing Works

### API Gateway Resource-Based Model

```
┌─────────────────────────────────────────────────────────────────┐
│                       API GATEWAY                                │
│                                                                  │
│  Resource Tree (hierarchical structure):                         │
│                                                                  │
│  /                          (root)                               │
│  └── dev                    (resource)                           │
│      ├── api1               (resource)                           │
│      │   ├── GET            (method → integration)               │
│      │   ├── POST           (method → integration)               │
│      │   └── {proxy+}       (proxy resource)                     │
│      │       ├── GET        (method → Lambda 1)                  │
│      │       ├── POST       (method → Lambda 1)                  │
│      │       └── ANY        (method → Lambda 1)                  │
│      │                                                           │
│      └── api2               (resource)                           │
│          └── {proxy+}       (proxy resource)                     │
│              └── ANY        (method → Lambda 2)                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Concepts

#### Resources vs Methods

```
┌──────────────────────────────────────────────────────────────┐
│                  RESOURCE vs METHOD                           │
│                                                               │
│  RESOURCE = URL path segment                                  │
│  ├── Examples: /users, /orders, /api1                         │
│  └── Defines WHAT you're accessing                            │
│                                                               │
│  METHOD = HTTP verb on a resource                             │
│  ├── Examples: GET, POST, PUT, DELETE, ANY                    │
│  └── Defines HOW you're accessing it                          │
│                                                               │
│  ┌────────────────┐                                           │
│  │   /users       │ ← Resource                                │
│  │   ├── GET      │ ← Method (list users)                     │
│  │   ├── POST     │ ← Method (create user)                    │
│  │   └── {id}     │ ← Nested resource (path param)            │
│  │       ├── GET  │ ← Method (get specific user)              │
│  │       ├── PUT  │ ← Method (update user)                    │
│  │       └── DELETE│ ← Method (delete user)                   │
│  └────────────────┘                                           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Explicit Resources vs Proxy Resources

### Explicit Resources (`/dev/api1/foo`)

Each path segment is defined explicitly:

```
Explicit Resource Structure
────────────────────────────

/dev
└── /api1
    └── /foo
        ├── GET  → Lambda (handles GET /dev/api1/foo)
        └── POST → Lambda (handles POST /dev/api1/foo)
```

**Characteristics:**

| Aspect | Behavior |
|--------|----------|
| **Matching** | Exact path match only |
| **Flexibility** | Must define each path explicitly |
| **Methods** | Must define each HTTP method |
| **Use Case** | When you know all paths upfront |

**Example:**
- ✅ `/dev/api1/foo` matches
- ❌ `/dev/api1/foo/bar` does NOT match (no resource defined)

---

### Proxy Resources (`/dev/api1/{proxy+}`)

The `{proxy+}` is a **greedy path parameter** that captures all remaining path segments.

```
Proxy Resource Structure
─────────────────────────

/dev
└── /api1
    └── /{proxy+}           ← Captures everything after /api1/
        └── ANY → Lambda    ← Single method handles all HTTP verbs
```

**Characteristics:**

| Aspect | Behavior |
|--------|----------|
| **Matching** | Any path after the parent |
| **Flexibility** | Single resource handles all subpaths |
| **Path Variable** | `{proxy+}` value available in Lambda |
| **Use Case** | When backend handles routing |

**Examples:**

| Request Path | `proxy` Value in Lambda |
|--------------|-------------------------|
| `/dev/api1/foo` | `foo` |
| `/dev/api1/users/123` | `users/123` |
| `/dev/api1/a/b/c/d` | `a/b/c/d` |

---

### Side-by-Side Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│              EXPLICIT vs PROXY RESOURCES                         │
├────────────────────────────┬────────────────────────────────────┤
│      EXPLICIT RESOURCES    │      PROXY RESOURCES               │
├────────────────────────────┼────────────────────────────────────┤
│                            │                                     │
│  /users                    │  /{proxy+}                         │
│  ├── GET                   │  └── ANY                           │
│  ├── POST                  │                                     │
│  └── {id}                  │  Captures: users, users/123,       │
│      ├── GET               │            orders/456/items        │
│      ├── PUT               │                                     │
│      └── DELETE            │                                     │
│                            │                                     │
│  Each path = 1 resource    │  One resource = all paths          │
│  Each verb = 1 method      │  ANY = all verbs                   │
│                            │                                     │
│  API Gateway handles       │  Backend (Lambda) handles          │
│  routing decisions         │  routing decisions                 │
│                            │                                     │
│  Fine-grained control at   │  Flexibility at backend level      │
│  gateway level             │                                     │
│                            │                                     │
└────────────────────────────┴────────────────────────────────────┘
```

---

## 4. Key Questions Answered

### Can API Gateway Replicate ALB-Style Base-Path Routing?

**Answer: YES, using `{proxy+}` resources.**

```
ALB Style:         API Gateway Equivalent:
────────────       ──────────────────────

/dev/api1/*   →    /dev/api1/{proxy+} + ANY method
/dev/api2/*   →    /dev/api2/{proxy+} + ANY method
```

**How it maps:**

| ALB Rule | API Gateway Structure |
|----------|----------------------|
| `/dev/api1/*` → Lambda 1 | `/dev/api1/{proxy+}` → Lambda 1 |
| `/dev/api2/*` → Lambda 2 | `/dev/api2/{proxy+}` → Lambda 2 |

**Important Difference:**
- ALB `/*` matches paths WITH and WITHOUT trailing content
- API Gateway `{proxy+}` requires at least one path segment after

```
Path               ALB /dev/api1/*    API Gateway /dev/api1/{proxy+}
────────────────   ───────────────    ─────────────────────────────
/dev/api1/foo      ✅ Match           ✅ Match (proxy = "foo")
/dev/api1          ✅ Match           ❌ No match!
```

**Solution:** If you need `/dev/api1` (without trailing path) to work, add a method directly on `/dev/api1` resource.

---

### When Should `{proxy+}` Be Used?

```
┌─────────────────────────────────────────────────────────────────┐
│                  USE {proxy+} WHEN:                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ✅ Migrating from ALB with wildcard routing                     │
│     └── Minimal API Gateway configuration needed                 │
│                                                                  │
│  ✅ Backend framework handles its own routing                    │
│     └── Express.js, Flask, FastAPI in Lambda                     │
│     └── Backend knows all valid routes                           │
│                                                                  │
│  ✅ Unknown or dynamic paths                                     │
│     └── CMS-style systems                                        │
│     └── File storage APIs (/files/{path+})                       │
│                                                                  │
│  ✅ Rapid prototyping / POC                                      │
│     └── Get working quickly, refine later                        │
│                                                                  │
│  ✅ Backend provides its own validation                          │
│     └── Schema validation in application code                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  AVOID {proxy+} WHEN:                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ❌ Need per-path authorization                                  │
│     └── Different auth for /admin/* vs /public/*                 │
│                                                                  │
│  ❌ Want request validation at gateway level                     │
│     └── API Gateway schema validation per endpoint               │
│                                                                  │
│  ❌ Need per-path throttling                                     │
│     └── Different rate limits per operation                      │
│                                                                  │
│  ❌ Want API Gateway caching                                     │
│     └── Caching works best with explicit resources               │
│                                                                  │
│  ❌ OpenAPI/Swagger documentation                                │
│     └── Explicit resources generate better docs                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### Trade-offs: Fine-Grained Routes vs Proxy Routing

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          ROUTING TRADE-OFF MATRIX                           │
├───────────────────────────┬────────────────────────┬───────────────────────┤
│         ASPECT            │   FINE-GRAINED ROUTES  │     PROXY ROUTING     │
├───────────────────────────┼────────────────────────┼───────────────────────┤
│ Setup Complexity          │ Higher                 │ Lower                 │
│                           │ (define each resource) │ (one {proxy+})        │
├───────────────────────────┼────────────────────────┼───────────────────────┤
│ Per-Route Auth            │ ✅ Easy                │ ❌ Backend must handle│
├───────────────────────────┼────────────────────────┼───────────────────────┤
│ Request Validation        │ ✅ API Gateway handles │ ❌ Backend must handle│
├───────────────────────────┼────────────────────────┼───────────────────────┤
│ Throttling Control        │ ✅ Per-route possible  │ ❌ All routes same    │
├───────────────────────────┼────────────────────────┼───────────────────────┤
│ Caching                   │ ✅ Effective           │ △ Less effective     │
├───────────────────────────┼────────────────────────┼───────────────────────┤
│ Documentation             │ ✅ Auto-generated      │ △ Generic            │
├───────────────────────────┼────────────────────────┼───────────────────────┤
│ Maintenance               │ Higher                 │ Lower                 │
│                           │ (sync routes)          │ (backend owns routes) │
├───────────────────────────┼────────────────────────┼───────────────────────┤
│ 404 Handling              │ API Gateway returns    │ Backend must return   │
│                           │ 404 for unknown paths  │ 404 for unknown paths │
├───────────────────────────┼────────────────────────┼───────────────────────┤
│ Security Surface          │ Smaller                │ Larger                │
│                           │ (only defined routes)  │ (all paths accepted)  │
└───────────────────────────┴────────────────────────┴───────────────────────┘
```

---

## 5. Routing Comparison Summary

### Complete Comparison Diagram

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ALB vs API GATEWAY ROUTING                               │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ALB APPROACH                                                               │
│  ─────────────                                                              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Listener                                                            │   │
│  │  └── Rule: /dev/api1/* → Target Group → Lambda                       │   │
│  │  └── Rule: /dev/api2/* → Target Group → Lambda                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  • One rule = one base path                                                 │
│  • Wildcard handles all subpaths                                            │
│  • Simple, but no gateway-level features                                    │
│                                                                             │
│                                                                             │
│  API GATEWAY APPROACH                                                       │
│  ────────────────────                                                       │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Resource Tree                                                       │   │
│  │  /dev                                                                │   │
│  │  ├── /api1                                                           │   │
│  │  │   └── /{proxy+}                                                   │   │
│  │  │       └── ANY → Lambda 1                                          │   │
│  │  └── /api2                                                           │   │
│  │      └── /{proxy+}                                                   │   │
│  │          └── ANY → Lambda 2                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  • Hierarchical resource structure                                          │
│  • {proxy+} mimics wildcard behavior                                        │
│  • Full API management features available                                   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

**← Previous:** [03-architecture.md](03-architecture.md) | **Next:** [05-hands-on-implementation.md](05-hands-on-implementation.md) →
