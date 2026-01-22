# Prerequisites for AWS API Gateway Private REST APIs

This section covers **only what is required** to successfully complete the POC—not generic AWS theory.

---

## 1. AWS Account Setup and Region Selection

### What You Need
- An active AWS account with administrative access
- A selected region where all resources will be created

### Why It Matters
- Private APIs require VPC resources that are **region-specific**
- All components (VPC, endpoints, API Gateway, Lambda) must be in the **same region**
- Choose a region close to your users/team (e.g., `ap-south-1` for India, `us-east-1` for US)

### Quick Check
```
✅ AWS Console access verified
✅ Region selected (all resources will be created here)
✅ Sufficient IAM permissions to create: VPC, Lambda, API Gateway, VPC Endpoints
```

---

## 2. VPC Basics

### Components Required for This POC

```
┌─────────────────────────────────────────────────────────────┐
│                         VPC                                  │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │  Private Subnet │    │  Private Subnet │                │
│  │     (AZ-1a)     │    │     (AZ-1b)     │                │
│  │                 │    │                 │                │
│  │  ┌───────────┐  │    │  ┌───────────┐  │                │
│  │  │ ENI (VPC  │  │    │  │ ENI (VPC  │  │                │
│  │  │ Endpoint) │  │    │  │ Endpoint) │  │                │
│  │  └───────────┘  │    │  └───────────┘  │                │
│  └─────────────────┘    └─────────────────┘                │
│                                                              │
│  Route Table: Local routes only (no internet gateway)       │
│  Security Group: Allow HTTPS (443) inbound                  │
└─────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Component | Purpose in This POC |
|-----------|---------------------|
| **VPC** | Isolated network containing all resources |
| **Private Subnets** | Subnets with no internet gateway route; internal traffic only |
| **Route Tables** | Define traffic routing; for private subnets, only VPC-local routes |
| **Security Groups** | Firewall rules; must allow HTTPS (port 443) for VPC endpoints |

### Requirements
- At least **2 private subnets** in different Availability Zones (for high availability)
- **No internet gateway** attached to route tables (ensures traffic stays internal)
- Security group allowing **inbound HTTPS (443)** from within the VPC

---

## 3. Interface VPC Endpoints

### What They Are

VPC Interface Endpoints (powered by AWS PrivateLink) create **private connections** to AWS services without leaving the VPC.

```
┌──────────────────────────────────────────────────────────────┐
│                           VPC                                 │
│                                                               │
│   ┌─────────────┐          ┌──────────────────────┐          │
│   │   Client    │ ──────►  │  VPC Interface       │          │
│   │ (EC2/Lambda)│  HTTPS   │  Endpoint (ENI)      │          │
│   └─────────────┘          │  execute-api         │          │
│                            └──────────┬───────────┘          │
│                                       │                       │
└───────────────────────────────────────┼───────────────────────┘
                                        │ AWS PrivateLink
                                        ▼
                              ┌──────────────────┐
                              │   API Gateway    │
                              │  (AWS Service)   │
                              └──────────────────┘
```

### Why Needed for Private APIs

| Without VPC Endpoint | With VPC Endpoint |
|---------------------|-------------------|
| Traffic goes through internet | Traffic stays within AWS network |
| Requires NAT Gateway + IGW | No internet connectivity needed |
| Exposes traffic externally | Completely private |
| **Cannot access Private APIs** | **Required for Private APIs** |

### Service Name
For API Gateway, the endpoint service name follows this pattern:
```
com.amazonaws.<region>.execute-api
```

Example: `com.amazonaws.ap-south-1.execute-api`

### Critical Configuration
- **Enable Private DNS** - Automatically resolves API Gateway URLs to the private endpoint
- **Associate with correct subnets** - Place in private subnets where clients run
- **Security group** - Must allow inbound HTTPS (443)

---

## 4. IAM Fundamentals

### Key Concepts for This POC

#### IAM Roles
- **Identity** that AWS services can assume
- Contains permissions (via policies) to perform actions

#### IAM Policies
- JSON documents defining **what actions are allowed/denied**
- Attached to roles, users, or groups

#### Trust Relationships
- Define **who can assume a role**
- For Lambda: allows the Lambda service to assume the role

### Roles Required

| Role | Purpose | Trust Relationship |
|------|---------|-------------------|
| **Lambda Execution Role** | Permissions for Lambda to run and log | `lambda.amazonaws.com` |
| **API Gateway Logging Role** (optional) | Permissions to write to CloudWatch | `apigateway.amazonaws.com` |

### Example: Lambda Execution Role Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## 5. Lambda Basics

### Lambda Execution Model

```
┌───────────────────────────────────────────────────────────────┐
│                      Lambda Function                           │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Code (Python)                                           │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │  def handler(event, context):                     │   │  │
│  │  │      # event = input from trigger                 │   │  │
│  │  │      # context = runtime info                     │   │  │
│  │  │      return response                              │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Execution Role: Permissions to access AWS resources           │
│  Logs: Automatically sent to CloudWatch Logs                   │
└───────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Execution Role** | IAM role that Lambda assumes to access other AWS services |
| **Handler** | Function entry point (`filename.function_name`) |
| **Event** | Input data passed to the function |
| **Context** | Runtime information (request ID, time remaining, etc.) |

### Permissions for This POC

The Lambda execution role needs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

This allows Lambda to create and write to CloudWatch Logs.

---

## 6. CloudWatch Logs and Metrics

### CloudWatch Logs

Every Lambda invocation automatically logs to CloudWatch:

```
Log Group: /aws/lambda/<function-name>
└── Log Stream: YYYY/MM/DD/[$LATEST]<random-id>
    ├── START RequestId: xxx
    ├── [Your print/logging statements]
    └── END RequestId: xxx
```

### What Gets Logged

| Source | Log Content |
|--------|-------------|
| **Lambda runtime** | START, END, REPORT (duration, memory) |
| **Your code** | `print()` statements, `logging` module output |
| **API Gateway** (if enabled) | Request/response details, latency |

### CloudWatch Metrics for API Gateway

| Metric | Description |
|--------|-------------|
| `Count` | Number of API calls |
| `Latency` | Time to process requests |
| `4XXError` | Client-side errors |
| `5XXError` | Server-side errors |
| `IntegrationLatency` | Time for backend (Lambda) to respond |

---

## 7. Private Service Access vs Public AWS Services

### Public AWS Service Access

```
┌─────────────────────────────────────────────────────────────┐
│                           VPC                                │
│   ┌─────────────┐                                           │
│   │   Client    │ ─────► NAT Gateway ─────► Internet        │
│   │   (EC2)     │                             │              │
│   └─────────────┘                             ▼              │
└─────────────────────────────────────────────────────────────┘
                                          AWS Public Services
                                          (S3, DynamoDB, etc.)
```

**Characteristics:**
- Traffic exits VPC through NAT Gateway/Internet Gateway
- Uses public endpoints
- Network traffic potentially visible externally
- Requires internet connectivity

### Private Service Access (PrivateLink)

```
┌─────────────────────────────────────────────────────────────┐
│                           VPC                                │
│   ┌─────────────┐     ┌───────────────┐                     │
│   │   Client    │────►│ VPC Endpoint  │                     │
│   │   (EC2)     │     │   (Private)   │                     │
│   └─────────────┘     └───────┬───────┘                     │
└───────────────────────────────┼─────────────────────────────┘
                                │ AWS PrivateLink
                                ▼
                    AWS Service (API Gateway Private API)
```

**Characteristics:**
- Traffic stays within AWS network
- Uses private IP addresses
- No internet connectivity required
- More secure for sensitive workloads

### Comparison Table

| Aspect | Public Access | Private Access (PrivateLink) |
|--------|--------------|------------------------------|
| **Network Path** | Through internet | Within AWS network |
| **Requires Internet** | Yes (NAT/IGW) | No |
| **IP Addresses** | Public IPs | Private IPs |
| **Security** | Network-exposed | Network-isolated |
| **Cost** | NAT Gateway costs | VPC Endpoint costs |
| **Use Case** | General access | Sensitive/internal APIs |

---

## Prerequisites Checklist

Before proceeding to the hands-on implementation, verify:

```
□ AWS Account & Permissions
  ├── □ Console access working
  ├── □ Region selected
  └── □ Can create: VPC, Lambda, API Gateway, VPC Endpoints

□ VPC Ready
  ├── □ VPC exists or will create
  ├── □ At least 2 private subnets (different AZs)
  └── □ Security group allowing HTTPS (443) inbound

□ Understanding Complete
  ├── □ Know what VPC Endpoints do
  ├── □ Understand IAM roles and trust relationships
  └── □ Know how Lambda execution and logging works
```

---

**← Previous:** [00-overview.md](00-overview.md) | **Next:** [02-fundamentals.md](02-fundamentals.md) →
