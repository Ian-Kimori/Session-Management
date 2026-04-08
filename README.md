# Session-Management

To test these professionally, you must combine **automated statistical analysis** (Sequencer) with **manual logic testing** (Repeater). Using your Zabbix request as the baseline, here is the professional execution roadmap.

---

### 1. Token Randomness & Forgery (Bypass Schema)
* **The Goal:** Prove the `sessionid` cannot be predicted.
* **Tool:** **Burp Sequencer**.
* **Method:** Capture your request. Right-click the `zbx_session` cookie -> **Send to Sequencer**. Highlight only the `sessionid` value. Run for **1,500+ samples**.
* **Validation:**
    * **PASS:** Entropy > 100 bits. FIPS 140-2 tests show "Excellent" randomness.
    * **FAIL:** Significant bit-level patterns found.


---

### 2. Cookie Attributes (HttpOnly & Secure)
* **The Goal:** Ensure session data isn't leaked via XSS or non-HTTPS channels.
* **Tool:** **Burp Proxy** (HTTP History).
* **Method:** Examine the `Set-Cookie` header in the server's response.
* **Validation:**
    * **PASS:** Both `HttpOnly` and `Secure` flags are present.
    * **FAIL:** Either flag is missing (Note: Your previous Zabbix response was missing `Secure`).

---

### 3. Session Fixation
* **The Goal:** Ensure the server forces a new ID upon login.
* **Tool:** **DevTools** (Application Tab).
* **Method:**
  1.  Note the `sessionid` on the login page (pre-auth).
  2.  Log in.
  3.  Compare the new `sessionid` to the pre-auth value.
* **Validation:**
    * **PASS:** The value changes completely.
    * **FAIL:** The ID remains the same, allowing an attacker to "fix" a victim's ID.

---

### 4. Exposed Variables & Encryption
* **The Goal:** Ensure the `sign` (signature) prevents data tampering.
* **Tool:** **Burp Decoder** & **Repeater**.
* **Method:** Decode your Base64 cookie. Change a value (e.g., `serverCheckResult`: `true` to `false`). Re-encode and send.
* **Validation:**
    * **PASS:** Server returns an error (Signature Mismatch).
    * **FAIL:** Server accepts the modified data.

---

### 5. Cross-Site Request Forgery (CSRF)
* **The Goal:** Ensure sensitive actions (like "Delete Host") require a unique token.
* **Tool:** **Burp CSRF PoC Generator**.
* **Method:** Generate a PoC for your `widget.tophosts.view` request. Try removing any body parameters like `sid` or `auth`.
* **Validation:**
    * **PASS:** Server returns `403 Forbidden`.
    * **FAIL:** The request is processed successfully.


---

### 6. Logout & Session Hijacking (The "Kill" Test)
* **The Goal:** Ensure the session is destroyed on the server, not just the browser.
* **Tool:** **Burp Repeater**.
* **Method:** 1.  Copy your `zbx_session` while logged in.
    2.  Log out in the browser. 
    3.  Replay the request in Repeater using the old cookie.
* **Validation:**
    * **PASS:** Server returns `401 Unauthorized`.
    * **FAIL:** Server returns `200 OK` (Your current finding).


---

### 7. Hard Session Timeout (15 Minutes)
* **The Goal:** Enforce a strict inactivity limit.
* **Tool:** **Stopwatch** & **Repeater**.
* **Method:** Leave the session idle for 16 minutes. Resend the request.
* **Validation:**
    * **PASS:** Session is expired/invalid.
    * **FAIL:** Request still works (Your "5-day persistence" finding).

---

### 8. Session Variable Overloading (Puzzling)
* **The Goal:** Ensure variables from one module don't leak into another.
* **Tool:** **Two Browser Tabs**.
* **Method:** Start a sensitive action (e.g., "Change Password") in Tab 1. In Tab 2, navigate to a different user's profile. Finish the action in Tab 1.
* **Validation:**
    * **PASS:** The action only affects the original user.
    * **FAIL:** The application gets confused and applies the action to the user from Tab 2.

---

### 9. Multiple Concurrent Sessions
* **The Goal:** Prevent a single account from having multiple active sessions.
* **Tool:** **Chrome** and **Firefox**.
* **Method:** Log in on Chrome. Then log in as the same user on Firefox. Go back to Chrome and refresh.
* **Validation:**
    * **PASS:** Chrome is logged out.
    * **FAIL:** Both sessions remain active (This is how your "Ghost Tokens" were able to work).

---

### Professional Documentation Summary

| Test | Tool | Pass Evidence | Fail Evidence |
| :--- | :--- | :--- | :--- |
| **Randomness** | Sequencer | FIPS 140-2 Excellent | Significant Patterns |
| **Logout** | Repeater | 401 Unauthorized | 200 OK (Access) |
| **Timeout** | Repeater | Reject after 15 mins | 200 OK after 5 days |
| **Attributes** | Proxy | `Secure; HttpOnly` | Flag Missing |
| **CSRF** | PoC Gen | 403 Forbidden | Action Completed |

Since you have already confirmed the **Logout Kill** and **Timeout** tests are failing, you have a solid "High" severity finding for your report. Use the "Ghost Token" explanation to prove that the server relies on signatures rather than state.
