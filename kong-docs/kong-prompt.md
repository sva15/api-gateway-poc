### **Context**

I want to design and implement a **production-grade Proof of Concept (POC)** using **Kong Gateway** as an API management and security layer in front of **AWS Lambda–based APIs**, replacing AWS API Gateway due to internal constraints.

My goal is to understand the **recommended architecture**, **request flow**, and **exact Kong configurations** required for this setup.

---

## **Current AWS Setup**

### **Lambda Setup**

* All Lambda functions are deployed inside **VPC private subnets**
* No public access
* No Lambda Function URLs enabled

  * If Function URLs are used, they **must use AWS IAM authentication**
* Lambda functions are written using **Flask**

  * Each Lambda already internally handles **multiple subpaths and HTTP methods**

---

### **AWS ALB Setup**

I am already using **AWS Internal ALB** with **multiple target group types**:

#### **1. Lambda-based Target Groups**

* ALB routes requests to Lambda using path-based routing
* Example routes:

  * `/dev/api1/*`
  * `/dev/api2/*`
* This setup is currently working

#### **2. IP-based Target Groups**

* ALB forwards traffic to an EC2 instance where:

  * Kong Gateway is installed
  * NGINX runs on port `80`
  * Angular application runs on port `4302`

---

### **Angular UI Flow (Current State)**

* Angular app is accessed via:

  ```
  http://<alb-url>/dev/configurator/*
  ```
* ALB → IP-based target group → NGINX → Angular app (`:4302`)
* Angular app directly calls Lambda APIs via:

  * ALB Lambda target group routes
* Since **UI and APIs share the same ALB hostname**, **CORS is not required**
* This setup works perfectly today

---

## **Proposed Change: Introduce Kong Gateway**

I want to introduce **Kong Gateway** between the **Angular UI** and the **Lambda APIs**.

### **Planned Kong Placement**

* Kong will be deployed on the **same EC2 instance**
* Kong will be placed **behind the same AWS ALB**
* ALB → IP-based target group → NGINX → Kong
* Kong endpoints accessed via browser like:

  ```
  http://<alb-url>/kong/
  ```
* NGINX will reverse proxy:

  * `/kong/*` → `http://private-ip:8002` (Kong Manager / Admin GUI)
  * other Kong proxy ports as required

---

## **Primary Goals**

1. **Add security and governance** to APIs
2. Avoid AWS API Gateway
3. Use **Kong Gateway as API Manager**
4. Support both:

   * Coarse-grained routing (proxy-style)
   * Fine-grained routing (method + schema validation)

---

## **Key Questions & Requirements**

### **1️⃣ Lambda Invocation Strategy**

Given:

* Lambdas are private
* No public Function URLs
* IAM auth required if Function URLs are enabled

**What is the recommended way for Kong to invoke Lambda?**

* Kong AWS Lambda Plugin?
* ALB → Lambda target group?
* Private Function URLs with IAM auth?
* VPC networking considerations?

Explain **which approach is recommended and why**, including **security implications**.

---

### **2️⃣ Kong Configuration (Very Important)**

For the selected architecture, explain **in detail** how to configure:

#### **Kong Gateway**

* Gateway Services
* Routes
* Consumers
* Plugins:

  * Key Auth
  * CORS
  * AWS Lambda plugin (if applicable)
  * Request Validator (schema validation)

---

### **3️⃣ Routing Strategy (Hybrid Requirement)**

I want **two routing models to coexist**:

#### **A. Proxy-style routing**

Similar to AWS API Gateway `{proxy+}`:

* Single Kong route
* Any HTTP method
* Path like:

  ```
  /api/dev/*
  ```
* Lambda internally handles:

  * Subpaths
  * Methods
  * Business logic

Explain:

* Whether Kong supports this cleanly
* Best practices
* Tradeoffs

---

#### **B. Fine-grained routing**

* Individual routes per API path
* Explicit HTTP methods
* Header validation
* Query parameter validation
* Request body schema validation

Explain:

* How to model this in Kong
* How schema validation works
* How this impacts Lambda invocation
* How to version APIs cleanly

---

### **4️⃣ Request Validation & Control**

One concern:

* If requests are directly forwarded from Kong → ALB → Lambda
* Then **Kong cannot fully validate schema per method** unless routes are split

Please explain:

* Where validation should happen
* Whether ALB-based Lambda routing limits Kong’s control
* Recommended pattern to maintain **full request governance**

---

### **5️⃣ End-to-End Request Flow**

Please provide:

* A **clear end-to-end request flow**
* From browser → ALB → NGINX → Kong → Lambda
* Include:

  * Headers
  * Auth
  * IAM role usage
  * Network path

---

## **Expected Output**

Please provide:

* **Architecture diagram (described in text)**
* **Recommended design**
* **Why alternatives were rejected**
* **Step-by-step Kong configuration**
* **Best practices for production**
* **Security considerations**
* **Clear pros & cons**

Assume this is for a **serious POC that may go to production**.

---
