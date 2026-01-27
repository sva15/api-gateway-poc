# Kong Lambda Integration Guide

This guide details how to configure Kong Gateway to invoke your AWS Lambda function (`kong-poc-lambda`). We cover two methods:
1.  **Direct Invocation (Recommended):** Kong uses the AWS SDK to invoke Lambda directly. Faster, cheaper (no ALB data transfer), and more secure.
2.  **Via ALB (HTTP Proxy):** Kong proxies requests to the ALB, which then routes to Lambda. Easier to set up if you already have a working ALB endpoint.

---

## Method 1: Direct Lambda Invocation (Recommended)

This method uses the `aws-lambda` plugin. Kong acts as the "API Gateway" directly.

### 1. Create a "Dummy" Gateway Service
Since the plugin handles the actual invocation, the Service URL is just a placeholder.

1.  **Name:** `poc-lambda-direct-service`
2.  **Protocol:** `http`
3.  **Host:** `localhost` (Dummy value)
4.  **Port:** `80` (Dummy value)
5.  **Path:** `/` (Dummy value)

**Why specific values?** The `aws-lambda` plugin intercepts the request *before* it tries to connect to `localhost`.

### 2. Create a Route
This defines the URL path that clients will use to access the Lambda.

1.  **Name:** `poc-lambda-route`
2.  **Service:** `poc-lambda-direct-service`
3.  **Paths:** `/api/v1` (or whatever prefix you want, e.g., `/users`)
4.  **Strip Path:** `No` (Important: We want to pass the full path to Lambda so it can route internally)
5.  **Methods:** `GET`, `POST`, `PUT`, `DELETE` (Select all relevant)

### 3. Configure Plugins
Apply these plugins to the **Service** or **Route** (Route is more granular).

#### A. AWS Lambda Plugin
This is the core integration.

1.  **Plugin Name:** `aws-lambda`
2.  **Config:**
    *   **AWS Region:** `ap-south-1`
    *   **Function Name:** `kong-poc-lambda`
    *   **Invocation Type:** `RequestResponse` (Synchronous)
    *   **Forward Request Body:** `Yes`
    *   **Forward Request Headers:** `Yes`
    *   **Forward Request Method:** `Yes`
    *   **Forward Request URI:** `Yes`
    *   **Is Proxy Integration:** `Yes` (Crucial for handling responses correctly)
    *   **Awsgateway Compatible:** `Yes` (Matches API Gateway event format)
    *   **AWS Key/Secret:** Leave **EMPTY** (Kong uses the EC2 IAM Role)

#### B. CORS Plugin
Enable Cross-Origin Resource Sharing for browser clients.

1.  **Plugin Name:** `cors`
2.  **Config:**
    *   **Origins:** `*` (or your specific domain)
    *   **Methods:** `GET, POST, PUT, DELETE, OPTIONS`
    *   **Headers:** `Content-Type, Authorization, x-api-key`
    *   **Exposed Headers:** `x-kong-request-id`
    *   **Max Age:** `3600`
    *   **Credentials:** `true`

#### C. Key Authentication Plugin
Secure the API with an API Key.

1.  **Plugin Name:** `key-auth`
2.  **Config:**
    *   **Key Names:** `x-api-key`
    *   **Key in Body:** `No` (Cleaner to keep in header)
    *   **Hide Credentials:** `Yes` (Don't pass the key to backend)

### 4. Create a Consumer & API Key
You need a "user" to attach the API Key to.

1.  **Consumer Username:** `frontend-app` (or any identifier)
2.  **Credentials:**
    *   Go to the created Consumer -> **Credentials** tab.
    *   Click **New Key Auth Credential**.
    *   **Key:** `s3cr3t-k3y-123` (or auto-generate)
    *   Save it.

### 5. Test It
Use `curl` to verify the setup.

```bash
# Direct Invocation Test
curl -X GET http://<ALB-DNS-NAME>/api/v1/users \
  -H "x-api-key: s3cr3t-k3y-123"
```
**Expected Response:** JSON from your Lambda's `list_users` function.

---

## Method 2: Via ALB (HTTP Proxy)

This method simply proxies traffic to the existing ALB endpoint that triggers Lambda.

### 1. Create Gateway Service
Point to the ALB's DNS name.

1.  **Name:** `poc-lambda-alb-service`
2.  **Protocol:** `http`
3.  **Host:** `<ALB-DNS-NAME>` (e.g., `kong-poc-123.ap-south-1.elb.amazonaws.com`)
4.  **Port:** `80`
5.  **Path:** `/`

### 2. Create Route
1.  **Name:** `poc-alb-route`
2.  **Service:** `poc-lambda-alb-service`
3.  **Paths:** `/alb-api`
4.  **Strip Path:** `Yes` (If you want `/alb-api/users` -> `/users` on ALB)

### 3. Configure Plugins
Same as Method 1 for **CORS** and **Key Auth**.
**Do NOT** add the `aws-lambda` plugin here.

### 4. Test It
```bash
curl -X GET http://<ALB-DNS-NAME>/alb-api/users \
  -H "x-api-key: s3cr3t-k3y-123"
```

---

## Summary of Differences

| Feature | Direct Invocation (Method 1) | Via ALB (Method 2) |
| :--- | :--- | :--- |
| **Performance** | **Faster** (Internal network call) | Slower (Extra hop out to ALB) |
| **Cost** | **Cheaper** (No ALB processing) | Standard ALB costs apply |
| **Setup** | slightly more config (Plugin) | simple proxy |
| **Security** | IAM Role based | Security Group based |
