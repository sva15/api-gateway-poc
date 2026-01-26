# Kong Lambda Integration

How to call AWS Lambda functions from Kong API Gateway.

---

## Lambda Integration Options

| Option | Complexity | Latency | Notes |
|--------|------------|---------|-------|
| **AWS Lambda Plugin** | Low | Medium | Direct invocation |
| **Lambda via API Gateway** | Medium | Higher | Use existing AWS APIGW |
| **Lambda via HTTP (Function URL)** | Low | Low | Lambda function URLs |
| **Lambda via ALB** | Medium | Low | Existing Lambda target groups |

---

## Option 1: AWS Lambda Plugin (Recommended)

Kong can invoke Lambda directly using the `aws-lambda` plugin.

### Prerequisites

- Kong installed
- AWS credentials configured on EC2 (IAM role or env vars)
- Lambda function ARN

### Configuration

```yaml
_format_version: "3.0"

services:
  - name: lambda-service
    url: http://localhost  # Placeholder, plugin overrides
    routes:
      - name: lambda-route
        paths:
          - /dev/api/lambda-function
        methods:
          - GET
          - POST
        plugins:
          - name: aws-lambda
            config:
              aws_region: ap-south-1
              function_name: my-lambda-function
              invocation_type: RequestResponse
              log_type: Tail
              timeout: 30000
              forward_request_body: true
              forward_request_headers: true
              forward_request_method: true
              forward_request_uri: true
```

### Admin API Configuration

```bash
# Create service (placeholder)
curl -X POST http://localhost:8001/services \
  -d name=lambda-service \
  -d url=http://localhost

# Create route
curl -X POST http://localhost:8001/services/lambda-service/routes \
  -d name=lambda-route \
  -d "paths[]=/dev/api/lambda"

# Enable Lambda plugin
curl -X POST http://localhost:8001/routes/lambda-route/plugins \
  -d name=aws-lambda \
  -d config.aws_region=ap-south-1 \
  -d config.function_name=my-lambda-function \
  -d config.forward_request_body=true \
  -d config.forward_request_headers=true
```

### IAM Permissions for EC2

Attach this policy to EC2 instance role:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction"
            ],
            "Resource": [
                "arn:aws:lambda:ap-south-1:ACCOUNT:function:my-lambda-*"
            ]
        }
    ]
}
```

### Lambda Response Format

Lambda should return:

```python
def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "body": json.dumps({"message": "success"})
    }
```

---

## Option 2: Lambda Function URLs

If your Lambda has a function URL (publicly accessible HTTPS endpoint).

### Configuration

```yaml
_format_version: "3.0"

services:
  - name: lambda-url-service
    url: https://abcd1234.lambda-url.ap-south-1.on.aws
    routes:
      - name: lambda-url-route
        paths:
          - /dev/api/lambda-func
        strip_path: true

plugins:
  - name: request-transformer
    service: lambda-url-service
    config:
      add:
        headers:
          - "x-forwarded-path:$(uri)"
```

### Pros and Cons

| Pros | Cons |
|------|------|
| ✅ Simple HTTP call | ❌ Lambda URL must be public |
| ✅ No plugin needed | ❌ Additional security concerns |
| ✅ Low latency | ❌ IAM auth requires signing |

---

## Option 3: Lambda via Existing ALB Target Group

If Lambdas are already behind ALB, Kong can call ALB.

### Configuration

```yaml
_format_version: "3.0"

services:
  - name: lambda-alb-service
    url: http://internal-alb-lambda-12345.ap-south-1.elb.amazonaws.com
    routes:
      - name: lambda-alb-route
        paths:
          - /dev/api/lambda-alb
        strip_path: false
```

### Pros and Cons

| Pros | Cons |
|------|------|
| ✅ Uses existing infrastructure | ❌ Additional hop (ALB) |
| ✅ ALB handles Lambda integration | ❌ Slightly higher latency |
| ✅ No Kong plugins needed | |

---

## Comparison: Kong → Lambda vs AWS APIGW → Lambda

| Aspect | Kong + Lambda Plugin | AWS API Gateway |
|--------|---------------------|-----------------|
| Setup complexity | Medium (IAM config) | Low (native) |
| Latency | ~50-100ms overhead | ~10-30ms overhead |
| Permissions | Manual IAM | Automatic |
| Cold starts | Same | Same |
| Response mapping | Manual in Lambda | Mapping templates |
| Cost | Kong free + Lambda | APIGW + Lambda |

---

## Event Format Comparison

### AWS API Gateway Event

```json
{
  "httpMethod": "POST",
  "path": "/users",
  "headers": {...},
  "queryStringParameters": {...},
  "body": "{...}",
  "requestContext": {...}
}
```

### Kong aws-lambda Plugin Event

```json
{
  "request_method": "POST",
  "request_uri": "/dev/api/lambda/users",
  "request_headers": {...},
  "request_body": "{...}"
}
```

### Universal Lambda Handler

```python
import json

def lambda_handler(event, context):
    # Handle both Kong and APIGW events
    if 'httpMethod' in event:
        # AWS API Gateway format
        method = event['httpMethod']
        path = event['path']
        body = event.get('body', '')
    elif 'request_method' in event:
        # Kong aws-lambda plugin format
        method = event['request_method']
        path = event['request_uri']
        body = event.get('request_body', '')
    else:
        # Direct invocation
        method = 'POST'
        path = '/'
        body = json.dumps(event)
    
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({
            "source": "lambda",
            "method": method,
            "path": path
        })
    }
```

---

## Recommended Pattern for Your Architecture

Given your existing setup:

```
┌────────────────────────────────────────────────────────────────┐
│                        ALB                                      │
│                                                                 │
│  /dev/api/*  →  Kong Target Group  →  Kong                     │
│                                         │                       │
│                                         ├──→ Lambda (plugin)    │
│                                         ├──→ EC2 backends       │
│                                         └──→ Other services     │
│                                                                 │
│  /dev/lambda/* →  Lambda Target Group  →  Lambda (existing)    │
│  (Keep existing Lambda routes if needed)                        │
└────────────────────────────────────────────────────────────────┘
```

### Recommendation

1. **New Lambda integrations**: Use Kong aws-lambda plugin
2. **Existing Lambda routes**: Keep via ALB target group
3. **Gradual migration**: Move Lambda routes to Kong over time

---

**← Previous:** [05-security.md](05-security.md) | **Next:** [07-operations.md](07-operations.md) →
