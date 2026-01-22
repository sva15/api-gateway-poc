# AWS API Gateway Private REST APIs - Documentation Overview

## Purpose

This documentation provides a **progressive proof-of-concept (POC)** guide for implementing **AWS API Gateway REST APIs (Private only)**, designed to evolve into a **dev-environment-ready internal API setup** suitable for real projects.

---

## Background: From ALB to API Gateway

Previously, internal APIs were exposed using an **internal Application Load Balancer (ALB)** with **path-based routing**:

```
/dev/api1/* → Lambda Target Group 1
/dev/api2/* → Lambda Target Group 2
```

This documentation guides you through **replacing this design with API Gateway**, understanding the routing differences, and validating base-path routing with wildcard subpaths.

---

## Documentation Structure

This guide is split into logically grouped files for easier navigation:

| File | Topic | Description |
|------|-------|-------------|
| [01-prerequisites.md](01-prerequisites.md) | Prerequisites | Required AWS concepts and setup |
| [02-fundamentals.md](02-fundamentals.md) | API Gateway Fundamentals | Core concepts, API types, comparisons |
| [03-architecture.md](03-architecture.md) | Private REST API Architecture | Request flow, components, and troubleshooting |
| [04-routing.md](04-routing.md) | Path-Based Routing | ALB vs API Gateway routing comparison |
| [05-hands-on-implementation.md](05-hands-on-implementation.md) | Hands-On Implementation | Step-by-step POC build with warnings |
| [06-security.md](06-security.md) | Security & API Protection | API Keys, IAM, Resource Policies |
| [07-operations.md](07-operations.md) | Dev-Ready Operations | Logging, monitoring, best practices |
| [08-troubleshooting.md](08-troubleshooting.md) | Troubleshooting | Common issues and solutions |
| [09-team-decision-guide.md](09-team-decision-guide.md) | Team Decision Guide | Routing & security options for team |
| [10-visual-implementation-guide.md](10-visual-implementation-guide.md) | **Visual Guide** | Screenshots from AWS Console |
| [11-explicit-path-routing.md](11-explicit-path-routing.md) | Explicit Routing | Non-proxy approach with per-endpoint control |
| [12-method-configuration-guide.md](12-method-configuration-guide.md) | Method Configuration | Integration types, proxy vs non-proxy, ANY vs individual |

---

## Mandatory Constraints

All implementations in this guide follow these constraints:

| Constraint | Requirement |
|------------|-------------|
| **API Type** | REST APIs only |
| **Access Type** | Private APIs only |
| **Network** | All traffic inside VPC |
| **Connectivity** | VPC Interface Endpoints |
| **Internet Exposure** | None |
| **Tooling** | AWS Console only (no IaC) |
| **Lambda Runtime** | Python |

---

## Target Outcome

After completing this guide, you will:

1. ✅ Fully understand **AWS API Gateway Private REST APIs**
2. ✅ Know how to replace **internal ALB path routing** correctly
3. ✅ Use `{proxy+}` safely and intentionally
4. ✅ Secure and operate internal APIs properly
5. ✅ Confidently design real project APIs using API Gateway

---

## How to Use This Guide

1. **Start with Prerequisites** - Ensure you understand the foundational concepts
2. **Read Fundamentals** - Understand what API Gateway is and when to use it
3. **Study Architecture** - Learn how private APIs work internally
4. **Master Routing** - Critical section comparing ALB vs API Gateway routing
5. **Complete Hands-On** - Build the POC step-by-step
6. **Implement Security** - Add protection layers
7. **Configure Operations** - Make it production-ready

> **Note:** This guide assumes you are technical but new to API Gateway. Each section builds progressively on the previous one.

---

## Quick Reference: Final API Structure

The POC implements this routing structure:

```
/dev
  ├── /api1
  │     └── /{proxy+}  → api1 Lambda
  └── /api2
        └── /{proxy+}  → api2 Lambda
```

**Expected Behavior:**
- `/dev/api1/test` → handled by api1 Lambda
- `/dev/api1/foo/bar` → handled by api1 Lambda
- `/dev/api2/hello` → handled by api2 Lambda
- `/dev/api2/v1/check` → handled by api2 Lambda

---

**Next:** [01-prerequisites.md](01-prerequisites.md) →
