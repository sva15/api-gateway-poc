# Kong vs AWS API Gateway - Feature Comparison

A detailed comparison to help decide between Kong and AWS API Gateway.

---

## Quick Comparison

| Aspect | Kong | AWS API Gateway |
|--------|------|-----------------|
| **Type** | Self-managed / Enterprise | Fully managed |
| **Deployment** | EC2, ECS, K8s, anywhere | AWS only |
| **Pricing** | Free (OSS) or license | Per-request + data transfer |
| **Private APIs** | Any network | Requires VPC Endpoint |
| **Extensibility** | Plugin ecosystem | Limited to AWS features |

---

## Feature Parity Matrix

### Routing Features

| Feature | AWS API Gateway | Kong | Notes |
|---------|-----------------|------|-------|
| Path-based routing | ✅ | ✅ | Both support |
| Method-based routing | ✅ | ✅ | Both support |
| Wildcard paths (`{proxy+}`) | ✅ | ✅ | Kong uses regex or prefix |
| Query param routing | ✅ | ✅ | Kong via plugins |
| Header-based routing | ❌ | ✅ | Kong more flexible |
| Host-based routing | Via custom domains | ✅ | Kong native |

### Security Features

| Feature | AWS API Gateway | Kong | Notes |
|---------|-----------------|------|-------|
| API Keys | ✅ | ✅ | Both support |
| Rate Limiting | ✅ (Usage Plans) | ✅ (Plugin) | Kong more granular |
| IP Whitelisting | ✅ (Resource Policy) | ✅ (Plugin) | Both support |
| IAM Auth | ✅ Native | ❌ Needs custom | AWS advantage |
| JWT Validation | ✅ (Cognito) | ✅ (Plugin) | Both support |
| mTLS | ✅ | ✅ | Both support |
| OAuth2 | Via Cognito | ✅ (Plugin) | Kong more flexible |

### Backend Integration

| Feature | AWS API Gateway | Kong | Notes |
|---------|-----------------|------|-------|
| Lambda | ✅ Native | ⚠️ Via plugin/HTTP | AWS easier |
| HTTP backends | ✅ | ✅ | Both support |
| VPC resources | ✅ (VPC Link) | ✅ (Direct) | Kong simpler |
| gRPC | ✅ (HTTP API) | ✅ | Both support |
| WebSocket | ✅ | ✅ | Both support |

### Operations

| Feature | AWS API Gateway | Kong | Notes |
|---------|-----------------|------|-------|
| Logging | CloudWatch | File/HTTP/Plugins | Kong more flexible |
| Metrics | CloudWatch | Prometheus/StatsD | Kong open ecosystem |
| Tracing | X-Ray | Zipkin/Jaeger | Both support |
| Caching | ✅ (REST only) | ✅ (Plugin) | Both support |
| Request Validation | ✅ | ✅ (Plugin) | Both support |

---

## What Kong Does Better

### 1. Deployment Flexibility
```
Kong can run:
- Same EC2 as your app
- Dedicated EC2
- ECS/EKS
- On-premises
- Any cloud
```

### 2. Cost at Scale
```
AWS API Gateway: $3.50 per million requests + data
Kong OSS:        Free (just EC2/compute costs)

At 100M requests/month:
- AWS: ~$350/month + data transfer
- Kong: ~$50-100/month (EC2 cost only)
```

### 3. Plugin Ecosystem
```
Kong has 100+ plugins for:
- Authentication (OAuth, LDAP, OIDC)
- Traffic control (rate limit, circuit breaker)
- Transformations (request/response)
- Logging (file, HTTP, Kafka, syslog)
- Analytics (Prometheus, Datadog)
```

### 4. Header-Based Routing
```yaml
# Kong can route based on any header
routes:
  - name: version-2
    paths: ["/api"]
    headers:
      x-api-version: ["v2"]
    service: api-v2-service
```

### 5. No Vendor Lock-in
```
Can migrate Kong anywhere:
- AWS → GCP
- Cloud → On-premises
- Vice versa
```

---

## What AWS API Gateway Does Better

### 1. Lambda Integration
```
AWS: Native, automatic permissions, no setup
Kong: Requires AWS Lambda plugin, IAM configuration
```

### 2. IAM Authorization
```
AWS: Native SigV4, CloudTrail audit, cross-account
Kong: Must implement custom authorizer
```

### 3. Zero Operations
```
AWS: Fully managed, no servers to maintain
Kong: Must manage EC2, updates, scaling, HA
```

### 4. Private API (VPC-only)
```
AWS: Built-in via VPC Endpoint + Resource Policy
Kong: Must configure networking manually
```

### 5. Stage Management
```
AWS: Native stages (dev, prod) with variables
Kong: Manual via workspaces or tags
```

---

## Integration with Existing ALB Architecture

### AWS API Gateway Approach
```
ALB → VPC Endpoint → API Gateway → Lambda
     (requires changes)
```

### Kong Approach
```
ALB → Kong Target Group → Kong → Lambda/EC2
     (just add target group)
```

**Winner for your use case: Kong** - Simpler integration with existing ALB.

---

## Concept Mapping

| AWS API Gateway | Kong Equivalent |
|-----------------|-----------------|
| REST API | Service + Routes |
| Resource | Route path |
| Method | Route methods |
| Stage | Upstream + Route tags |
| Integration | Service upstream |
| Usage Plan | Rate Limiting Plugin |
| API Key | Key Auth Plugin |
| Resource Policy | ACL Plugin / Network |
| Lambda Authorizer | Custom Auth Plugin |
| Request Validator | Request Validator Plugin |

---

## When to Choose Which

### Choose Kong When:

| Scenario | Reason |
|----------|--------|
| High request volume | Cost savings at scale |
| Existing ALB architecture | Easy integration |
| Need header-based routing | Native support |
| Multi-cloud strategy | Portability |
| Advanced plugins needed | Extensibility |
| On-premises requirement | Runs anywhere |

### Choose AWS API Gateway When:

| Scenario | Reason |
|----------|--------|
| Lambda-heavy architecture | Native integration |
| IAM auth required | Built-in SigV4 |
| Zero ops preference | Fully managed |
| Low request volume | Pay-per-use cheaper |
| AWS-only environment | Simplicity |
| Private API with VPC | Built-in support |

---

## Recommendation for Your Use Case

Based on your requirements:

| Factor | Recommendation |
|--------|----------------|
| Existing ALB | ✅ Kong integrates easier |
| Lambda backends | ⚠️ Requires plugin setup |
| API Keys | ✅ Both work |
| Internal-only | ✅ Kong behind ALB works |
| Operational overhead | ⚠️ Kong needs maintenance |

**Verdict:** Kong is a viable option for your architecture. The main trade-off is operational overhead vs. cost savings and flexibility.

---

**← Back to:** [00-overview.md](00-overview.md) | **Next:** [02-architecture.md](02-architecture.md) →
