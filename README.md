# Session-Management

This is your comprehensive, step-by-step technical guide for each session management test case. For a "clean" audit, ensure you clear your browser cache/cookies between each major test section to avoid data contamination.

---

### 1. Bypass Session Schema & Token Randomness
* **Component:** Cryptographic strength of the session identifier.
* **Tool:** Burp Suite (Sequencer).
* **The Steps:**
    1.  Capture the login response containing the `Set-Cookie` header in Burp Proxy.
    2.  Right-click the request -> **Send to Sequencer**.
    3.  Select the cookie (e.g., `sessionid`). Click **Start Live Capture**.
    4.  Collect at least 1,000 tokens, then click **Analyze Now**.
* **Pass:** "Effective Entropy" > 100 bits.
* **Fail:** Low entropy (< 64 bits) or visible patterns (e.g., tokens contain timestamps or sequential numbers).


### 2. Cookie Attributes (`HttpOnly` and `Secure`)
* **Component:** Client-side protection flags.
* **Tool:** Burp Suite (Proxy History) or Browser DevTools.
* **The Steps:**
    1.  Log in and find the server's `Set-Cookie` header in the HTTP response.
    2.  Check for the presence of the `Secure` and `HttpOnly` flags.
* **Pass:** Header looks like: `Set-Cookie: ID=xyz; Secure; HttpOnly`.
* **Fail:** Flags are missing.


### 3. Session Fixation
* **Component:** ID rotation during authentication.
* **Tool:** Browser DevTools (Application Tab).
* **The Steps:**
    1.  Open the login page but **do not log in**. Record the current session cookie.
    2.  Log in with your credentials.
    3.  Check the cookie value again.
* **Pass:** The session ID is completely different after login.
* **Fail:** The session ID remains the same, meaning an attacker could "pre-set" an ID for a victim.

### 4. Exposed Session Variables (Encryption)
* **Component:** Data integrity and confidentiality within the cookie.
* **Tool:** Burp Decoder.
* **The Steps:**
    1.  Capture your session cookie. If it is Base64 (like your `eyJ...` example), decode it.
    2.  Try to change a value (e.g., change `admin: false` to `true`).
    3.  Re-encode and send the request.
* **Pass:** The server rejects the modified cookie because the signature (`sign`) is now invalid.
* **Fail:** The server accepts the modified data, granting unauthorized access or privileges.

### 5. Cross-Site Request Forgery (CSRF)
* **Component:** Request origin validation.
* **Tool:** Burp Suite (Generate CSRF PoC).
* **The Steps:**
    1.  Capture a sensitive request (e.g., "Change Password").
    2.  Check for a random token in the request body/header.
    3.  Attempt the request again while removing or changing the token.
* **Pass:** The server returns an error (e.g., `403 Forbidden`).
* **Fail:** The request succeeds without a valid CSRF token.


### 6. Logout Functionality & Server-Side Kill
* **Component:** Immediate session invalidation.
* **Tool:** Burp Suite (Repeater).
* **The Steps:**
    1.  Log in and capture a request to a private page. Send it to **Repeater**.
    2.  In the browser, click **Logout**.
    3.  In Repeater, click **Send** on that original request.
* **Pass:** Server returns a redirect to login or `401 Unauthorized`.
* **Fail:** Server returns `200 OK` (the session is still "alive" on the server).

### 7. Hard Session Timeout (15 mins)
* **Component:** Inactivity expiration.
* **Tool:** Stopwatch & Burp Repeater.
* **The Steps:**
    1.  Capture a valid authenticated request.
    2.  Wait exactly 16 minutes without performing any actions.
    3.  Resend the captured request.
* **Pass:** Request fails due to an expired session.
* **Fail:** Request still succeeds.

### 8. Session Variable Overloading (Puzzling)
* **Component:** Variable isolation.
* **Tool:** Two different browser tabs.
* **The Steps:**
    1.  Start a multi-step process (like a checkout or password reset) for Account A in Tab 1.
    2.  In Tab 2, log in as Account B.
    3.  Finish the process in Tab 1.
* **Pass:** The process completes for Account A only.
* **Fail:** The app uses Account B's information to complete Account A's process.

### 9. Session Hijacking (Replay)
* **Component:** Environment/Hardware binding.
* **Tool:** Two different machines (or a VPN).
* **The Steps:**
    1.  Log in on Machine 1 and copy the session cookie.
    2.  Paste that cookie into a browser on Machine 2 (different IP).
    3.  Refresh the page.
* **Pass:** Machine 2 is forced to log in.
* **Fail:** Machine 2 is instantly logged in as the user.


### 10. Concurrent Sessions
* **Component:** Single session enforcement.
* **Tool:** Two different browsers (Chrome and Firefox).
* **The Steps:**
    1.  Log in on Chrome.
    2.  While Chrome is open, log in with the **same** account on Firefox.
    3.  Go back to Chrome and refresh.
* **Pass:** Chrome is automatically logged out.
* **Fail:** Both sessions remain active.
