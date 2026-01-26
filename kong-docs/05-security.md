# Kong Security - API Keys, Rate Limiting, Access Control

This document covers security features in Kong, mapping to AWS API Gateway equivalents.

---

## Security Feature Mapping

| AWS API Gateway | Kong Plugin |
|-----------------|-------------|
| API Keys + Usage Plans | Key Authentication + Rate Limiting |
| Resource Policy (VPC) | ACL + IP Restriction |
| IAM Authorization | Custom plugin / OIDC |
| Lambda Authorizer | Custom auth plugin |
| Rate Limiting | Rate Limiting plugin |
| Request Validation | Request Validator plugin |

---

## 1. API Key Authentication

### What API Keys Protect

| Protects | Does NOT Protect |
|----------|------------------|
| ✅ Identifies client | ❌ Does not verify identity |
| ✅ Ties to rate limits | ❌ Keys can be shared/stolen |
| ✅ Usage tracking | ❌ Does not encrypt requests |

### Enable Key Auth Plugin

#### DB-less Configuration

```yaml
_format_version: "3.0"

consumers:
  - username: client-app-1
    keyauth_credentials:
      - key: my-secret-api-key-12345

  - username: client-app-2
    keyauth_credentials:
      - key: another-api-key-67890

plugins:
  - name: key-auth
    config:
      key_names:
        - x-api-key
        - apikey
      hide_credentials: true

services:
  - name: protected-service
    url: http://backend:8080
    routes:
      - name: protected-route
        paths:
          - /dev/api/protected
```

#### Admin API Configuration

```bash
# Enable key-auth globally
curl -X POST http://localhost:8001/plugins \
  -d name=key-auth \
  -d "config.key_names[]=x-api-key"

# Create consumer
curl -X POST http://localhost:8001/consumers \
  -d username=client-app-1

# Create API key for consumer
curl -X POST http://localhost:8001/consumers/client-app-1/key-auth \
  -d key=my-secret-api-key-12345

# List keys
curl http://localhost:8001/key-auths
```

### Per-Route API Keys

```yaml
services:
  - name: api-service
    url: http://backend:8080
    routes:
      - name: public-route
        paths:
          - /dev/api/public
        # No key-auth plugin = no key required

      - name: protected-route
        paths:
          - /dev/api/protected
        plugins:
          - name: key-auth
            config:
              key_names: [x-api-key]
```

### Testing

```bash
# Without API key - 401 Unauthorized
curl http://localhost:8000/dev/api/protected
# Response: {"message":"No API key found in request"}

# With API key - Success
curl -H "x-api-key: my-secret-api-key-12345" \
  http://localhost:8000/dev/api/protected
```

---

## 2. Rate Limiting

### Comparison with AWS Usage Plans

| AWS Usage Plans | Kong Rate Limiting |
|-----------------|-------------------|
| Per stage | Global / per route / per consumer |
| Per API key | Per consumer / per credential |
| Quota (monthly) | Period-based (second/minute/hour/day/month/year) |
| Throttling (burst) | Window-based limiting |

### Configuration

#### Global Rate Limit

```yaml
plugins:
  - name: rate-limiting
    config:
      second: 10
      minute: 100
      hour: 1000
      policy: local
      hide_client_headers: false
```

#### Per-Route Rate Limit

```yaml
services:
  - name: api-service
    url: http://backend:8080
    routes:
      - name: heavy-route
        paths:
          - /dev/api/heavy
        plugins:
          - name: rate-limiting
            config:
              minute: 10
              policy: local

      - name: light-route
        paths:
          - /dev/api/light
        plugins:
          - name: rate-limiting
            config:
              minute: 1000
              policy: local
```

#### Per-Consumer Rate Limit

```yaml
consumers:
  - username: free-tier
    plugins:
      - name: rate-limiting
        config:
          minute: 10

  - username: premium-tier
    plugins:
      - name: rate-limiting
        config:
          minute: 1000
```

### Rate Limit Response Headers

```
X-RateLimit-Limit-Minute: 100
X-RateLimit-Remaining-Minute: 95
RateLimit-Limit: 100
RateLimit-Remaining: 95
RateLimit-Reset: 45
```

### When Limit Exceeded

```json
HTTP/1.1 429 Too Many Requests
{
  "message": "API rate limit exceeded"
}
```

---

## 3. IP Restriction

Restrict API access to specific IPs or CIDR ranges.

```yaml
plugins:
  - name: ip-restriction
    config:
      allow:
        - 10.0.0.0/8       # VPC CIDR
        - 192.168.1.0/24   # Specific subnet
      deny:
        - 0.0.0.0/0        # Deny all others (implicit)
```

### Per-Route IP Restriction

```yaml
services:
  - name: admin-service
    url: http://admin-backend:8080
    routes:
      - name: admin-route
        paths:
          - /dev/api/admin
        plugins:
          - name: ip-restriction
            config:
              allow:
                - 10.0.1.0/24  # Admin subnet only
```

---

## 4. Access Control Lists (ACL)

Group consumers and control access by group.

### Configuration

```yaml
consumers:
  - username: admin-user
    acls:
      - group: admin-group

  - username: read-user
    acls:
      - group: read-only-group

services:
  - name: api-service
    url: http://backend:8080
    routes:
      - name: admin-route
        paths:
          - /dev/api/admin
        plugins:
          - name: key-auth
          - name: acl
            config:
              allow:
                - admin-group

      - name: read-route
        paths:
          - /dev/api/read
        plugins:
          - name: key-auth
          - name: acl
            config:
              allow:
                - admin-group
                - read-only-group
```

---

## 5. Request Validation

Validate request body against JSON schema.

```yaml
plugins:
  - name: request-validator
    config:
      body_schema: |
        {
          "type": "object",
          "required": ["name", "email"],
          "properties": {
            "name": {"type": "string", "minLength": 1},
            "email": {"type": "string", "format": "email"}
          }
        }
```

---

## Complete Security Stack Example

```yaml
_format_version: "3.0"

consumers:
  - username: internal-service
    keyauth_credentials:
      - key: internal-key-xxxxx
    acls:
      - group: internal
    plugins:
      - name: rate-limiting
        config:
          minute: 1000

  - username: external-partner
    keyauth_credentials:
      - key: partner-key-yyyyy
    acls:
      - group: external
    plugins:
      - name: rate-limiting
        config:
          minute: 100

services:
  - name: api-backend
    url: http://backend:8080
    routes:
      - name: internal-api
        paths:
          - /dev/api/internal
        plugins:
          - name: key-auth
          - name: acl
            config:
              allow: [internal]
          - name: ip-restriction
            config:
              allow: [10.0.0.0/8]

      - name: partner-api
        paths:
          - /dev/api/partner
        plugins:
          - name: key-auth
          - name: acl
            config:
              allow: [internal, external]
```

---

## Security Best Practices

| Practice | Implementation |
|----------|----------------|
| Never expose Admin API | Bind to 127.0.0.1 only |
| Rotate API keys | Use consumer management |
| Layer security | Key auth + rate limit + ACL |
| Log all access | Enable file-log or http-log plugin |
| Use HTTPS | Configure SSL/TLS in Kong or ALB |

---

**← Previous:** [04-routing.md](04-routing.md) | **Next:** [06-lambda-integration.md](06-lambda-integration.md) →
