# Hands-On Implementation

This section provides **step-by-step instructions** to build the POC using AWS Console (updated for 2026).

---

## Critical Warnings Before You Start

> ⚠️ **WARNING 1: Stage vs Resource Path**
> 
> Do NOT create a `/dev` resource if your stage is named `dev`. API Gateway automatically adds the stage name to the URL.
> - ❌ Resource `/dev` + Stage `dev` = `/dev/dev/api1/...`
> - ✅ Resource `/api1` + Stage `dev` = `/dev/api1/...`

> ⚠️ **WARNING 2: Resource Policy Before Deploy**
> 
> For Private REST APIs, you MUST configure the resource policy BEFORE first deployment, otherwise you'll get 403 errors.

> ⚠️ **WARNING 3: VPC Endpoint Requirements**
> 
> If subnets are not visible when creating VPC endpoint:
> - Enable **DNS hostnames** on VPC
> - Enable **DNS resolution** on VPC
> - Ensure subnets have available IP addresses
> - Use default VPC if custom VPC doesn't work

---

## Target Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                    VPC                                        │
│                                                                               │
│   ┌─────────────────────┐                                                     │
│   │      Client         │                                                     │
│   │   (EC2 Instance)    │                                                     │
│   └──────────┬──────────┘                                                     │
│              │                                                                │
│              │ HTTPS Request                                                  │
│              ▼                                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐    │
│   │                    VPC Endpoint (execute-api)                        │    │
│   │                    Private DNS Enabled                               │    │
│   └──────────────────────────────┬──────────────────────────────────────┘    │
│                                  │                                            │
└──────────────────────────────────┼────────────────────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │       API Gateway             │
                    │    Private REST API          │
                    │                              │
                    │  Stage: dev                  │
                    │  ├── /api1/{proxy+}          │───────┐
                    │  │   └── ANY                 │       │
                    │  └── /api2/{proxy+}          │───┐   │
                    │      └── ANY                 │   │   │
                    └──────────────────────────────┘   │   │
                                                       │   │
                               ┌───────────────────────┘   │
                               │                           │
                               ▼                           ▼
                    ┌──────────────────┐       ┌──────────────────┐
                    │   api2-lambda    │       │   api1-lambda    │
                    └──────────────────┘       └──────────────────┘
```

**Final URL Structure:**
```
https://{api-id}.execute-api.{region}.amazonaws.com/dev/api1/test
                                                    ↑    ↑
                                               Stage  Resource
```

---

## Step 1: Create Lambda Functions

### 1.1 Create api1-lambda

1. Go to AWS Console → Lambda → **Create function**
2. Configure:
   - Function name: `api1-lambda`
   - Runtime: Python 3.12
   - Architecture: x86_64
   - Permissions: Create new role with basic Lambda permissions

3. Replace code with:

```python
import json

def lambda_handler(event, context):
    path = event.get('path', 'unknown')
    http_method = event.get('httpMethod', 'unknown')
    path_parameters = event.get('pathParameters', {})
    proxy_path = path_parameters.get('proxy', '') if path_parameters else ''
    
    response_body = {
        "service": "api1-lambda",
        "message": "Request handled by api1-lambda",
        "request_details": {
            "full_path": path,
            "http_method": http_method,
            "proxy_path": proxy_path,
            "query_parameters": event.get('queryStringParameters', {}),
            "body": event.get('body', None)
        }
    }
    
    print(f"Received request: {json.dumps(event, indent=2)}")
    
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps(response_body, indent=2)
    }
```

4. Click **Deploy**

### 1.2 Create api2-lambda

Repeat with function name `api2-lambda` and change `"service": "api2-lambda"` in the code.

---

## Step 2: Create VPC Endpoint for API Gateway

### Pre-requisites Check

Before creating the endpoint, verify your VPC settings:

1. Go to VPC Console → Your VPC → **Edit VPC settings**
2. Ensure both are enabled:
   - ✅ DNS hostnames
   - ✅ DNS resolution

### Create the Endpoint

1. VPC Console → Endpoints → **Create endpoint**
2. Configure:

| Setting | Value |
|---------|-------|
| Name | `api-gateway-endpoint` |
| Service category | AWS services |
| Services | Search for `execute-api`, select `com.amazonaws.{region}.execute-api` |
| VPC | Select your VPC |
| Subnets | Select subnets (at least 2 in different AZs) |
| Security groups | Create/select SG with HTTPS (443) inbound from VPC CIDR |
| Policy | Full access |

3. **Enable DNS name** section: Ensure "Enable DNS name" is checked (this is Private DNS)
4. Click **Create endpoint**
5. Wait for status: **Available**
6. **Copy the VPC Endpoint ID** (e.g., `vpce-0abc123def456789`)

> ⚠️ **If subnets don't appear:** Use the default VPC, or ensure DNS settings are enabled on your VPC.

---

## Step 3: Create Private REST API

1. API Gateway Console → **Create API**
2. Under "REST API", click **Build**
3. Configure:

| Setting | Value |
|---------|-------|
| API name | `internal-api-poc` |
| API description | Internal API Gateway POC |
| API endpoint type | **Private** |

4. Under "VPC endpoint IDs", enter your VPC Endpoint ID
5. Click **Create API**

---

## Step 4: Configure Resource Policy (BEFORE Creating Resources)

> ⚠️ **CRITICAL:** Configure this BEFORE deploying the API!

1. In the API, go to **Resource policy** (left sidebar)
2. Enter this policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:REGION:ACCOUNT_ID:API_ID/*",
            "Condition": {
                "StringEquals": {
                    "aws:SourceVpce": "vpce-YOUR_ENDPOINT_ID"
                }
            }
        }
    ]
}
```

**Replace:**
- `REGION`: Your region (e.g., `ap-south-1`)
- `ACCOUNT_ID`: Your AWS account ID (12 digits)
- `API_ID`: The API ID (from URL or API details)
- `vpce-YOUR_ENDPOINT_ID`: Your VPC endpoint ID

3. Click **Save**

> **Note:** Use `aws:SourceVpce` NOT `aws:SourceVpc`. Private APIs route through the VPC endpoint.

---

## Step 5: Create API Resources

### Correct Structure (No /dev resource!)

```
/                      ← Root (already exists)
├── /api1              ← Resource
│   └── /{proxy+}      ← Proxy resource
└── /api2              ← Resource
    └── /{proxy+}      ← Proxy resource

Stage: dev             ← Applied at deployment
```

### 5.1 Create `/api1` Resource

1. Select the root `/` resource
2. Click **Create resource**
3. Configure:
   - Resource name: `api1`
   - Resource path: `api1`
   - CORS: Leave unchecked (for now)
4. Click **Create resource**

### 5.2 Create `/{proxy+}` under `/api1`

1. Select `/api1` resource
2. Click **Create resource**
3. Configure:
   - ✅ **Proxy resource** (enable this checkbox)
   - Resource name: `{proxy+}`
   - Resource path: `{proxy+}`
4. Click **Create resource**

### 5.3 Create `/api2` and `/{proxy+}`

Repeat:
1. Select `/` → Create resource → `api2`
2. Select `/api2` → Create resource → Enable proxy → `{proxy+}`

---

## Step 6: Configure Lambda Integrations

### 6.1 Configure `/api1/{proxy+}`

1. Select `/{proxy+}` under `/api1`
2. The **ANY** method should be visible
3. Click on **ANY** → **Integration request** → **Edit**
4. Configure:
   - Integration type: **Lambda function**
   - ✅ Lambda proxy integration
   - Lambda function: `api1-lambda`
5. Click **Save**
6. If prompted to add permission, click **OK**

### 6.2 Configure `/api2/{proxy+}`

Repeat for `/api2/{proxy+}` → integrate with `api2-lambda`

---

## Step 7: Deploy the API

1. Click **Deploy API** button
2. Select:
   - Stage: **New stage**
   - Stage name: `dev`
3. Click **Deploy**

**Your Invoke URL:**
```
https://{api-id}.execute-api.{region}.amazonaws.com/dev
```

---

## Step 8: Test the API

### From EC2 in the VPC

SSH into an EC2 instance in your VPC and test:

```bash
# Test api1
curl https://{api-id}.execute-api.{region}.amazonaws.com/dev/api1/test

# Test nested path
curl https://{api-id}.execute-api.{region}.amazonaws.com/dev/api1/users/123

# Test api2
curl https://{api-id}.execute-api.{region}.amazonaws.com/dev/api2/hello
```

### Validate Private Routing

```bash
# Should resolve to private IPs
nslookup {api-id}.execute-api.{region}.amazonaws.com
```

If it returns private IPs (10.x.x.x or 172.x.x.x), traffic is correctly routed through the VPC endpoint.

---

## Common Issues During Implementation

| Issue | Cause | Solution |
|-------|-------|----------|
| 403 "explicit deny" | Wrong condition in resource policy | Use `aws:SourceVpce` not `aws:SourceVpc` |
| 403 after deployment | Resource policy not configured | Add resource policy, redeploy |
| URL shows `/dev/dev/...` | Created `/dev` resource | Delete `/dev` resource, use stage only |
| Subnets not visible | VPC DNS not enabled | Enable DNS hostnames and resolution |
| API key not enforced | No usage plan | Create usage plan and associate |

See [08-troubleshooting.md](08-troubleshooting.md) for detailed solutions.

---

## Verification Checklist

```
□ VPC Endpoint
  ├── □ DNS hostnames enabled on VPC
  ├── □ DNS resolution enabled on VPC
  ├── □ Private DNS enabled on endpoint
  ├── □ Status: Available
  └── □ Endpoint ID recorded

□ API Gateway
  ├── □ Private REST API created
  ├── □ Resource policy configured (BEFORE deploy)
  ├── □ Using aws:SourceVpce condition
  ├── □ NO /dev resource (only /api1, /api2)
  ├── □ {proxy+} resources created
  ├── □ Lambda integrations configured
  └── □ Deployed to "dev" stage

□ Testing
  ├── □ DNS resolves to private IPs
  ├── □ /dev/api1/test returns api1-lambda response
  └── □ /dev/api2/test returns api2-lambda response
```

---

**← Previous:** [04-routing.md](04-routing.md) | **Next:** [06-security.md](06-security.md) →
