# API Gateway Fundamentals

This section explains what API Gateway is, the different API types, and helps you understand when to use (and when not to use) API Gateway.

---

## 1. What is AWS API Gateway?

### Definition

AWS API Gateway is a **fully managed service** that allows you to create, publish, maintain, monitor, and secure APIs at any scale.

### What Problems It Solves

```
Without API Gateway                    With API Gateway
─────────────────────                  ──────────────────
                                       
┌────────┐   ┌────────┐               ┌────────┐
│Client 1│──►│Lambda 1│               │Client 1│──┐
└────────┘   └────────┘               └────────┘  │    ┌─────────────┐
                                                   ├───►│             │   ┌────────┐
┌────────┐   ┌────────┐               ┌────────┐  │    │ API Gateway │──►│Lambda 1│
│Client 2│──►│Lambda 2│               │Client 2│──┤    │             │   └────────┘
└────────┘   └────────┘               └────────┘  │    │  - Routing  │
                                                   │    │  - Security │   ┌────────┐
┌────────┐   ┌────────┐               ┌────────┐  │    │  - Logging  │──►│Lambda 2│
│Client 3│──►│Lambda 3│               │Client 3│──┘    │  - Throttle │   └────────┘
└────────┘   └────────┘               └────────┘       └─────────────┘

Problems:                              Benefits:
- No unified entry point               - Single entry point
- Security per function                - Centralized security
- No rate limiting                     - Built-in throttling
- Scattered logging                    - Unified logging
- Direct Lambda exposure               - Abstraction layer
```

### Key Capabilities

| Capability | Description |
|------------|-------------|
| **Request Routing** | Route requests to different backends based on path/method |
| **Request Transformation** | Modify requests before forwarding to backend |
| **Response Transformation** | Modify responses before returning to client |
| **Authentication/Authorization** | API keys, IAM, Cognito, Lambda authorizers |
| **Rate Limiting/Throttling** | Protect backends from overload |
| **Caching** | Reduce backend calls, improve response time |
| **Monitoring** | CloudWatch integration for metrics and logs |
| **API Versioning** | Use stages to manage versions (dev, staging, prod) |

---

## 2. API Types: REST vs HTTP vs WebSocket

AWS API Gateway offers three API types. Understanding the differences is crucial.

### Comparison Table

| Feature | REST API | HTTP API | WebSocket API |
|---------|----------|----------|---------------|
| **Use Case** | Full-featured APIs | Simple, low-latency APIs | Real-time bidirectional |
| **Private API Support** | ✅ **Yes** | ❌ No | ❌ No |
| **Pricing** | Higher | ~70% cheaper | Per-message pricing |
| **API Keys** | ✅ Yes | ✅ Yes | ❌ No |
| **Usage Plans** | ✅ Yes | ❌ No | ❌ No |
| **Resource Policies** | ✅ Yes | ❌ No | ❌ No |
| **Request Validation** | ✅ Yes | ❌ No | ❌ No |
| **Caching** | ✅ Yes | ❌ No | ❌ No |
| **WAF Integration** | ✅ Yes | ❌ No | ✅ Yes |
| **Edge Optimized** | ✅ Yes | ❌ No (Regional only) | ❌ No |

### Visual Decision Guide

```
                     Need Private API?
                           │
              ┌────────────┴────────────┐
              │                         │
             YES                        NO
              │                         │
              ▼                         │
        ┌──────────┐                    │
        │ REST API │◄───────────────────┤
        │ (Required)│                   │
        └──────────┘                    ▼
                                  Need WebSocket?
                                        │
                             ┌──────────┴──────────┐
                             │                     │
                            YES                    NO
                             │                     │
                             ▼                     ▼
                      ┌────────────┐        Need Advanced
                      │ WebSocket  │        Features?
                      │    API     │              │
                      └────────────┘    ┌────────┴────────┐
                                        │                 │
                                       YES                NO
                                        │                 │
                                        ▼                 ▼
                                  ┌──────────┐     ┌──────────┐
                                  │ REST API │     │ HTTP API │
                                  └──────────┘     │ (Cheaper)│
                                                   └──────────┘
```

---

## 3. Why REST APIs Are Required for Private APIs

### The Key Limitation

> **HTTP APIs do NOT support Private endpoints. Only REST APIs can be configured as Private APIs.**

### Technical Reason

| Feature | REST API | HTTP API |
|---------|----------|----------|
| **Endpoint Types** | Edge, Regional, **Private** | Regional only |
| **Resource Policies** | ✅ Supported | ❌ Not supported |
| **VPC Endpoint Integration** | ✅ Full support | ❌ Not supported |

Private APIs require:
1. **Resource policies** to restrict access to VPC endpoints
2. **VPC endpoint association** for private connectivity

Since HTTP APIs don't support these, **REST APIs are the only option for Private APIs**.

### When This Matters

| Scenario | API Type Required |
|----------|-------------------|
| Internal microservices in VPC | REST API (Private) |
| Internal APIs, no internet exposure | REST API (Private) |
| Public-facing lightweight APIs | HTTP API (cheaper) |
| APIs needing caching, validation | REST API |
| Real-time chat, notifications | WebSocket API |

---

## 4. Conceptual Comparison: API Gateway vs Internal ALB + Lambda

This is a **critical comparison** for understanding when to use each approach.

### Architecture Comparison

```
Internal ALB + Lambda                    API Gateway Private REST API
─────────────────────                    ───────────────────────────

┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│            VPC                   │     │            VPC                   │
│                                  │     │                                  │
│  ┌────────┐    ┌─────────────┐  │     │  ┌────────┐    ┌─────────────┐  │
│  │ Client │───►│ Internal ALB│  │     │  │ Client │───►│ VPC Endpoint│  │
│  └────────┘    └──────┬──────┘  │     │  └────────┘    └──────┬──────┘  │
│                       │          │     │                       │          │
│         ┌─────────────┼──────┐  │     │                       │          │
│         │             │      │  │     └───────────────────────┼──────────┘
│         ▼             ▼      │  │                             │
│    ┌────────┐    ┌────────┐  │  │                             ▼
│    │Lambda 1│    │Lambda 2│  │  │                    ┌─────────────────┐
│    │(Target │    │(Target │  │  │                    │   API Gateway   │
│    │ Group) │    │ Group) │  │  │                    │  (Private API)  │
│    └────────┘    └────────┘  │  │                    └────────┬────────┘
│         (via Target Groups)  │  │                             │
└─────────────────────────────────┘     │            ┌──────────┴──────────┐
                                         │            ▼                     ▼
                                         │       ┌────────┐            ┌────────┐
                                         │       │Lambda 1│            │Lambda 2│
                                         │       └────────┘            └────────┘
                                         │
```

### Feature Comparison

| Feature | Internal ALB | API Gateway Private |
|---------|-------------|---------------------|
| **Routing** | Path/host-based rules | Resource-based + methods |
| **Wildcard Paths** | ✅ Native (`/*`) | Via `{proxy+}` resource |
| **Authentication** | None built-in | API keys, IAM, Cognito, Lambda auth |
| **Rate Limiting** | ❌ None | ✅ Built-in throttling |
| **Request Validation** | ❌ None | ✅ Schema validation |
| **Caching** | ❌ None | ✅ Built-in caching |
| **API Versioning** | Manual | ✅ Stages |
| **Usage Plans** | ❌ None | ✅ Quotas per client |
| **Cost Model** | Per hour + LCU | Per million requests |
| **Setup Complexity** | Lower | Higher |
| **Health Checks** | ✅ Built-in | ❌ None (Lambda handles) |

### When to Choose Each

#### Choose Internal ALB When:
- Simple internal routing with no security requirements
- Already using ALB for other services
- Need native health checks for backend services
- Want simpler configuration
- Cost optimization for steady, high-volume traffic

#### Choose API Gateway When:
- Need authentication/authorization
- Require rate limiting and throttling
- Want request validation
- Need usage tracking per client
- Require response caching
- API versioning is important
- Centralized API management is desired

---

## 5. When API Gateway Is NOT the Right Choice

### Scenarios Where API Gateway May Be Overkill

```
┌─────────────────────────────────────────────────────────────────┐
│                  DON'T USE API Gateway When:                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Simple Internal Routing Only                                 │
│     ├── No auth needed                                           │
│     ├── No rate limiting needed                                  │
│     └── ALB path routing is sufficient                           │
│                                                                  │
│  2. Non-HTTP Protocols                                           │
│     ├── TCP-based services                                       │
│     ├── gRPC without HTTP/2 (use NLB)                           │
│     └── UDP traffic                                              │
│                                                                  │
│  3. Extremely High Throughput + Cost Sensitive                   │
│     ├── Millions of requests with simple routing                 │
│     └── ALB may be more cost-effective                           │
│                                                                  │
│  4. Direct Service-to-Service Communication                      │
│     ├── Lambda to Lambda (use direct invoke)                     │
│     └── Container to container (use service mesh)                │
│                                                                  │
│  5. Long-Running Connections                                     │
│     ├── Streaming (unless WebSocket)                             │
│     └── Requests > 30 seconds (REST API timeout limit)           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Limitations to Consider

| Limitation | Value | Impact |
|------------|-------|--------|
| **Payload Size** | 10 MB (REST), 10 MB (HTTP) | Large file uploads need presigned S3 URLs |
| **Timeout** | 30 seconds (REST) | Long-running tasks need async patterns |
| **Request Rate** | 10,000 RPS default | Can be increased via quota request |
| **Concurrent Connections** | Varies | May need to request increase |
| **WebSocket Message Size** | 128 KB | Large messages need chunking |

---

## Summary: API Gateway Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                    API Gateway Decision Matrix                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐                                             │
│  │ Is it internal/ │─── YES ───► Must use REST API (Private)    │
│  │ VPC-only?       │                                             │
│  └────────┬────────┘                                             │
│           │ NO                                                   │
│           ▼                                                      │
│  ┌─────────────────┐                                             │
│  │ Need advanced   │─── YES ───► Use REST API                    │
│  │ features?       │                                             │
│  │ (cache, WAF,    │                                             │
│  │  validation)    │                                             │
│  └────────┬────────┘                                             │
│           │ NO                                                   │
│           ▼                                                      │
│  ┌─────────────────┐                                             │
│  │ Cost is primary │─── YES ───► Use HTTP API                    │
│  │ concern?        │                                             │
│  └────────┬────────┘                                             │
│           │ NO                                                   │
│           ▼                                                      │
│  ┌─────────────────┐                                             │
│  │ Real-time       │─── YES ───► Use WebSocket API               │
│  │ bidirectional?  │                                             │
│  └────────┬────────┘                                             │
│           │ NO                                                   │
│           ▼                                                      │
│       Consider REST API (most flexible)                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

**← Previous:** [01-prerequisites.md](01-prerequisites.md) | **Next:** [03-architecture.md](03-architecture.md) →
