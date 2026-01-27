# Kong Gateway Configuration & Decision Guide

**Purpose:** This document outlines the architectural choices for routing, security, and integration in our Kong Gateway implementation. Use this guide to facilitate team discussions and finalize the design pattern.

---

## 1. API Routing Strategy

How will client requests be routed to backend microservices/Lambdas?

### Option A: Path-Based Routing (Recommended)
Routing based on the URL path.
- **Example:** `api.example.com/users` → `User Service`
- **Example:** `api.example.com/orders` → `Order Service`
- **Pros:** Single domain, easy to manage, intuitive API structure.
- **Cons:** Shared domain bandwidth/limits.

### Option B: Host-Based Routing
Routing based on the `Host` header (Subdomains).
- **Example:** `users.api.example.com` → `User Service`
- **Example:** `orders.api.example.com` → `Order Service`
- **Pros:** Stronger isolation, independent scaling/DNS.
- **Cons:** Requires wildcard certificate or multiple certs, more DNS management.

### Option C: Header-Based / Versioning
Routing based on custom headers (often used for versioning).
- **Example:** `GET /users` + `Accept-Version: v2` → `User Service v2`
- **Pros:** Clean URLs.
- **Cons:** Harder to test from browser address bar, caching complexity.

### 📋 Decision Matrix
| Feature | Path-Based | Host-Based | Header-Based |
|---------|:----------:|:----------:|:------------:|
| **Complexity** | Low | Medium | High |
| **DNS Management** | Simple (1 Record) | Complex (*.api) | Simple |
| **Isolation** | Shared | Isolated | Shared |
| **Browser Friendly**| Yes | Yes | No |
| **Recommendation**| ✅ Start Here | For large scale | For Versioning |

---

## 2. Security Configuration

How will we secure the APIs? (Focusing on Kong Free Tier capabilities)

### 2.1 Authentication (AuthN)

#### Option A: Key Authentication (API Keys)
Clients send a secret key in header (`x-api-key: abc123xyz`).
- **Best for:** Internal services, B2B partners, trusted clients.
- **Kong Plugin:** `key-auth` (Free)
- **Pros:** Simple, fast, easy revocation.
- **Cons:** Key management overhead, need secure distribution.

#### Option B: Basic Authentication
Username/Password (Base64 encoded).
- **Best for:** Legacy systems, simple internal tools.
- **Kong Plugin:** `basic-auth` (Free)
- **Pros:** Universally supported.
- **Cons:** Credentials sent with every request (requires HTTPS), no expiration.

#### Option C: OAuth2 / OIDC (Advanced)
Token-based using an Identity Provider (Cognito, Auth0).
- **Best for:** Public mobile/web apps, user-centric security.
- **Kong Plugin:** `oauth2` (Free), `openid-connect` (Enterprise only).
- **Workaround:** For Free tier, use the `jwt` plugin and validate tokens issued by IDP.
- **Pros:** Standard, secure, short-lived tokens.
- **Cons:** High complexity setup.

### 2.2 Traffic Control (Rate Limiting)

#### Option A: Global Rate Limiting
Limit total requests per second/minute for the entire gateway.
- **Pros:** Protects infrastructure.
- **Cons:** One abusive tenant can block everyone.

#### Option B: Consumer-Based Rate Limiting (Recommended)
Limit requests per consumer (API Key).
- **Example:** "Silver Tier" = 100 req/min, "Gold Tier" = 1000 req/min.
- **Pros:** Fair usage, monetization potential.
- **Cons:** Requires managing consumer groups.

### 📋 Security Decision Checklist

- [ ] **Auth Strategy:**
    - [ ] Key Auth (Simple, B2B)
    - [ ] JWT/OAuth (User facing)
    - [ ] Basic Auth (Legacy)

- [ ] **Rate Limiting:**
    - [ ] By IP Address (Anonymous protection)
    - [ ] By Consumer (Tiered access)
    - [ ] None (Internal only)

- [ ] **Network Security:**
    - [ ] Public Load Balancer (Internet facing)
    - [ ] Private Load Balancer (Internal VPC only)
    - [ ] IP Allow-listing (Kong `ip-restriction` plugin)

---

## 3. Deployment & Integration

### 3.1 Lambda Integration Pattern

#### Option A: Kong AWS Lambda Plugin (Recommended)
Kong directly invokes Lambda via AWS SDK.
- **Flow:** Client → ALB → Kong → Lambda
- **Pros:** No API Gateway cost, lower latency, unified config.
- **Cons:** Kong needs IAM permissions.

#### Option B: Double Proxy (via AWS API Gateway)
Kong proxies to AWS API Gateway URL.
- **Flow:** Client → ALB → Kong → AWS API Gateway → Lambda
- **Pros:** Use AWS managed features.
- **Cons:** Double cost, extra hop/latency.

---

## 4. Team Preferences Form

Please fill this out to guide the implementation:

1. **Routing Preferred:** _________________________ (Path / Host)
2. **Auth Mechanism:** ___________________________ (Key / JWT / None)
3. **Public or Private Access:** __________________
4. **Rate Limit Requirements:** ___________________
5. **Timeline for POC:** __________________________
