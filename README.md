# Session-Management

To perform a professional security audit on session management, you should use **Burp Suite Professional/Community** as your primary tool. It allows you to pause, inspect, and modify traffic between your browser and the server.

Here is the end-to-end testing guide for each requirement.

---

### 1. Bypass Session Schema & Token Randomness
* **Goal:** Determine if session IDs are predictable or manually manipulatable.
* **Tools:** Burp Suite (Sequencer), Browser DevTools.
* **Steps:**
    1.  Open Burp Suite and capture the login response (`Set-Cookie`).
    2.  Right-click the request and select **"Send to Sequencer."**
    3.  In Sequencer, select the cookie name (e.g., `PHPSESSID` or `session_id`).
    4.  Click **"Start Live Capture"** and let it collect at least 500–1,000 tokens.
    5.  Analyze the results for "Effective Entropy."
* **Pass:** High entropy (completely random).
* **Fail:** Low entropy; patterns found; tokens are Base64 encoded IDs (e.g., `dXNlcl8xMDE=` which is `user_101`).

### 2. Cookie Attributes (`HttpOnly` and `Secure`)
* **Goal:** Ensure cookies cannot be stolen via JavaScript (XSS) or intercepted over non-HTTPS connections.
* **Tools:** Burp Suite (Proxy History) or Browser DevTools (Application tab).
* **Steps:**
    1.  Log in to the application.
    2.  In Burp, go to **Proxy > HTTP History**.
    3.  Find the `Set-Cookie` header in the server's response.
* **Pass:** Header looks like: `Set-Cookie: ID=123; Secure; HttpOnly; SameSite=Strict`.
* **Fail:** The `Secure` or `HttpOnly` flags are missing.

### 3. Session Fixation
* **Goal:** Check if the app reuses a pre-login session ID after authentication.
* **Tools:** Burp Suite or two different browsers.
* **Steps:**
    1.  Navigate to the login page but **do not log in**. 
    2.  Note the session cookie value (e.g., `ABC123`).
    3.  Enter your credentials and log in.
    4.  Check the session cookie value again.
* **Pass:** The cookie value changes to something new (e.g., `XYZ789`).
* **Fail:** The cookie remains `ABC123`.

### 4. Exposed Session Variables & Encryption
* **Goal:** Ensure sensitive data isn't stored in plain text in cookies or the URL.
* **Tools:** Burp Suite, Base64 Decoder.
* **Steps:**
    1.  Capture your session cookies.
    2.  If they look like gibberish, try to decode them (Base64, Hex).
    3.  Check if your username, role (e.g., `role=user`), or PII is inside the cookie.
* **Pass:** Cookies contain only an opaque, random string.
* **Fail:** Cookies contain `admin=false` (which you can change to `true`) or unencrypted user data.

### 5. Cross-Site Request Forgery (CSRF)
* **Goal:** Verify if an attacker can force your browser to perform actions without your consent.
* **Tools:** Burp Suite (Engagement Tools > Generate CSRF PoC).
* **Steps:**
    1.  Find a sensitive action (e.g., "Update Profile").
    2.  Capture that request in Burp.
    3.  Check if there is a random token in the body (e.g., `csrf_token=...`).
    4.  Attempt to remove the token or change one character and resubmit.
* **Pass:** Server returns `403 Forbidden` or an error.
* **Fail:** The request is successful even without the token or with a fake one.

### 6. Logout Functionality & Server-Side Kill
* **Goal:** Ensure the session is actually destroyed on the server, not just hidden from the browser.
* **Tools:** Burp Suite (Repeater).
* **Steps:**
    1.  Log in and capture a request to a private page (e.g., `/settings`).
    2.  Send that request to **Repeater**.
    3.  In your browser, click **Logout**.
    4.  Go back to Burp Repeater and click **Send** on that original request.
* **Pass:** The server returns a `302` redirect to login or `401 Unauthorized`.
* **Fail:** The server returns `200 OK` and the private data is still visible.

### 7. Hard Session Timeout (e.g., 15 mins)
* **Goal:** Ensure sessions expire after a set period of inactivity.
* **Tools:** A timer/stopwatch and Burp Repeater.
* **Steps:**
    1.  Log in and capture a request.
    2.  Do nothing for 16 minutes.
    3.  After the time is up, resubmit that request in Burp Repeater.
* **Pass:** Session is expired; you are forced to log in again.
* **Fail:** The session is still active after the timeout period.

### 8. Session Variable Overloading (Puzzling)
* **Goal:** Check if one part of the app can "confuse" another part's session variables.
* **Tools:** Burp Suite.
* **Steps:**
    1.  Start a "Forgot Password" flow for `User A`.
    2.  In a separate tab, log in as `User B`.
    3.  Go back to the "Forgot Password" tab and see if it now thinks you are `User B`.
* **Pass:** The flows remain isolated.
* **Fail:** Initiating one process changes the state of another authenticated process.

### 9. Session Hijacking
* **Goal:** Verify if a session can be used from a completely different environment.
* **Tools:** Two different machines (or a VM and a Host).
* **Steps:**
    1.  Log in on Machine 1. Copy the session cookie string.
    2.  On Machine 2, open the same URL.
    3.  Open DevTools > Application > Cookies and manually **inject** the cookie from Machine 1.
    4.  Refresh the page.
* **Pass:** The server detects the change in IP/User-Agent and blocks access (Ideal) or the session management is otherwise hardened.
* **Fail:** You are instantly logged in as the user from Machine 1.

### 10. Multiple Concurrent Sessions
* **Goal:** Ensure a single account cannot be logged in from multiple places at once.
* **Tools:** Two different browsers (Chrome and Firefox).
* **Steps:**
    1.  Log in on Chrome.
    2.  Log in on Firefox with the same credentials.
    3.  Refresh the page on Chrome.
* **Pass:** Chrome is logged out automatically (the newer session killed the old one).
* **Fail:** Both browsers remain logged in simultaneously.
