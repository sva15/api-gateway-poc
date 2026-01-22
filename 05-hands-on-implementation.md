# Hands-On Implementation

This section provides **step-by-step instructions** to build the POC using AWS Console.

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
                    │  /dev                        │
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
                    │                  │       │                  │
                    │  Handles:        │       │  Handles:        │
                    │  /dev/api2/*     │       │  /dev/api1/*     │
                    └──────────────────┘       └──────────────────┘
```

---

## Implementation Steps

### Step 1: Create Lambda Functions

#### 1.1 Create api1-lambda

**Navigate to Lambda Console:**
1. Go to AWS Console → Lambda
2. Click **Create function**

**Configuration:**

| Setting | Value |
|---------|-------|
| Function name | `api1-lambda` |
| Runtime | Python 3.12 |
| Architecture | x86_64 |
| Execution role | Create a new role with basic Lambda permissions |

**Function Code:**

```python
import json

def lambda_handler(event, context):
    """
    Handler for /dev/api1/{proxy+} requests
    Demonstrates how path parameters are received from API Gateway
    """
    
    # Extract path information
    path = event.get('path', 'unknown')
    http_method = event.get('httpMethod', 'unknown')
    
    # Get the proxy path parameter (everything after /dev/api1/)
    path_parameters = event.get('pathParameters', {})
    proxy_path = path_parameters.get('proxy', '') if path_parameters else ''
    
    # Get query string parameters
    query_params = event.get('queryStringParameters', {})
    
    # Get request body
    body = event.get('body', None)
    
    # Build response
    response_body = {
        "service": "api1-lambda",
        "message": "Request handled by api1-lambda",
        "request_details": {
            "full_path": path,
            "http_method": http_method,
            "proxy_path": proxy_path,
            "query_parameters": query_params,
            "body": body
        }
    }
    
    # Log for debugging (visible in CloudWatch)
    print(f"Received request: {json.dumps(event, indent=2)}")
    
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "body": json.dumps(response_body, indent=2)
    }
```

3. Click **Deploy** after pasting the code

---

#### 1.2 Create api2-lambda

**Repeat the same process with:**

| Setting | Value |
|---------|-------|
| Function name | `api2-lambda` |
| Runtime | Python 3.12 |
| Architecture | x86_64 |
| Execution role | Create a new role with basic Lambda permissions |

**Function Code:**

```python
import json

def lambda_handler(event, context):
    """
    Handler for /dev/api2/{proxy+} requests
    Demonstrates how path parameters are received from API Gateway
    """
    
    # Extract path information
    path = event.get('path', 'unknown')
    http_method = event.get('httpMethod', 'unknown')
    
    # Get the proxy path parameter (everything after /dev/api2/)
    path_parameters = event.get('pathParameters', {})
    proxy_path = path_parameters.get('proxy', '') if path_parameters else ''
    
    # Get query string parameters
    query_params = event.get('queryStringParameters', {})
    
    # Get request body
    body = event.get('body', None)
    
    # Build response
    response_body = {
        "service": "api2-lambda",
        "message": "Request handled by api2-lambda",
        "request_details": {
            "full_path": path,
            "http_method": http_method,
            "proxy_path": proxy_path,
            "query_parameters": query_params,
            "body": body
        }
    }
    
    # Log for debugging (visible in CloudWatch)
    print(f"Received request: {json.dumps(event, indent=2)}")
    
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "body": json.dumps(response_body, indent=2)
    }
```

---

### Step 2: Create VPC Endpoint for API Gateway

> **Note:** Skip this step if you're using an existing VPC endpoint. Ensure Private DNS is enabled.

**Navigate to VPC Console:**
1. Go to AWS Console → VPC → Endpoints
2. Click **Create endpoint**

**Configuration:**

| Setting | Value |
|---------|-------|
| Name tag | `api-gateway-endpoint` |
| Service category | AWS services |
| Service | `com.amazonaws.<your-region>.execute-api` |
| VPC | Select your VPC |
| Subnets | Select private subnets (at least 2, different AZs) |
| Security groups | Select/create SG allowing HTTPS (443) inbound from VPC CIDR |
| DNS name type | Enable private DNS name ✅ |

**Security Group Rules:**

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| HTTPS | TCP | 443 | VPC CIDR (e.g., 10.0.0.0/16) | Allow API Gateway access |

3. Click **Create endpoint**
4. Wait for status to become **Available**

**Record the VPC Endpoint ID:** `vpce-xxxxxxxxxxxxxxxxx` (needed for resource policy)

---

### Step 3: Create Private REST API

**Navigate to API Gateway Console:**
1. Go to AWS Console → API Gateway
2. Click **Create API**
3. Under REST API, click **Build** (NOT REST API Private - we'll configure that)

**Configuration:**

| Setting | Value |
|---------|-------|
| API name | `internal-api-poc` |
| API endpoint type | **Private** |
| VPC Endpoint IDs | Enter your VPC Endpoint ID (vpce-xxx) |

4. Click **Create API**

---

### Step 4: Configure Resource Policy

**In the API Gateway Console:**
1. Select your API (`internal-api-poc`)
2. In the left sidebar, click **Resource Policy**
3. Enter the following policy (replace placeholders):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:<REGION>:<ACCOUNT_ID>:<API_ID>/*",
            "Condition": {
                "StringNotEquals": {
                    "aws:sourceVpce": "<YOUR_VPC_ENDPOINT_ID>"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:<REGION>:<ACCOUNT_ID>:<API_ID>/*"
        }
    ]
}
```

**Replace:**
- `<REGION>`: Your AWS region (e.g., `ap-south-1`)
- `<ACCOUNT_ID>`: Your AWS account ID
- `<API_ID>`: The API ID (visible in the console URL or API settings)
- `<YOUR_VPC_ENDPOINT_ID>`: The VPC endpoint ID from Step 2

4. Click **Save**

---

### Step 5: Create API Resources

**Create Resource Structure:**

```
/dev
├── /api1
│   └── /{proxy+}
└── /api2
    └── /{proxy+}
```

#### 5.1 Create `/dev` Resource

1. In the API, select the root resource `/`
2. Click **Create Resource**
3. Configure:
   - Resource Name: `dev`
   - Resource Path: `dev`
4. Click **Create Resource**

#### 5.2 Create `/api1` Resource

1. Select the `/dev` resource
2. Click **Create Resource**
3. Configure:
   - Resource Name: `api1`
   - Resource Path: `api1`
4. Click **Create Resource**

#### 5.3 Create `/{proxy+}` under `/api1`

1. Select the `/dev/api1` resource
2. Click **Create Resource**
3. Configure:
   - **Configure as proxy resource:** ✅ Check this box
   - Resource Name: `proxy`
   - Resource Path: `{proxy+}`
4. Click **Create Resource**

#### 5.4 Create `/api2` and its `/{proxy+}`

Repeat steps 5.2 and 5.3 for `/api2`:
1. Select `/dev` → Create Resource → `api2`
2. Select `/dev/api2` → Create Resource → Enable proxy → `{proxy+}`

---

### Step 6: Configure Integrations

#### 6.1 Configure `/dev/api1/{proxy+}` Integration

1. Select the `/{proxy+}` resource under `/dev/api1`
2. Select the **ANY** method
3. Click **Edit integration**
4. Configure:

| Setting | Value |
|---------|-------|
| Integration type | Lambda Function |
| Lambda proxy integration | ✅ Enabled |
| Lambda function | `api1-lambda` |

5. Click **Save**
6. Click **OK** when prompted to add permission

#### 6.2 Configure `/dev/api2/{proxy+}` Integration

1. Select the `/{proxy+}` resource under `/dev/api2`
2. Select the **ANY** method
3. Click **Edit integration**
4. Configure:

| Setting | Value |
|---------|-------|
| Integration type | Lambda Function |
| Lambda proxy integration | ✅ Enabled |
| Lambda function | `api2-lambda` |

5. Click **Save**
6. Click **OK** when prompted to add permission

---

### Step 7: Deploy the API

1. Click **Deploy API**
2. Configure:

| Setting | Value |
|---------|-------|
| Stage | **New Stage** |
| Stage name | `dev` |
| Stage description | Development stage |

3. Click **Deploy**

**Record the Invoke URL:** 
```
https://<api-id>.execute-api.<region>.amazonaws.com/dev
```

---

### Step 8: Test the API

#### Testing from EC2 Instance in VPC

1. SSH into an EC2 instance in your VPC (in a private subnet associated with the VPC endpoint)
2. Run the following tests:

**Test 1: Basic request to api1**
```bash
curl https://<api-id>.execute-api.<region>.amazonaws.com/dev/api1/test
```

**Expected Response:**
```json
{
  "service": "api1-lambda",
  "message": "Request handled by api1-lambda",
  "request_details": {
    "full_path": "/dev/api1/test",
    "http_method": "GET",
    "proxy_path": "test",
    "query_parameters": null,
    "body": null
  }
}
```

**Test 2: Nested path**
```bash
curl https://<api-id>.execute-api.<region>.amazonaws.com/dev/api1/foo/bar
```

**Expected `proxy_path`:** `foo/bar`

**Test 3: api2 endpoint**
```bash
curl https://<api-id>.execute-api.<region>.amazonaws.com/dev/api2/hello
```

**Expected Response:**
```json
{
  "service": "api2-lambda",
  "message": "Request handled by api2-lambda",
  "request_details": {
    "full_path": "/dev/api2/hello",
    "http_method": "GET",
    "proxy_path": "hello",
    "query_parameters": null,
    "body": null
  }
}
```

**Test 4: Deeply nested path**
```bash
curl https://<api-id>.execute-api.<region>.amazonaws.com/dev/api2/v1/users/123/orders
```

**Expected `proxy_path`:** `v1/users/123/orders`

---

## Understanding the Lambda Event

When API Gateway forwards a request to Lambda with proxy integration, the event looks like:

```json
{
  "resource": "/dev/api1/{proxy+}",
  "path": "/dev/api1/users/123",
  "httpMethod": "GET",
  "headers": {
    "Host": "xxxxx.execute-api.region.amazonaws.com",
    "User-Agent": "curl/7.68.0",
    "Accept": "*/*"
  },
  "queryStringParameters": {
    "status": "active",
    "limit": "10"
  },
  "pathParameters": {
    "proxy": "users/123"
  },
  "stageVariables": null,
  "requestContext": {
    "resourcePath": "/dev/api1/{proxy+}",
    "httpMethod": "GET",
    "path": "/dev/dev/api1/users/123",
    "stage": "dev",
    "domainName": "xxxxx.execute-api.region.amazonaws.com",
    "apiId": "xxxxx"
  },
  "body": null,
  "isBase64Encoded": false
}
```

### Key Fields

| Field | Description | Example Value |
|-------|-------------|---------------|
| `path` | Full request path | `/dev/api1/users/123` |
| `pathParameters.proxy` | Captured by `{proxy+}` | `users/123` |
| `httpMethod` | HTTP method | `GET`, `POST`, `PUT`, `DELETE` |
| `queryStringParameters` | Query params as object | `{"status": "active"}` |
| `body` | Request body (string) | `"{"name": "John"}"` |
| `headers` | Request headers | `{"Content-Type": "application/json"}` |

---

## How This Replaces ALB Routing

### Comparison

| ALB Configuration | API Gateway Equivalent |
|-------------------|----------------------|
| Rule: `/dev/api1/*` → TG1 | `/dev/api1/{proxy+}` → api1-lambda |
| Rule: `/dev/api2/*` → TG2 | `/dev/api2/{proxy+}` → api2-lambda |

### Key Differences

| Aspect | ALB | API Gateway |
|--------|-----|-------------|
| **Captured Path** | Full path in `event.path` | Full path + `proxy` parameter |
| **Path Variable** | Must parse from path | Available in `pathParameters.proxy` |
| **Method Routing** | All methods to same target | Can configure per-method handlers |
| **Base Path** | Included in path | Included in path (stage adds prefix) |

---

## Limitations of This Approach

```
┌─────────────────────────────────────────────────────────────────┐
│               LIMITATIONS OF {proxy+} APPROACH                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. No Gateway-Level Validation                                  │
│     └── All paths accepted, backend must validate               │
│     └── Invalid paths reach Lambda (cost + latency)             │
│                                                                  │
│  2. Same Auth for All Subpaths                                   │
│     └── Can't have /admin/* require IAM while /public/* is open│
│     └── Must implement in Lambda if needed                      │
│                                                                  │
│  3. Same Throttling for All Subpaths                            │
│     └── Rate limits apply to entire {proxy+} resource           │
│     └── Can't rate-limit /write/* differently from /read/*      │
│                                                                  │
│  4. Caching Less Effective                                       │
│     └── Cache key must include entire path                      │
│     └── No per-endpoint cache configuration                     │
│                                                                  │
│  5. OpenAPI Documentation                                        │
│     └── Generates generic docs for {proxy+}                     │
│     └── Doesn't reflect actual endpoints                        │
│                                                                  │
│  6. 404 Handling                                                 │
│     └── Lambda must return 404 for invalid paths                │
│     └── API Gateway can't distinguish valid from invalid        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Verification Checklist

```
□ Lambda Functions
  ├── □ api1-lambda created and deployed
  ├── □ api2-lambda created and deployed
  └── □ Both have CloudWatch Logs permissions

□ VPC Endpoint
  ├── □ Created in correct VPC
  ├── □ Private DNS enabled
  ├── □ Security group allows HTTPS (443)
  └── □ Status is "Available"

□ API Gateway
  ├── □ Private REST API created
  ├── □ Resource policy configured with VPC endpoint
  ├── □ Resources created (/dev/api1/{proxy+}, /dev/api2/{proxy+})
  ├── □ Integrations configured (Lambda proxy)
  └── □ Deployed to "dev" stage

□ Testing
  ├── □ /dev/api1/test returns api1-lambda response
  ├── □ /dev/api1/foo/bar returns proxy_path = "foo/bar"
  ├── □ /dev/api2/hello returns api2-lambda response
  └── □ Nested paths work correctly
```

---

**← Previous:** [04-routing.md](04-routing.md) | **Next:** [06-security.md](06-security.md) →
