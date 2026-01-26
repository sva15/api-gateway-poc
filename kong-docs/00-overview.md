# Kong API Gateway POC - Documentation Overview

This documentation provides a guide for implementing **Kong API Gateway** on AWS, integrating with an existing ALB-based architecture, and achieving feature parity with AWS API Gateway.

---

## Background: Why Kong?

Kong is an **open-source API Gateway** that can run anywhere (EC2, ECS, Kubernetes) and provides:
- Full control over deployment
- No per-request AWS charges
- Plugin ecosystem for extensibility
- Open-source or Enterprise options

---

## Documentation Structure

| File | Topic | Description |
|------|-------|-------------|
| [01-kong-vs-aws-apigw.md](01-kong-vs-aws-apigw.md) | Comparison | Feature parity, differences, pros/cons |
| [02-architecture.md](02-architecture.md) | Architecture | Kong deployment in existing ALB setup |
| [03-installation.md](03-installation.md) | Installation | Kong setup on EC2 with Docker |
| [04-routing.md](04-routing.md) | Routing | Proxy-based and explicit routing |
| [05-security.md](05-security.md) | Security | API keys, rate limiting, access control |
| [06-lambda-integration.md](06-lambda-integration.md) | Lambda | Calling AWS Lambda from Kong |
| [07-operations.md](07-operations.md) | Operations | Logging, metrics, configuration |

---

## Current Architecture (Preserved)

```
┌─────────────────────────────────────────────────────────────┐
│                    Internal ALB                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  /ui/*              → Nginx (port 80) → Angular UI          │
│  /dev/ui/*          → Nginx → Angular UI                    │
│  /dev/configurator/*→ Nginx → Angular UI                    │
│                                                              │
│  [EXISTING - NO CHANGE]                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Target Architecture (With Kong)

```
┌─────────────────────────────────────────────────────────────┐
│                    Internal ALB                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  /ui/*              → Nginx Target Group → Angular UI       │
│  /dev/ui/*          → Nginx Target Group → Angular UI       │
│                                                              │
│  /dev/api/*         → Kong Target Group  → Kong Gateway     │
│                              │                               │
│                              ├─→ Lambda Functions           │
│                              ├─→ EC2 Backends               │
│                              └─→ Internal Services          │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Constraints

| Constraint | Requirement |
|------------|-------------|
| **UI Layer** | Cannot be changed |
| **ALB** | Add target group, don't restructure |
| **Kong Location** | Same EC2 as UI/Nginx |
| **API Paths** | `/dev/api/*` routes to Kong |
| **UI Paths** | Stay on Nginx |

---

## Learning Outcomes

After completing this POC, you will:

1. ✅ Know how to deploy Kong in an existing ALB architecture
2. ✅ Understand Kong's routing model vs AWS API Gateway
3. ✅ Implement both proxy-based and explicit API routing
4. ✅ Secure APIs with API keys and rate limiting
5. ✅ Decide if Kong can replace AWS API Gateway for your use case

---

**Next:** [01-kong-vs-aws-apigw.md](01-kong-vs-aws-apigw.md) →
