# Private REST API Architecture

This section explains how Private REST APIs work internally, the role of each component, and common error causes.

---

## 1. Internal Request Flow

### Complete Request Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                    VPC                                       │
│                                                                              │
│  ┌──────────────────┐                                                        │
│  │      CLIENT      │                                                        │
│  │  (EC2/Lambda/    │                                                        │
│  │   Container)     │                                                        │
│  └────────┬─────────┘                                                        │
│           │                                                                  │
│           │ 1. HTTPS Request                                                 │
│           │    POST https://<api-id>.execute-api.<region>.amazonaws.com      │
│           │         /dev/api1/users/123                                      │
│           │                                                                  │
│           ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         VPC ENDPOINT                                  │   │
│  │                   (Interface Endpoint - ENI)                          │   │
│  │                                                                       │   │
│  │  • Private IPs in your subnets                                        │   │
│  │  • DNS resolves API Gateway URL to these IPs                          │   │
│  │  • Security Group controls access                                     │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│           │                                                                  │
└───────────┼──────────────────────────────────────────────────────────────────┘
            │
            │ 2. AWS PrivateLink (Private Network)
            │
            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           API GATEWAY SERVICE                                 │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                     RESOURCE POLICY CHECK                            │     │
│  │   3. Verify: Is request from allowed VPC endpoint?                   │     │
│  │              Is source account allowed?                              │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                               │                                               │
│                               │ ✅ Allowed                                    │
│                               ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                        ROUTE MATCHING                                │     │
│  │   4. Match URL path + HTTP method to resource                        │     │
│  │      /dev/api1/users/123 → /api1/{proxy+} (POST)                    │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                               │                                               │
│                               ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                     METHOD REQUEST                                   │     │
│  │   5. Validate request (if configured)                                │     │
│  │      - Check required headers                                        │     │
│  │      - Validate request body schema                                  │     │
│  │      - Verify API key (if required)                                  │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                               │                                               │
│                               ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                   INTEGRATION REQUEST                                │     │
│  │   6. Transform request for backend                                   │     │
│  │      - Map path parameters                                           │     │
│  │      - Add/modify headers                                            │     │
│  │      - Transform body (optional)                                     │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                               │                                               │
└───────────────────────────────┼───────────────────────────────────────────────┘
                                │
                                │ 7. Invoke Lambda
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                             LAMBDA FUNCTION                                   │
│                                                                               │
│  8. Execute function code                                                     │
│     - Receive event with path, body, headers                                  │
│     - Process request                                                         │
│     - Return response                                                         │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
                                │
                                │ 9. Response
                                ▼
                    (Reverse flow back to client)
```

---

## 2. Component Roles

### VPC Endpoint (`execute-api`)

```
┌────────────────────────────────────────────────────────────────────┐
│                         VPC ENDPOINT                                │
│                                                                     │
│  Type: Interface Endpoint (powered by AWS PrivateLink)             │
│  Service: com.amazonaws.<region>.execute-api                       │
│                                                                     │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐              │
│  │   Subnet A  │   │   Subnet B  │   │   Subnet C  │              │
│  │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │              │
│  │ │   ENI   │ │   │ │   ENI   │ │   │ │   ENI   │ │              │
│  │ │10.0.1.50│ │   │ │10.0.2.50│ │   │ │10.0.3.50│ │              │
│  │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │              │
│  └─────────────┘   └─────────────┘   └─────────────┘              │
│                                                                     │
│  Private DNS Enabled:                                               │
│  *.execute-api.<region>.amazonaws.com → Private IPs                │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

**Role:**
- Creates private network interface (ENI) in your subnets
- Routes traffic to API Gateway without leaving AWS network
- With Private DNS enabled, API Gateway URLs automatically resolve to private IPs

**Key Configuration:**
| Setting | Value | Purpose |
|---------|-------|---------|
| Service Name | `com.amazonaws.<region>.execute-api` | Connects to API Gateway |
| Private DNS | **Enabled** | Auto-resolves API Gateway URLs |
| Subnets | Multiple (HA) | ENIs for private connectivity |
| Security Group | Allow HTTPS (443) | Controls network access |

---

### Resource Policies

Resource Policies are **JSON documents** attached to the API that control **who can invoke the API**.

```
┌─────────────────────────────────────────────────────────────────┐
│                      RESOURCE POLICY                             │
│                                                                  │
│  Purpose: Control WHO can access the API                         │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  What it checks:                                         │    │
│  │  • Source VPC Endpoint ID (vpce-xxxxxxxxx)               │    │
│  │  • Source VPC ID                                         │    │
│  │  • Source IP range (for non-private APIs)                │    │
│  │  • AWS Account ID                                        │    │
│  │  • IAM Principal (for IAM auth)                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Example Resource Policy for Private API:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:<region>:<account>:<api-id>/*",
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0123456789abcdef0"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:<region>:<account>:<api-id>/*"
    }
  ]
}
```

**Policy Logic:**
1. **Deny** all requests NOT from the specified VPC endpoint
2. **Allow** all other requests (which must be from the VPC endpoint to pass the deny)

---

### IAM Permissions

```
┌─────────────────────────────────────────────────────────────────┐
│                      IAM PERMISSIONS                             │
│                                                                  │
│  Two Separate Permission Checks:                                 │
│                                                                  │
│  1. API Gateway → Lambda (Invoke Permission)                     │
│     ┌──────────────────────────────────────────────────────┐    │
│     │  Lambda Resource Policy must allow:                   │    │
│     │  Principal: apigateway.amazonaws.com                  │    │
│     │  Action: lambda:InvokeFunction                        │    │
│     │  (Automatically added via Console integration)        │    │
│     └──────────────────────────────────────────────────────┘    │
│                                                                  │
│  2. Lambda Execution Role                                        │
│     ┌──────────────────────────────────────────────────────┐    │
│     │  What Lambda can do when running:                     │    │
│     │  • Write to CloudWatch Logs                           │    │
│     │  • Access DynamoDB, S3, etc. (if needed)              │    │
│     └──────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**For This POC, Minimum Required:**

| Permission | Where | Purpose |
|------------|-------|---------|
| Lambda invoke | Lambda resource policy | Allow API Gateway to call Lambda |
| CloudWatch Logs | Lambda execution role | Allow Lambda to create logs |

---

### Lambda Execution Role

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAMBDA EXECUTION ROLE                         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Trust Relationship (Who can assume this role)            │   │
│  │  {                                                        │   │
│  │    "Principal": {                                         │   │
│  │      "Service": "lambda.amazonaws.com"                    │   │
│  │    },                                                     │   │
│  │    "Action": "sts:AssumeRole"                             │   │
│  │  }                                                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Permissions (What the role allows)                       │   │
│  │                                                           │   │
│  │  Minimum for this POC:                                    │   │
│  │  • logs:CreateLogGroup                                    │   │
│  │  • logs:CreateLogStream                                   │   │
│  │  • logs:PutLogEvents                                      │   │
│  │                                                           │   │
│  │  Additional (if your Lambda needs):                       │   │
│  │  • dynamodb:GetItem, PutItem, etc.                        │   │
│  │  • s3:GetObject, PutObject, etc.                          │   │
│  │  • And so on...                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Common Causes of 403 and 500 Errors

### 403 Forbidden Errors

```
┌─────────────────────────────────────────────────────────────────┐
│                    403 FORBIDDEN CAUSES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Resource Policy Blocking Request                             │
│     ├── Symptom: 403 with "User: anonymous"                      │
│     ├── Cause: Request not from allowed VPC endpoint             │
│     └── Fix: Update resource policy with correct vpce-id         │
│                                                                  │
│  2. Missing VPC Endpoint Private DNS                             │
│     ├── Symptom: 403 or connection timeout                       │
│     ├── Cause: DNS not resolving to private IPs                  │
│     └── Fix: Enable Private DNS on VPC endpoint                  │
│                                                                  │
│  3. API Not Deployed                                             │
│     ├── Symptom: 403 "Missing Authentication Token"              │
│     ├── Cause: Changes not deployed to stage                     │
│     └── Fix: Deploy API to the stage                             │
│                                                                  │
│  4. IAM Authorization Failure                                    │
│     ├── Symptom: 403 "not authorized to access"                  │
│     ├── Cause: Caller lacks execute-api:Invoke permission        │
│     └── Fix: Add IAM permission or use API key                   │
│                                                                  │
│  5. Invalid API Key                                              │
│     ├── Symptom: 403 "Forbidden"                                 │
│     ├── Cause: Missing/invalid x-api-key header                  │
│     └── Fix: Include valid API key in request                    │
│                                                                  │
│  6. WAF Blocking (if enabled)                                    │
│     ├── Symptom: 403 from WAF                                    │
│     ├── Cause: Request violates WAF rules                        │
│     └── Fix: Review and adjust WAF rules                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Troubleshooting 403 Errors - Decision Tree

```
                        403 Error?
                            │
            ┌───────────────┴───────────────┐
            ▼                               │
    Check Resource Policy                   │
    (is vpce-id correct?)                   │
            │                               │
    ┌───────┴───────┐                       │
   YES              NO                      │
    │               │                       │
    ▼               ▼                       │
  Continue      Fix policy,              Return
    │           redeploy API               │
    │               │                       │
    ▼               └───────────────────────┘
  Check if API
  is deployed?
        │
    ┌───┴───┐
   YES      NO
    │       │
    ▼       ▼
  Check   Deploy
  auth    to stage
  method
    │
    ▼
  Using IAM auth?  ───YES──►  Check caller IAM perms
    │
   NO
    │
    ▼
  API Key required?  ───YES──►  Check x-api-key header
    │
   NO
    │
    ▼
  Check CloudWatch logs for details
```

---

### 500 Internal Server Errors

```
┌─────────────────────────────────────────────────────────────────┐
│               500 INTERNAL SERVER ERROR CAUSES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Lambda Permission Missing                                    │
│     ├── Symptom: 500 "Internal server error"                     │
│     ├── Cause: API Gateway cannot invoke Lambda                  │
│     └── Fix: Add Lambda resource-based policy for API Gateway    │
│             Or recreate integration via Console                  │
│                                                                  │
│  2. Lambda Execution Error                                       │
│     ├── Symptom: 500 with Lambda error in logs                   │
│     ├── Cause: Code error, timeout, out of memory                │
│     └── Fix: Check Lambda CloudWatch logs                        │
│                                                                  │
│  3. Lambda Timeout                                               │
│     ├── Symptom: 500 after ~29-30 seconds                        │
│     ├── Cause: Lambda took too long (or API Gateway timeout)     │
│     └── Fix: Optimize Lambda or increase timeout                 │
│                                                                  │
│  4. Invalid Lambda Response Format                               │
│     ├── Symptom: 500, Lambda logs show success                   │
│     ├── Cause: Response not in expected format                   │
│     └── Fix: Return proper statusCode, headers, body             │
│                                                                  │
│  5. Integration Configuration Error                              │
│     ├── Symptom: 500 immediately                                 │
│     ├── Cause: Wrong Lambda ARN, region mismatch                 │
│     └── Fix: Verify integration settings                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Correct Lambda Response Format

For API Gateway to correctly process Lambda responses:

```python
def handler(event, context):
    # Your logic here
    
    # CORRECT response format
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "body": '{"message": "Success"}'  # Must be STRING, not dict
    }
```

**Common Mistakes:**

| Mistake | Result |
|---------|--------|
| Missing `statusCode` | 500 error |
| `body` as dict (not string) | 500 error |
| Missing return statement | 500 error |
| Exception in code | 500 error |

---

## 4. Request Flow Debugging Checklist

When troubleshooting, check each component in order:

```
□ Step 1: DNS Resolution
  └── Does API Gateway URL resolve to private IP?
      Test: nslookup <api-id>.execute-api.<region>.amazonaws.com

□ Step 2: Network Connectivity
  └── Can you reach the VPC endpoint?
      Test: curl -v https://<api-id>.execute-api.<region>.amazonaws.com

□ Step 3: Resource Policy
  └── Is the VPC endpoint allowed?
      Check: API Gateway Console → Resource Policy

□ Step 4: API Deployment
  └── Is the latest version deployed?
      Check: API Gateway Console → Stages → Deployment history

□ Step 5: Route Matching
  └── Does the path match a resource?
      Check: API Gateway Console → Resources

□ Step 6: Integration
  └── Is Lambda correctly configured?
      Check: API Gateway Console → Integration Request

□ Step 7: Lambda Permissions
  └── Can API Gateway invoke Lambda?
      Check: Lambda Console → Configuration → Permissions

□ Step 8: Lambda Execution
  └── Does Lambda run successfully?
      Check: CloudWatch Logs for Lambda function
```

---

**← Previous:** [02-fundamentals.md](02-fundamentals.md) | **Next:** [04-routing.md](04-routing.md) →
