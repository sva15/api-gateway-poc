# Why Do We Need Both a Plugin AND a Consumer?

The confusion comes from thinking they are the same thing. They are not.

Think of it like a **Security Door**.

## 1. The Lock (Key Auth Plugin)
**Purpose:** Enforce Security (Say "STOP! Who goes there?")

*   When you add the **Key Auth Plugin** to a Service or Route, you are installing a **Lock** on the door. 🔒
*   The plugin's job is simple: **"Do not let anyone in unless they show me a valid API Key."**
*   If you *don't* add this plugin, the door is wide open. Anyone can walk in.

## 2. The Visitor's ID (Consumer + Key)
**Purpose:** Prove Identity (Say "It's me, Frontend App. Here is my key.")

*   A **Consumer** represents a specific user or application (e.g., "Frontend App", "Mobile App", "Partner X").
*   The **Key Credential** (e.g., `abc-123-secret`) is the actual physical key in their pocket. 🔑
*   When a request comes in with `x-api-key: abc-123-secret`, Kong looks at its list of Consumers: *"Does anyone have this key in their pocket?"*
*   **Yes!** It belongs to "Frontend App". Therefore, "Frontend App" is allowed to pass through the **Lock**.

---

## 3. The Lambda Plugin (The Destination)
**Purpose:** Do the work (Execute code).

*   The Lambda Plugin is like the **Worker inside the room**. 👨‍💻
*   It doesn't care about keys or locks. It just does the work when someone enters the room.
*   **Crucial:** You do **NOT** need to configure a Consumer inside the Lambda Plugin settings.
    *   The Lambda Plugin settings just say: *"Which function should I run?"* (`kong-poc-lambda`)
    *   It assumes that if the request reached it, the **Lock** (Key Auth Plugin) must have already done its job.

---

## 4. How They Connect (The Flow)

You configure them separately, but they work together in a chain.

### Configuration Steps
1.  **Service/Route:** You create the path to the room (`/users`).
2.  **Key Auth Plugin (The Lock):** You attach this to the **Route**. *"Please lock this door."*
3.  **Consumer (The User):** You create `frontend-app` and give it key `secret-123`. *"Here is a key for our trusted user."*
4.  **Lambda Plugin (The Worker):** You attach this to the **Route**. *"When someone enters, run this code."*

### Execution Steps (When a Request Arrives)

1.  **Request:** `GET /users` with Header `x-api-key: secret-123`
2.  **Plugin Check 1 (Key Auth):**
    *   Kong sees the **Lock**.
    *   Kong checks: *"Is `secret-123` valid?"*
    *   Kong finds the **Consumer** `frontend-app` owns this key.
    *   **Result:** ✅ Access Granted. The door opens.
3.  **Plugin Check 2 (Lambda):**
    *   Kong sees the **Worker** instructions.
    *   Kong invokes AWS Lambda `kong-poc-lambda`.
    *   **Result:** 🚀 Code runs.

---

## summary
*   **Key Auth Plugin** = The Rule ("You need a key").
*   **Consumer** = The Identity ("I am User X and I have a key").
*   **Lambda Plugin** = The Action ("Run this code").

You need **BOTH** the Plugin (to enforce the rule) and the Consumer (to be the valid user).
You do **NOT** link them directly in the Lambda configuration. They are linked by the request flow.
