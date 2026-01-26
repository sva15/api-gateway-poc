# Kong Free Tier - Available Components & Plugins for POC

Based on your Kong Gateway 3.13.0.0 setup, this guide identifies the key components and plugins you can use for the POC.

---

## Your Kong Setup

| Component | Value |
|-----------|-------|
| **Kong Version** | 3.13.0.0 |
| **Edition** | Kong Gateway (Free / Enterprise Trial) |
| **Mode** | Database mode (PostgreSQL) |
| **Admin API** | Port 8001 |
| **Proxy** | Port 8000 |
| **Kong Manager** | Port 8002 |

---

## Available Components (Default Workspace)

| Component | Purpose | AWS API Gateway Equivalent |
|-----------|---------|---------------------------|
| **Gateway Services** | Define upstream backends | Integration (backend URL) |
| **Routes** | Define paths/methods to match | Resources & Methods |
| **Consumers** | Identify API clients | API Keys (client identity) |
| **Plugins** | Add functionality | Features (auth, throttling) |
| **Upstreams** | Load balancing config | N/A (built-in) |
| **Certificates** | TLS/SSL certs | Custom domain certs |
| **SNIs** | Server Name Indication | Custom domain routing |
| **Keys** | Key management | N/A |
| **Vaults** | Secret management | Secrets Manager integration |

---

## Recommended Plugins for Your POC

Based on your requirements and the available plugins list, here are the **essential plugins** for feature parity with AWS API Gateway:

### 1. Authentication

| Plugin | Purpose | AWS Equivalent | Recommended |
|--------|---------|----------------|-------------|
| **Key Authentication** | API key validation | API Keys + Usage Plans | ✅ **YES** |
| JWT | JWT token validation | Cognito JWT | Optional |
| Basic Authentication | Username/password | N/A | No |
| OAuth 2.0 Authentication | OAuth flows | Cognito | Optional |

### 2. Traffic Control

| Plugin | Purpose | AWS Equivalent | Recommended |
|--------|---------|----------------|-------------|
| **Rate Limiting** | Request rate limits | Usage Plan throttling | ✅ **YES** |
| **ACL** | Access control lists | Resource Policy groups | ✅ **YES** |
| **Request Size Limiting** | Limit payload size | N/A | Optional |
| Proxy Caching | Response caching | API Gateway caching | Optional |
| Request Termination | Block requests | N/A | Optional |

### 3. Security

| Plugin | Purpose | AWS Equivalent | Recommended |
|--------|---------|----------------|-------------|
| **CORS** | Cross-origin requests | CORS configuration | ✅ **YES** (for browser clients) |
| **IP Restriction** | IP whitelist/blacklist | Resource Policy (IP) | ✅ **YES** |
| Bot Detection | Block bots | N/A | Optional |
| Request Validator | Validate requests | Request validation | Optional |

### 4. Serverless

| Plugin | Purpose | AWS Equivalent | Recommended |
|--------|---------|----------------|-------------|
| **AWS Lambda** | Invoke Lambda functions | Lambda integration | ✅ **YES** (if using Lambda) |

### 5. Logging

| Plugin | Purpose | AWS Equivalent | Recommended |
|--------|---------|----------------|-------------|
| **File Log** | Log to file | CloudWatch Logs | ✅ **YES** |
| **HTTP Log** | Log to HTTP endpoint | CloudWatch Logs | Optional |
| TCP Log | Log to TCP server | N/A | Optional |

### 6. Analytics & Monitoring

| Plugin | Purpose | AWS Equivalent | Recommended |
|--------|---------|----------------|-------------|
| **Prometheus** | Export metrics | CloudWatch Metrics | ✅ **YES** |
| Datadog | Datadog integration | N/A | Optional |
| Zipkin | Distributed tracing | X-Ray | Optional |

### 7. Transformations

| Plugin | Purpose | AWS Equivalent | Recommended |
|--------|---------|----------------|-------------|
| **Request Transformer** | Modify requests | Mapping templates | Optional |
| **Response Transformer** | Modify responses | Mapping templates | Optional |
| Correlation ID | Add request IDs | Request ID header | ✅ **YES** |

---

## Minimum POC Plugin Set

For a basic POC matching AWS API Gateway features:

```
┌─────────────────────────────────────────────────────────────┐
│                 MINIMUM POC PLUGINS                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Key Authentication  ─→ API Key validation               │
│  2. Rate Limiting       ─→ Throttling per key/route         │
│  3. ACL                 ─→ Group-based access control        │
│  4. CORS                ─→ Browser client support            │
│  5. File Log            ─→ Access logging                    │
│  6. Prometheus          ─→ Metrics                           │
│                                                              │
│  Optional:                                                   │
│  7. AWS Lambda          ─→ If calling Lambda functions       │
│  8. IP Restriction      ─→ If limiting by IP                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Plugin Configuration Order

When configuring, follow this order:

1. **First**: Create Services (backend URLs)
2. **Second**: Create Routes (paths/methods)
3. **Third**: Create Consumers (API clients)
4. **Fourth**: Enable Plugins (in this order):
   - CORS (if browser clients)
   - Key Authentication
   - ACL
   - Rate Limiting
   - Logging (File Log)
   - Prometheus

---

## Configuration via Kong Manager UI

### Navigation Path

```
Kong Manager (http://localhost:8002)
│
├── Gateway Services    → Define backends
│   └── [Create Service] → Name, URL
│
├── Routes              → Define API paths
│   └── [Create Route]  → Paths, Methods, Service
│
├── Consumers           → Define API clients
│   └── [Create Consumer] → Username
│       └── Credentials → API Keys
│       └── Groups      → ACL groups
│
└── Plugins             → Add features
    └── [New Plugin]    → Select, Configure
```

---

## Next Steps

Share the actual plugin configurations you see in the UI for these plugins, and I'll help you configure them correctly:

1. **Key Authentication** - For API key validation
2. **Rate Limiting** - For throttling
3. **ACL** - For access control
4. **CORS** - For browser support (if needed)
5. **File Log** - For logging
6. **AWS Lambda** - (if you need Lambda integration)

---

## Docker Compose Notes

Your docker-compose.yaml is correct for Kong Gateway 3.13.0.0. Key points:

| Setting | Value | Notes |
|---------|-------|-------|
| `KONG_PASSWORD` | handyshake | Kong Manager login password |
| `KONG_DATABASE` | postgres | Using database mode ✅ |
| Admin API | 8001 | For API configuration |
| Kong Manager | 8002 | Web UI |
| Proxy | 8000 | Where APIs are accessed |

**Default login:**
- URL: http://localhost:8002
- Password: `handyshake` (as set in compose)

---

**Please share the configuration screens for the plugins you want to enable, and I'll provide the exact settings for your POC.**
