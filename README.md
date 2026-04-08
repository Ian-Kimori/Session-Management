# Session-Management

To test these requirements professionally on a Zabbix-like infrastructure, you should follow this structured methodology. We will use the Zabbix `zbx_session` cookie (which contains a Base64-encoded JSON object and a signature) as our primary target.

---

### 1. Bypass Session Schema & Token Randomness
* **The Goal:** Ensure the `sessionid` isn't predictable and the `sign` prevents forgery.
* **Step-by-Step:**
    1.  **Sequencer Test:** Capture the login response in Burp. Right-click the `zbx_session` -> **Send to Sequencer**. Highlight the `sessionid` value. Collect 2,000+ samples.
    2.  **Forgery Test:** In **Burp Decoder**, decode your cookie. Change a value (e.g., `serverCheckResult: true` to `false`). Re-encode it and try to use it in **Repeater**.
* **Validation:** * **PASS:** Sequencer shows "Excellent" entropy (>100 bits). The server rejects the modified cookie (Signature Mismatch).
    * **FAIL:** Entropy is low (predictable) or the server accepts the modified data.



---

### 2. Cookie Attributes (`HttpOnly` & `Secure`)
* **The Goal:** Ensure the cookie cannot be stolen via Javascript (XSS) or unencrypted connections.
* **Step-by-Step:**
    1.  Log in and find the `Set-Cookie` header in the **Burp Proxy** history.
* **Validation:**
    * **PASS:** Header contains `HttpOnly; Secure`.
    * **FAIL:** If `Secure` is missing (risky over HTTPS) or `HttpOnly` is missing.

---

### 3. Session Fixation
* **The Goal:** Ensure the session ID rotates during the transition from "Guest" to "Authenticated."
* **Step-by-Step:**
    1.  Go to the Zabbix login page. Note the `sessionid` in your browser's DevTools.
    2.  Log in with valid credentials.
    3.  Check the `sessionid` again.
* **Validation:**
    * **PASS:** The value is completely different after login.
    * **FAIL:** The value remains identical, meaning an attacker can "fix" a session ID for a victim.

---

### 4. Exposed Session Variables & Hijacking
* **The Goal:** Check if sensitive data is leaked in the URL or if the session is bound to the user's environment.
* **Step-by-Step:**
    1.  **Exposed Variables:** Navigate through Zabbix. Check if the `sessionid` appears in the URL (GET parameters).
    2.  **Hijacking/Binding:** Copy your `zbx_session` cookie. Open a second browser (e.g., Firefox) or use a VPN to change your IP. Try to access the dashboard by pasting the cookie.
* **Validation:**
    * **PASS:** Session is only in the cookie. Access is denied when the IP/User-Agent changes (Environment Binding).
    * **FAIL:** Session ID is in the URL. Access is granted from a different IP/Machine (Hijacking possible).

---

### 5. Cross-Site Request Forgery (CSRF)
* **The Goal:** Ensure state-changing actions (like disabling a host) require more than just a cookie.
* **Step-by-Step:**
    1.  Go to **Data collection > Hosts**. Click "Disable" on a host.
    2.  In **Burp Repeater**, locate the `sid` parameter in the JSON body or URL.
    3.  Delete the `sid` and send the request.
* **Validation:**
    * **PASS:** Server returns `403 Forbidden` or an "Incorrect SID" error.
    * **FAIL:** The action is performed successfully.



---

### 6. Logout Functionality & Server-Side Invalidation
* **The Goal:** Ensure the session is "dead" on the server, not just deleted from the browser.
* **Step-by-Step:**
    1.  Capture a dashboard request in **Repeater**.
    2.  Click **Logout** in your browser.
    3.  Go back to **Repeater** and send the request again.
* **Validation:**
    * **PASS:** `401 Unauthorized` or `302 Redirect` to login.
    * **FAIL:** `200 OK` (This is your current finding: the session is still active on the server).

---

### 7. Hard Session Timeout (15 Minutes)
* **The Goal:** Enforce inactivity expiration.
* **Step-by-Step:**
    1.  Log in. Close the tab. Do not interact for 16 minutes.
    2.  Re-open the tab or replay the request in **Repeater**.
* **Validation:**
    * **PASS:** Session is expired; you are forced to re-login.
    * **FAIL:** Access is granted (Your "5-day persistence" finding).

---

### 8. Session Variable Overloading (Puzzling)
* **The Goal:** Ensure variables don't leak between different modules or users.
* **Step-by-Step:**
    1.  **Tab 1:** Open the "User Profile" page for User A.
    2.  **Tab 2:** Open the "Data collection" page for Host Group B.
    3.  In **Tab 1**, click "Save."
* **Validation:**
    * **PASS:** Only User A's profile is updated.
    * **FAIL:** The application mistakenly applies settings to the host group or crashes because variables were "puzzled."

---

### 9. Multiple Concurrent Sessions
* **The Goal:** Prevent a single account from being used in multiple locations simultaneously.
* **Step-by-Step:**
    1.  Log in on **Chrome**.
    2.  Log in as the *same user* on **Firefox**.
    3.  Refresh the page on **Chrome**.
* **Validation:**
    * **PASS:** The Chrome session is automatically terminated.
    * **FAIL:** Both browsers remain logged in at the same time.

---

### Summary Checklist for the Report

| Test Case | Method | Pass | Fail |
| :--- | :--- | :--- | :--- |
| **Randomness** | Sequencer | High Entropy | Patterns Found |
| **Logout** | Repeater | Reject after logout | Accept after logout |
| **Timeout** | Stopwatch | Inactive at 15m | Active at 5 days |
| **Attributes** | Proxy | `Secure; HttpOnly` | Flags missing |
| **Concurrent** | 2 Browsers | Kills old session | Multiple sessions |

**Next Step:** Since you have confirmed the **Logout** and **Timeout** tests are currently failing, focus your report on the **Server-Side State Management** failure. This is the root cause for both findings.
