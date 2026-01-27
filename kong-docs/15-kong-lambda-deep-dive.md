# Kong ↔ Lambda: The Deep Dive

This document explains **exactly** how Kong invokes your Lambda function, how the value `function_name` is enough, and how the URL transforms at every step.

---

## 1. How Kong Invokes Lambda (The "Magic")

You asked: *"we are just passing lambda name is this enough?"*

**Yes, absolutely.** Here is how it works:

1.  **IAM Role:** Your EC2 instance has an **IAM Role** attached.
2.  **Permissions:** This role has the policy `lambda:InvokeFunction`.
3.  **The Plugin:** When you configure the `aws-lambda` plugin with `function_name: kong-poc-lambda`, Kong uses the AWS SDK (built-in) to sign a request using the EC2's temporary credentials.
4.  **The Call:** Kong does **not** make a standard HTTP request to the Lambda. It makes a special `Invoke` API call to the AWS Lambda Service (Regional Endpoint).
    *   It wraps your original HTTP request (headers, body, path) into a JSON object.
    *   It sends this JSON to Lambda.
    *   Your Python Lambda receives this JSON as the `event` object.

**Visualizing the "Invoke" payload:**

When you call `GET /users?id=1`, Kong sends this JSON to Lambda:

```json
{
    "request_method": "GET",
    "request_uri": "/users?id=1",
    "request_headers": {
        "x-api-key": "your-key",
        "host": "alb-url.com"
    },
    "request_body": ""
}
```

---

## 2. The Exact Traffic Flow (Step-by-Step)

Here is the precise journey of a request from your UI to the Lambda.

### Scenario: UI calls "Get All Users"

**1. Browser / UI Application**
*   **Target URL:** `http://kong-poc-123.ap-south-1.elb.amazonaws.com/api-gateway/users`
*   **Method:** `GET`
*   **Header:** `x-api-key: secret-123`

⬇️ *Request hits the Application Load Balancer (ALB)*

**2. AWS ALB**
*   **Listener Rule:** Matches Path `/api-gateway/*`
*   **Action:** Forward to Target Group `kong-ec2-tg` (Port 80 on EC2)

⬇️ *Request hits Nginx on EC2*

**3. Nginx (Reverse Proxy)**
*   **Location Block:** `location /api-gateway/ { proxy_pass http://127.0.0.1:8000/; }`
*   **Transformation:** Nginx **removes** `/api-gateway/` from the path.
*   **New URL:** `http://127.0.0.1:8000/users`
*   **Headers:** Preserved (`x-api-key` is still there).

⬇️ *Request hits Kong Gateway (Service)*

**4. Kong Gateway**
*   **Port:** 8000 (Proxy)
*   **Router:** Checks configured Routes.
    *   Does `/users` match your configured Route? **YES.**
*   **Plugins:** Kong sees the `aws-lambda` plugin is enabled on this route.
*   **Action:** Stop! Do not forward to any HTTP upstream. **Invoke Lambda instead.**

⬇️ *Kong SDK calls AWS Lambda Service*

**5. AWS Lambda Service**
*   Receives the `Invoke` command for function `kong-poc-lambda`.
*   Spins up your Python environment.
*   Passes the JSON event (containing `path: /users`).

**6. Your Lambda Code (Python)**
*   `event['request_uri']` is `/users`.
*   Your router sees `/users` and calls the `list_users()` function.
*   Returns JSON: `{"users": [...]}`.

---

## 3. Configuration for Your Specific Routes

You mentioned: `/users*`, `/config*`, `/orders*`, `/health`, `/`

You should create **One Service** and **One Route** (with multiple paths).

### Step A: The Gateway Service
*   **Name:** `poc-lambda-service`
*   **Protocol:** `http`
*   **Host:** `localhost` (Placeholder, unused by Lambda plugin)
*   **Port:** `80`

### Step B: The Route (Connecting URLs to Service)
*   **Name:** `poc-main-route`
*   **Service:** `poc-lambda-service`
*   **Paths:** (Click "Add Path" for each)
    *   `/users`
    *   `/config`
    *   `/orders`
    *   `/health`
    *   `/`
*   **Strip Path:** `No` (We want Lambda to see the full path `/users`, not just empty `/`)

### Step C: The Lambda Plugin
*   **Target:** `poc-main-route` (The route we just created)
*   **Function Name:** `kong-poc-lambda`
*   **Region:** `ap-south-1` (or your region)

---

## 4. Why "Consumer" is Required for API Keys

You asked to **remove** the Consumer.

**Logic Check:**
If you want to use **API Keys** (passed in headers), Kong **must** know which keys are valid.
Kong stores keys inside "Consumers".

*   **No Consumer = No Place to store the API Key.**
*   **No API Key = Public API (Anyone can call it).**

**The Solution:**
You don't need a Consumer for *every real user*. You just need **ONE** generic consumer to hold the key for your UI application.

1.  Create Consumer named `ui-application`.
2.  Create a Key Credential for it: `my-secret-ui-key`.
3.  Now, when the UI sends `x-api-key: my-secret-ui-key`, Kong checks: *"Does this key exist? Yes, it belongs to 'ui-application'. Allow access."*

---

## 5. The Return Journey (Response Flow)

You asked: *"how exactly this will call lambda how the url will look in each phases"*

We saw the **Request**. Now let's see the **Response** path.

**Scenario:** Lambda successfully finds the user and returns data.

**1. Your Python Code (Lambda)**
*   Finishes processing `list_users()`.
*   Constructs a JSON response (Required format for Kong/API Gateway):
    ```json
    {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": "{\"users\": [{\"id\": 1, \"name\": \"Alice\"}]}"
    }
    ```
*   **Crucial:** This is just a JSON object. It is NOT an HTTP response yet.

⬇️ *Lambda Service returns JSON to Kong*

**2. Kong Gateway (The Translator)**
*   **Receives:** The JSON object from Lambda.
*   **Unpacks it:**
    *   Reads `statusCode: 200` → Sets HTTP Status 200.
    *   Reads `headers` → Adds them to the HTTP response.
    *   Reads `body` → Sets as the response body.
*   **CORS Plugin:** Adds its own headers (`Access-Control-Allow-Origin: *`) if configured, ensuring the browser accepts the response.

⬇️ *Kong sends HTTP Response to Nginx*

**3. Nginx (Reverse Proxy)**
*   Receives the HTTP 200 OK from `127.0.0.1:8000`.
*   Forwards it back to the ALB.

⬇️ *Nginx sends HTTP Response to ALB*

**4. AWS ALB**
*   Receives the response.
*   Forwards it to the internet/client.

⬇️ *ALB sends HTTP Response to Browser*

**5. Browser / UI Application**
*   Receives the final JSON:
    ```json
    {"users": [{"id": 1, "name": "Alice"}]}
    ```
*   **Success!**

---

## 6. Impossible URLs? Troubleshooting the "Invisible"

Since Kong invokes Lambda directly, **there is no "URL" between Kong and Lambda.**

If you look at Kong logs, you won't see `POST http://lambda-url...`.
You will see: `[aws-lambda] invoking function: kong-poc-lambda`.

### How to Debug this "Invisible" Link?

If your UI gets a `500 Internal Server Error`, who failed? Kong? Or Lambda?

**Check 1: Kong Logs (The Initiator)**
*   Command: `docker logs kong-cp`
*   Look for: `[aws-lambda]`.
*   *Error Example:* `Connect timeout` (Kong couldn't reach AWS Lambda Service).
*   *Error Example:* `AccessDeniedException` (EC2 Role doesn't have `lambda:InvokeFunction`).

**Check 2: CloudWatch Logs (The Receiver)**
*   Go to **AWS Console -> CloudWatch -> Log Groups**.
*   Find `/aws/lambda/kong-poc-lambda`.
*   **If you see a log stream:** Kong successfully reached Lambda! The error is in your Python code.
*   **If no new logs:** Kong failed to reach Lambda (Check EC2 IAM Role or Security Groups).

### Summary Checklist for Success

1.  **EC2 IAM Role** has `AWSLambdaRole` policy? ✅
2.  **Security Group** allows Outbound 443 (HTTPS) to AWS? ✅
3.  **Kong Plugin** has `aws_key` and `aws_secret` EMPTY? (So it uses the Role) ✅
4.  **Kong Plugin** has `function_name` exactly matching Lambda? ✅
5.  **Kong Plugin** has `region` correct (`ap-south-1`)? ✅
