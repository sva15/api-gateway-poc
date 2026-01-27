# Kong Gateway Visual Implementation Guide

A complete, screenshot-annotated walkthrough of configuring Kong Gateway to invoke AWS Lambda with API Key authentication.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Step 1: Create Gateway Service](#2-step-1-create-gateway-service)
3. [Step 2: Create Route](#3-step-2-create-route)
4. [Step 3: Configure AWS Lambda Plugin](#4-step-3-configure-aws-lambda-plugin)
5. [Step 4: Create Consumer & Key Credential](#5-step-4-create-consumer--key-credential)
6. [Step 5: Configure CORS Plugin](#6-step-5-configure-cors-plugin)
7. [Step 6: Configure Key Auth Plugin](#7-step-6-configure-key-auth-plugin)
8. [Step 7: Testing the Integration](#8-step-7-testing-the-integration)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Architecture Overview

Before diving into the configuration, understand the flow:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Browser   │────▶│     ALB     │────▶│    Nginx    │────▶│    Kong     │────▶│   Lambda    │
│  (UI App)   │◀────│  (AWS LB)   │◀────│  (Reverse   │◀────│  (Gateway)  │◀────│  (Function) │
│             │     │             │     │   Proxy)    │     │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
      │                   │                   │                   │                   │
      │   HTTP Request    │   Forward to      │   Strip prefix    │   Invoke via      │
      │   with x-api-key  │   EC2:80          │   /api-gateway    │   AWS SDK         │
      │                   │                   │                   │                   │
```

**Key Components Being Configured:**
| Component | Purpose |
|-----------|---------|
| Gateway Service | Abstract definition of your backend (placeholder for Lambda) |
| Route | URL paths that trigger this service (`/users`, `/config`, `/orders`) |
| AWS Lambda Plugin | Enables direct Lambda invocation instead of HTTP proxying |
| Consumer | Identity entity that holds API key credentials |
| CORS Plugin | Handles browser cross-origin requests |
| Key Auth Plugin | Enforces API key validation on incoming requests |

---

## 2. Step 1: Create Gateway Service

The Gateway Service is a logical grouping that represents your backend. Even though we'll use the Lambda plugin (which bypasses normal HTTP proxying), Kong requires a Service to attach Routes and Plugins.

### 2.1 Navigate to Gateway Services

In Kong Manager, click **Gateway Services** in the left sidebar, then click **+ New Gateway Service**.

### 2.2 Service Endpoint Configuration

![Gateway Service Part 1](images/kong-setup/1.create-gateway-service-part1.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Service endpoint** | Protocol, host, port, and path | Choose "Protocol, host, port, and path" for advanced control |
| **Protocol** | `http` | Standard HTTP protocol (Lambda plugin will override this) |
| **Host** | `localhost` | Placeholder value - not used when Lambda plugin is active |

> [!NOTE]
> The Host and Port values are **placeholders**. When the AWS Lambda plugin is attached, Kong bypasses normal HTTP routing and instead invokes the Lambda function directly via the AWS SDK.

### 2.3 Port and Advanced Fields

![Gateway Service Part 2](images/kong-setup/1.create-gateway-service-part2.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Path** | *(empty)* | Not needed for Lambda integration |
| **Port** | `80` | Default HTTP port (placeholder) |
| **Retries** | `5` | Number of retry attempts on failure |
| **Connection timeout** | `60000` | 60 seconds to establish connection |
| **Write timeout** | `60000` | 60 seconds to send request |

### 2.4 Service Name

![Gateway Service Part 3](images/kong-setup/1.create-gateway-service-part3.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Read timeout** | `60000` | 60 seconds to receive response |
| **Name** | `poc-lambda-service` | Human-readable identifier (change from auto-generated) |

> [!TIP]
> Always give your service a meaningful name like `poc-lambda-service` or `user-api-service` instead of accepting the auto-generated timestamp-based name.

Click **Save** to create the Gateway Service.

---

## 3. Step 2: Create Route

Routes define which URL paths should be handled by your Gateway Service. This is where you specify `/users`, `/config`, `/orders`, etc.

### 3.1 Navigate to Create Route

In Kong Manager, click **Routes** in the left sidebar, then click **+ Create Route**.

### 3.2 General Information

![Route Part 1](images/kong-setup/2.create-route-part1.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Name** | `poc-main-route` | Descriptive name for this route |
| **Service** | Select `poc-lambda-service` | Links this route to your Gateway Service |
| **Tags** | *(optional)* | For organization and filtering |

### 3.3 Route Configuration - Advanced Mode

![Route Part 2](images/kong-setup/2.create-route-part2.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Mode** | **Advanced** | Enables multiple paths and fine-grained control |
| **Protocols** | `HTTP, HTTPS` | Accept both protocols |
| **Paths** | `/users`, `/config` | First two paths added here |

> [!IMPORTANT]
> Select **Advanced** mode to add multiple paths. Basic mode only allows a single path.

### 3.4 Adding More Paths

![Route Part 3](images/kong-setup/2.create-route-part3.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Paths** | Add `/orders` | Click "+ Add a path" to add more |
| **Strip Path** | **OFF** *(unchecked)* | Lambda receives the full path like `/users` |
| **Methods** | *(empty = all methods)* | Accept GET, POST, PUT, DELETE, etc. |
| **Hosts** | *(empty)* | Match any host header |

> [!CAUTION]
> **Strip Path = OFF is critical!** If enabled, Kong would strip the path before sending to Lambda. Your Lambda would receive `/` instead of `/users`, breaking your routing logic.

### 3.5 Headers and SNIs

![Route Part 4](images/kong-setup/2.create-route-part4.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Headers** | *(empty)* | No header-based routing needed |
| **SNIs** | *(empty)* | Server Name Indication for TLS (not needed) |
| **Path Handling** | `v0` | Legacy path handling mode |
| **HTTPS Redirect Status Code** | `426` | Upgrade Required response for HTTP→HTTPS redirect |

### 3.6 Buffering Options

![Route Part 5](images/kong-setup/2.create-route-part5.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Regex Priority** | `0` | Order when multiple regex routes match |
| **Preserve Host** | **OFF** | Let Kong set the host header |
| **Request buffering** | **ON** *(checked)* | Buffer entire request before proxying |
| **Response buffering** | **ON** *(checked)* | Buffer entire response before returning |

Click **Save** to create the Route.

---

## 4. Step 3: Configure AWS Lambda Plugin

This is the **core plugin** that makes Kong invoke your Lambda function directly instead of making an HTTP request to an upstream server.

### 4.1 Navigate to Plugins

Go to **Plugins** → **+ New Plugin** → Search for `aws-lambda` → Click **Enable**.

### 4.2 Plugin Scope Selection

![Lambda Plugin Part 1](images/kong-setup/3.aws-lambda-plugin-part1.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Plugin Status** | **Enabled** | Toggle ON |
| **Scope** | **Scoped** | Apply to specific routes, not globally |
| **Select a gateway service** | `poc-lambda-service` | *(optional - can also scope to route)* |
| **Select a route** | `poc-main-route` | Best practice: scope to route |
| **Select a consumer** | *(empty)* | Leave empty - applies to all consumers |

> [!NOTE]
> **Why leave Consumer empty?** The Lambda plugin should run for ANY authenticated request. The Key Auth plugin handles WHO can access; the Lambda plugin handles WHAT happens when they do.

### 4.3 Protocol Configuration

![Lambda Plugin Part 2](images/kong-setup/3.aws-lambda-plugin-part2.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Protocols** | `grpc`, `grpcs`, `http`, `https` | All protocols enabled |
| **AWS IMDS Protocol Version** | `v1` | Instance Metadata Service version for EC2 credentials |
| **AWSGateway Compatible Payload Version** | `1.0` | API Gateway event format |
| **Empty Arrays Mode** | `legacy` | How to handle empty arrays in JSON |

### 4.4 Invocation Settings

![Lambda Plugin Part 3](images/kong-setup/3.aws-lambda-plugin-part3.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Invocation Type** | `RequestResponse` | Synchronous invocation (wait for Lambda response) |
| **Keepalive** | `60000` | Connection keep-alive in milliseconds |
| **Log Type** | `Tail` | Include Lambda execution logs in response |
| **Timeout** | `60000` | Maximum wait time for Lambda response |

> [!TIP]
> `RequestResponse` = Synchronous (wait for answer)  
> `Event` = Asynchronous (fire and forget)  
> `DryRun` = Validate without executing

### 4.5 AWS Authentication (Advanced Parameters)

![Lambda Plugin Part 4](images/kong-setup/3.aws-lambda-plugin-part4.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Instance Name** | *(empty)* | For multi-instance deployments |
| **Tags** | *(empty)* | Optional metadata |
| **AWS Assume Role ARN** | *(empty)* | Leave empty to use EC2 instance role |
| **AWS Key** | *(empty)* | Leave empty to use EC2 instance role |

> [!IMPORTANT]
> **Leave AWS Key and AWS Secret EMPTY!** Kong will automatically use the IAM role attached to your EC2 instance. This is more secure than hardcoding credentials.

### 4.6 AWS Region and Session

![Lambda Plugin Part 5](images/kong-setup/3.aws-lambda-plugin-part5.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **AWS Region** | *(empty or `ap-south-1`)* | Required if not using IMDS |
| **AWS Role Session Name** | `kong` | Identifier for CloudTrail logging |
| **AWS Secret** | *(empty)* | Leave empty for IAM role |
| **AWS STS Endpoint URL** | *(empty)* | Custom STS endpoint (not needed) |

### 4.7 Request Forwarding Options

![Lambda Plugin Part 6](images/kong-setup/3.aws-lambda-plugin-part6.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **AWSGateway Compatible** | **OFF** | Use Kong's native payload format |
| **Base64 Encode Body** | **ON** *(checked)* | Encode binary request bodies |
| **Disable HTTPS** | **OFF** | Always use HTTPS to Lambda |
| **Forward Request Body** | **OFF** | See note below |
| **Forward Request Headers** | **OFF** | See note below |
| **Forward Request Method** | **OFF** | See note below |
| **Forward Request URI** | **OFF** | See note below |
| **Function Name** | `kong-poc-lambda` | **YOUR LAMBDA FUNCTION NAME** |
| **Host** | *(empty)* | Custom Lambda endpoint (not needed) |

> [!WARNING]
> The `Forward Request *` options are only needed when `AWSGateway Compatible` is OFF and you want Kong to manually add these to the event. Usually, the payload already includes this data.

### 4.8 Proxy and Final Settings

![Lambda Plugin Part 7](images/kong-setup/3.aws-lambda-plugin-part7.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Is Proxy Integration** | **OFF** | Let Kong transform the response |
| **Port** | `443` | HTTPS port for Lambda service |
| **Proxy URL** | *(empty)* | HTTP proxy for Lambda calls (not needed) |
| **Qualifier** | *(empty)* | Lambda alias or version (e.g., `$LATEST`) |
| **Skip Large Bodies** | **ON** *(checked)* | Don't send bodies larger than 1MB |

### 4.9 Unhandled Status

![Lambda Plugin Part 8](images/kong-setup/3.aws-lambda-plugin-part8.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Unhandled Status** | *(empty)* | HTTP status for Lambda errors (defaults to 500) |

Click **Save** to enable the AWS Lambda plugin.

---

## 5. Step 4: Create Consumer & Key Credential

A Consumer is an identity in Kong that holds API key credentials. Even if you have just one client (your UI), you need a Consumer to store its key.

### 5.1 Navigate to Consumers

Go to **Consumers** → **+ New Consumer**.

### 5.2 Consumer Identification

![Consumer Part 1](images/kong-setup/4.create-consumer-part1.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Username** | `frontend-app` | Human-readable identifier |
| **Custom ID** | *(optional)* | Your internal user ID |

> [!TIP]
> Think of the Consumer as "who is calling your API". For a single frontend app, one Consumer is sufficient.

### 5.3 Tags and Save

![Consumer Part 2](images/kong-setup/4.create-consumer-part2.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Tags** | *(optional)* | For organization |

Click **Save** to create the Consumer.

### 5.4 Create Key Credential

After creating the Consumer, navigate to the Consumer's detail page:
**Consumers** → Click `frontend-app` → **Credentials** tab → **+ New Key Auth Credential**

![Consumer Part 3](images/kong-setup/4.create-consumer-part3.png)

This shows the Credentials page for `frontend-app` with the **Key Authentication** section. A key has already been created (shown masked as `•••••`).

### 5.5 Key Credential Form

![Consumer Part 4](images/kong-setup/4.create-consumer-part4.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Key** | `your-secret-api-key-here` | The API key value (auto-generated if left empty) |
| **Tags** | *(optional)* | For organization |
| **TTL** | *(empty)* | Time-to-live in seconds (empty = never expires) |

> [!CAUTION]
> **Copy the key immediately!** If you let Kong auto-generate it, you can only see it once. Store it securely.

Click **Save** to create the credential.

---

## 6. Step 5: Configure CORS Plugin

The CORS plugin handles browser preflight requests and adds the necessary `Access-Control-*` headers for cross-origin requests.

### 6.1 Navigate to CORS Plugin

Go to **Plugins** → **+ New Plugin** → Search for `cors` → Click **Enable**.

### 6.2 Plugin Scope and Protocols

![CORS Part 1](images/kong-setup/5.cors-plugin-part1.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Plugin Status** | **Enabled** | Toggle ON |
| **Scope** | **Global** | Apply to all routes |
| **Protocols** | `grpc`, `grpcs`, `http`, `https` | All protocols |
| **Allow Origin Absent** | **ON** *(checked)* | Allow requests without Origin header |

> [!NOTE]
> **Global scope for CORS** is common since you typically want consistent CORS behavior across all endpoints.

### 6.3 Advanced Options

![CORS Part 2](images/kong-setup/5.cors-plugin-part2.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Credentials** | **OFF** | Don't allow `Access-Control-Allow-Credentials` |
| **Preflight Continue** | **OFF** | Kong handles preflight, doesn't pass to upstream |
| **Private Network** | **OFF** | Not a private network access request |
| **Instance Name** | *(empty)* | For multi-instance deployments |
| **Tags** | *(empty)* | Optional metadata |
| **Exposed Headers** | *(empty)* | Headers browsers can access (e.g., `X-Request-Id`) |

### 6.4 Headers and Max Age

![CORS Part 3](images/kong-setup/5.cors-plugin-part3.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Exposed Headers** | Click "+ Add" to add values | Headers visible to JavaScript |
| **Headers** | Click "+ Add" to add values | Allowed request headers |
| **Max Age** | *(empty)* | Seconds to cache preflight (empty = browser default) |

### 6.5 Methods and Origins

![CORS Part 4](images/kong-setup/5.cors-plugin-part4.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Methods** | `GET,HEAD,PUT,PATCH,POST,DELETE,OPTIONS,TRACE,CONNECT` | All HTTP methods allowed |
| **Origins** | Click "+ Add" | Allowed origins (e.g., `*` for all, or `https://your-domain.com`) |

> [!TIP]
> For development, you can use `*` for Origins. For production, specify your exact frontend domain like `https://app.example.com`.

Click **Save** to enable the CORS plugin.

---

## 7. Step 6: Configure Key Auth Plugin

The Key Auth plugin **enforces** API key authentication. Without it, anyone can access your APIs.

### 7.1 Navigate to Key Auth Plugin

Go to **Plugins** → **+ New Plugin** → Search for `key-auth` → Click **Enable**.

### 7.2 Plugin Scope and Protocols

![Key Auth Part 1](images/kong-setup/6.key-auth-plugin-part1.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Plugin Status** | **Enabled** | Toggle ON |
| **Scope** | **Global** | Apply to all routes |
| **Protocols** | `http`, `https` | Only HTTP protocols |
| **Hide Credentials** | **ON** *(checked)* | Don't forward the API key to Lambda |

### 7.3 Key Location and Names

![Key Auth Part 2](images/kong-setup/6.key-auth-plugin-part2.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Hide Credentials** | **ON** *(checked)* | Strip key from request before forwarding |
| **Key In Body** | **OFF** | Don't look for key in request body |
| **Key In Header** | **ON** *(checked)* | Look for key in request headers |
| **Key In Query** | **OFF** | Don't look for key in query string |
| **Key Names** | `x-api-key` | Header name(s) to check for the key |
| **Run On Preflight** | **OFF** | Don't require key for OPTIONS requests |

> [!IMPORTANT]
> **Key Names = `x-api-key`** means the client must send the header:  
> `x-api-key: your-secret-key-here`

### 7.4 Advanced Options

![Key Auth Part 3](images/kong-setup/6.key-auth-plugin-part3.png)

| Field | Value | Explanation |
|-------|-------|-------------|
| **Instance Name** | *(empty)* | For multi-instance deployments |
| **Tags** | *(empty)* | Optional metadata |
| **Anonymous** | *(empty)* | Consumer ID to use for anonymous access |
| **Realm** | *(empty)* | Realm value for WWW-Authenticate header |

> [!TIP]
> The **Anonymous** field lets you specify a fallback Consumer for unauthenticated requests. Leave empty to require authentication.

Click **Save** to enable the Key Auth plugin.

---

## 8. Step 7: Testing the Integration

Now let's verify everything works end-to-end.

### 8.1 Test WITHOUT API Key (Should Fail)

![No API Key Error](images/kong-setup/7.no-api-key-error-kong-invoke.png)

**Command:**
```bash
curl -w "\n" "http://kong-poc-1434114124.ap-south-1.elb.amazonaws.com/api-gateway/users/id=1"
```

**Response:**
```json
{
  "message": "No API key found in request",
  "request_id": "7f1f4f17ed8e343f42c1962ad707ba47"
}
```

**What happened:**
1. Request reached Kong via ALB → Nginx
2. Key Auth Plugin checked for `x-api-key` header
3. No key found → **403 Forbidden** returned
4. Lambda was **never invoked**

### 8.2 Test WITH API Key (Should Succeed)

![With API Key Success](images/kong-setup/8.with-api-key-success-kong-invoke.png)

**Command:**
```bash
curl -w "\n" -H "x-api-key: s3K5mGfjHN08hCJwlxxvY99PIH6lgfOe" \
  "http://kong-poc-1434114124.ap-south-1.elb.amazonaws.com/api-gateway/users/"
```

**Response:**
```json
{
  "action": "LIST_USERS",
  "users": [
    {
      "id": "1",
      "name": "Alice",
      "email": "alice@example.com"
    },
    {
      "id": "2",
      "name": "Bob",
      "email": "bob@example.com"
    },
    {
      "id": "3",
      "name": "Charlie",
      "email": "charlie@example.com"
    }
  ]
}
```

**What happened:**
1. Request reached Kong via ALB → Nginx
2. Key Auth Plugin found `x-api-key` header
3. Key validated against Consumer `frontend-app` → **Access Granted**
4. Lambda Plugin invoked `kong-poc-lambda` via AWS SDK
5. Lambda processed the request and returned user list
6. Kong formatted response and sent back to client

---

## 9. Troubleshooting

### Issue: "No API key found in request"
- **Cause:** Missing or incorrect header name
- **Fix:** Ensure you're sending `x-api-key` (check spelling, case-sensitive)

### Issue: "Invalid authentication credentials"
- **Cause:** Key exists but doesn't match any Consumer
- **Fix:** Verify the key value matches what's stored in the Consumer's credentials

### Issue: 500 Internal Server Error
- **Cause:** Lambda invocation failed
- **Fixes:**
  1. Check EC2 IAM role has `lambda:InvokeFunction` permission
  2. Check security groups allow outbound HTTPS (443) to AWS
  3. Check Kong logs: `docker logs kong-cp`
  4. Check CloudWatch logs for Lambda errors

### Issue: CORS errors in browser
- **Cause:** CORS plugin not configured or Origins not set
- **Fix:** Add your frontend domain to the Origins list, or use `*` for testing

---

## Summary: Plugin Chain Execution Order

When a request arrives, Kong processes plugins in this order:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         REQUEST ARRIVES                              │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
              ┌─────────────────────────────────────┐
              │         1. Key Auth Plugin          │
              │   "Is there a valid API key?"       │
              │   ❌ No → Return 401/403            │
              │   ✅ Yes → Continue                 │
              └─────────────────────────────────────┘
                                  │
                                  ▼
              ┌─────────────────────────────────────┐
              │         2. CORS Plugin              │
              │   "Add CORS headers if needed"      │
              │   (Handles OPTIONS preflight)       │
              └─────────────────────────────────────┘
                                  │
                                  ▼
              ┌─────────────────────────────────────┐
              │       3. AWS Lambda Plugin          │
              │   "Invoke the Lambda function"      │
              │   → Wraps request in JSON event     │
              │   → Uses AWS SDK to call Lambda     │
              │   → Unwraps response to HTTP        │
              └─────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       RESPONSE RETURNED                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

**Congratulations!** 🎉 You have successfully configured Kong Gateway to:
1. ✅ Accept requests on multiple paths (`/users`, `/config`, `/orders`)
2. ✅ Enforce API key authentication
3. ✅ Handle browser CORS requirements
4. ✅ Invoke AWS Lambda directly (no HTTP upstream needed)
5. ✅ Return formatted responses to clients
