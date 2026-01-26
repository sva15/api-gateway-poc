# Kong Operations - Logging, Metrics, Maintenance

This document covers operational features for running Kong in production.

---

## 1. Logging

### Access Logging (Enabled by Default)

Kong logs to stdout/stderr by default. Configure in docker-compose:

```yaml
environment:
  KONG_PROXY_ACCESS_LOG: /dev/stdout
  KONG_PROXY_ERROR_LOG: /dev/stderr
```

### File Logging Plugin

```yaml
plugins:
  - name: file-log
    config:
      path: /var/log/kong/access.log
      reopen: true
```

### HTTP Log Plugin (Send to External Service)

```yaml
plugins:
  - name: http-log
    config:
      http_endpoint: http://log-collector:8080/logs
      method: POST
      content_type: application/json
      timeout: 10000
      keepalive: 60000
```

### CloudWatch Logs Integration

Use the AWS CloudWatch Logs plugin or ship logs via Fluent Bit:

```yaml
# docker-compose.yml - add fluent-bit sidecar
services:
  fluent-bit:
    image: amazon/aws-for-fluent-bit:latest
    volumes:
      - ./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf
      - kong-logs:/var/log/kong
```

### Log Format

Default Kong log format:

```
client_ip - - [timestamp] "method path HTTP/1.1" status bytes "referrer" "user-agent" latency
```

Custom log format via plugins provides JSON:

```json
{
  "request": {
    "uri": "/dev/api/test",
    "method": "GET",
    "headers": {...}
  },
  "response": {
    "status": 200,
    "headers": {...}
  },
  "latencies": {
    "request": 15,
    "kong": 3,
    "proxy": 12
  },
  "client_ip": "10.0.1.50",
  "consumer": {
    "username": "client-app-1"
  }
}
```

---

## 2. Metrics and Monitoring

### Prometheus Plugin

```yaml
plugins:
  - name: prometheus
    config:
      per_consumer: true
      status_code_metrics: true
      latency_metrics: true
      bandwidth_metrics: true
```

Metrics available at: `http://localhost:8001/metrics`

### Key Metrics

| Metric | Description |
|--------|-------------|
| `kong_http_requests_total` | Total requests |
| `kong_request_latency_ms` | Request latency |
| `kong_upstream_latency_ms` | Backend latency |
| `kong_bandwidth_bytes` | Bandwidth usage |
| `kong_datastore_reachable` | DB connectivity |

### Grafana Dashboard

Kong provides official Grafana dashboards. Import dashboard ID: `7424`

### StatsD Plugin

```yaml
plugins:
  - name: statsd
    config:
      host: statsd-host
      port: 8125
      metrics:
        - request_count
        - latency
        - status_count
```

---

## 3. Health Checks

### Kong Status Endpoint

```bash
curl http://localhost:8001/status
```

Response:
```json
{
  "database": {
    "reachable": true
  },
  "memory": {
    "workers_lua_vms": [...],
    "lua_shared_dicts": {...}
  },
  "server": {
    "connections_accepted": 1234,
    "connections_active": 5,
    "connections_handled": 1234,
    "connections_reading": 0,
    "connections_waiting": 4,
    "connections_writing": 1,
    "total_requests": 5678
  }
}
```

### ALB Health Check

Configure ALB to check:
- Path: `/status`
- Port: 8000
- Expected: 200 OK

### Upstream Health Checks

```yaml
services:
  - name: backend-service
    url: http://backend:8080
    connect_timeout: 5000
    write_timeout: 5000
    read_timeout: 5000

upstreams:
  - name: backend-upstream
    slots: 10000
    healthchecks:
      active:
        healthy:
          interval: 5
          successes: 2
        unhealthy:
          interval: 5
          http_failures: 3
        http_path: /health
      passive:
        healthy:
          successes: 2
        unhealthy:
          http_failures: 3
```

---

## 4. Configuration Updates

### DB-less Mode

```bash
# Edit kong.yml
vim /opt/kong/kong.yml

# Reload config (no downtime)
curl -X POST http://localhost:8001/config \
  -F config=@/opt/kong/kong.yml

# Or restart container
docker-compose restart kong
```

### Database Mode

```bash
# Changes via Admin API take effect immediately
curl -X POST http://localhost:8001/services \
  -d name=new-service \
  -d url=http://new-backend:8080

# No restart needed
```

---

## 5. Backup and Recovery

### DB-less Mode

Backup is just the `kong.yml` file:

```bash
cp /opt/kong/kong.yml /backup/kong-$(date +%Y%m%d).yml
```

### Database Mode

```bash
# Backup database
docker exec kong-postgres pg_dump -U kong kong > backup.sql

# Export config
curl http://localhost:8001/services > services-backup.json
curl http://localhost:8001/routes > routes-backup.json
curl http://localhost:8001/plugins > plugins-backup.json
```

---

## 6. Scaling Kong

### Horizontal Scaling

Run multiple Kong instances behind ALB:

```
ALB → Kong Instance 1 (EC2-A)
    → Kong Instance 2 (EC2-B)
    → Kong Instance 3 (EC2-C)
```

Requirements:
- All instances share same database OR
- All instances use same kong.yml (DB-less)

### Vertical Scaling

Increase Kong workers:

```yaml
environment:
  KONG_NGINX_WORKER_PROCESSES: auto  # Uses CPU count
```

---

## 7. Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 502 Bad Gateway | Backend unreachable | Check backend health, security groups |
| 503 Service Unavailable | Kong overloaded | Scale up, check rate limits |
| Connection refused | Kong not running | Check container, ports |
| Config not loading | YAML syntax error | Validate: `kong config parse kong.yml` |

### Debug Commands

```bash
# Check Kong logs
docker logs -f kong-gateway

# Validate config
docker exec kong-gateway kong config parse /kong/declarative/kong.yml

# Check routes
curl http://localhost:8001/routes | jq .

# Check plugins
curl http://localhost:8001/plugins | jq .

# Test upstream
curl http://localhost:8001/upstreams/backend/health
```

---

## 8. Comparison with AWS API Gateway Operations

| Feature | AWS API Gateway | Kong |
|---------|-----------------|------|
| Logging | CloudWatch (automatic) | Plugin-based (manual setup) |
| Metrics | CloudWatch (automatic) | Prometheus/StatsD (manual) |
| Scaling | Automatic | Manual (add instances) |
| Updates | Stage deployment | Admin API / config reload |
| Backup | Export/Import | DB dump / YAML file |
| Monitoring | X-Ray tracing | Zipkin/Jaeger plugin |

---

## Operational Checklist

```
□ Logging
  ├── □ Access logs configured
  ├── □ Error logs configured
  └── □ Log shipping to central system

□ Monitoring
  ├── □ Prometheus plugin enabled
  ├── □ Grafana dashboard imported
  └── □ Alerts configured

□ Health Checks
  ├── □ ALB health check configured
  ├── □ Upstream health checks enabled
  └── □ Status endpoint accessible

□ Security
  ├── □ Admin API restricted (127.0.0.1)
  ├── □ HTTPS configured
  └── □ API keys rotated regularly

□ Backup
  ├── □ Config backup scheduled
  └── □ Database backup (if applicable)
```

---

**← Previous:** [06-lambda-integration.md](06-lambda-integration.md) | **Back to:** [00-overview.md](00-overview.md)
