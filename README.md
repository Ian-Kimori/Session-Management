# Session-Management

This is the definitive, step-by-step validation guide for your session management audit. For a clean report, each test must have a clear **Action**, **Tool**, and **Evidence** for the result.

---

### 1. Token Randomness & Forgery (Bypass Schema)
* **The Component:** The statistical unpredictability of the `sessionid`.
* **The Tools:** Burp Sequencer.
* **Where FIPS comes in:** **FIPS 140-2** is a security standard that defines several statistical tests (Monobit, Poker, Runs). Sequencer runs these to see if your `sessionid` follows any mathematical pattern.
* **The Steps:**
    1.  Capture the login response in Burp Proxy.
    2.  Right-click the `zbx_session` cookie -> **Send to Sequencer**.
    3.  Select the `sessionid` part of the token and click **Start Live Capture**.
    4.  Analyze after 1,000+ samples.
* **Validation:** * **Pass:** "Overall quality: Excellent." Effective entropy > 100 bits.
    * **Fail:** "Significant" patterns found or entropy < 64 bits.


---

### 2. Cookie Attributes (`HttpOnly` & `Secure`)
* **The Component:** Flags that protect the cookie from being accessed by scripts or sent over plain text.
* **The Tools:** Burp Proxy / Browser DevTools.
* **The Steps:** 1.  Log in and look at the `Set-Cookie` header in the response.
* **Validation:**
    * **Pass:** The header includes both `HttpOnly` and `Secure`.
    * **Fail:** If `Secure` is missing (common in local environments) or `HttpOnly` is missing. 

---

### 3. Session Fixation
* **The Component:** ID rotation during the authentication "flow."
* **The Tools:** Browser DevTools (Application Tab).
* **The Steps:**
    1.  Open the login page. Note the `sessionid` before logging in.
    2.  Enter credentials and log in.
    3.  Check the `sessionid` again.
* **Validation:**
    * **Pass:** The cookie value changes completely after login.
    * **Fail:** The cookie value remains the same. 

---

### 4. Exposed Variables & Encryption (Manipulation)
* **The Component:** Data integrity within the JSON structure.
* **The Tools:** Burp Decoder & Repeater.
* **The Steps:**
    1.  Take your Base64 cookie and decode it in **Decoder**.
    2.  Modify a non-signature field (e.g., change a user ID or a timestamp).
    3.  Re-encode and send the request in **Repeater**.
* **Validation:**
    * **Pass:** The server returns an error or forces a logout (Signature invalid).
    * **Fail:** The server accepts the request (Data manipulation possible).

---

### 5. Cross-Site Request Forgery (CSRF)
* **The Component:** Origin validation for sensitive actions.
* **The Tools:** Burp CSRF PoC Generator.
* **The Steps:**
    1.  Capture a POST request (e.g., "Add User").
    2.  Remove the `sid` or `auth` parameter from the body.
* **Validation:**
    * **Pass:** Server rejects with `403 Forbidden`.
    * **Fail:** Server executes the action using only the cookie.


---

### 6. Logout & Hijacking (Server-Side Kill)
* **The Component:** Immediate destruction of the session in the server database.
* **The Tools:** Burp Repeater.
* **The Steps:**
    1.  Log in and capture a dashboard request in Repeater.
    2.  Log out in the browser.
    3.  Resend the request in Repeater.
* **Validation:**
    * **Pass:** `401 Unauthorized`.
    * **Fail:** `200 OK` (This is your current result: the session is still active on the server).


---

### 7. Timeouts (Hard & Idle)
* **The Component:** Temporal expiration (15 minutes).
* **The Tools:** Stopwatch & Burp Repeater.
* **The Steps:**
    1.  Capture an active request. Wait 16 minutes without using the app.
    2.  Resend the request.
* **Validation:**
    * **Pass:** Session is expired.
    * **Fail:** Request still works (Inactivity timeout failure).

---

### 8. Session Puzzling (Variable Overloading)
* **The Component:** Isolation of variables across modules.
* **The Tools:** Two Browser Tabs.
* **The Steps:**
    1.  Start a "Reset Password" flow in Tab 1 for User A.
    2.  Log in as User B in Tab 2.
    3.  Finish the flow in Tab 1.
* **Validation:**
    * **Pass:** Flow is isolated to User A.
    * **Fail:** Flow affects User B (Session pollution).

---

### 9. Concurrent Sessions (The "Single Login" Rule)
* **The Component:** Enforcing one active session per account.
* **The Tools:** Two different browsers.
* **The Steps:**
    1.  Log in on Chrome.
    2.  Log in as the same user on Firefox.
    3.  Refresh the page on Chrome.
* **Validation:**
    * **Pass:** Chrome is logged out.
    * **Fail:** Both sessions remain active.

---

### 10. Rate Limiting & Brute Force
* **The Component:** Defense against automated guessing.
* **The Tools:** Burp Intruder.
* **The Steps:** 1.  Send a login request to Intruder.
    2.  Attempt 50 failed logins in 10 seconds.
* **Validation:**
    * **Pass:** Server returns `429 Too Many Requests` or locks the account.
    * **Fail:** Server processes all 50 attempts (Vulnerable to Brute-force).

---

### Summary of Result Interpretation
* **Persistent vs. Session Cookie:** Check if the cookie has an `Expires` date. If it does, it's **Persistent**. If not, it's a **Session** cookie.
* **Audit Result:** Your application **Passes 1 and 3**, but **Fails 2, 6, 7, and 9**. This represents a significant risk where stolen sessions can be used indefinitely across different browsers without expiring.
