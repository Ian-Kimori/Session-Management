# Session-Management

To professionally audit these session management controls, you must validate both the **cryptographic strength** of the tokens and the **logic** of the server's session lifecycle.

Below is the clean, step-by-step methodology for each requirement based on your specific application environment.

---

### 1. Bypass Session Schema & Token Randomness
**Goal:** Ensure session IDs cannot be guessed or forged by manipulating unsigned data.
1.  **Gather Samples:** Use **Burp Sequencer** to capture 2,000+ session tokens (e.g., `sessionid` within your `zbx_session` cookie).
2.  **Statistical Analysis:** Analyze the samples to ensure they pass **FIPS 140-2** tests for randomness. High entropy (typically >100 bits) is required.
3.  **Cookie Manipulation:** Decode the Base64 cookie payload. Modify a variable (like a UserID or timestamp), re-encode it, and send it back to the server.
*   **Validation:**
      **PASS** if the server rejects the forged cookie due to a signature mismatch.
      **FAIL** if the server accepts the modified data.

### 2. Cookie Attributes (`HttpOnly` & `Secure`)
**Goal:** Prevent session theft via XSS (HttpOnly) or network sniffing (Secure).
1.  **Capture Response:** Observe the `Set-Cookie` header in your HTTP history during login.
2.  **Verify Flags:** Look for the specific strings `; HttpOnly` and `; Secure`.
*   **Validation:**
      **PASS** if both flags are present.
      **FAIL** if either is missing, especially on an HTTPS production site.

### 3. Session Fixation
**Goal:** Ensure the server issues a brand-new ID upon successful authentication.
1.  **Establish Pre-Auth ID:** Visit the login page but do not log in. Note the `sessionid`.
2.  **Authenticate:** Log in with valid credentials.
3.  **Compare IDs:** Check the `sessionid` again.
*   **Validation:**
      **PASS** if the ID changed completely.
      **FAIL** if the same ID from the login page is maintained after authentication.

### 4. Exposed Session Variables
**Goal:** Ensure sensitive identifiers aren't leaked in URLs, headers, or local storage.
1.  **URL Check:** Click through several pages and check if the session ID appears in the URL (e.g., `?sid=...`).
2.  **Header Check:** Check the `Referer` header in outgoing requests to ensure the ID isn't leaked to third-party domains.
3.  **Payload Check:** Decode the cookie and ensure it doesn't contain plaintext sensitive data like passwords or internal IP addresses.
*   **Validation:**
      **PASS** if the ID is strictly confined to the `Cookie` header and payload is encrypted/signed.

### 5. Cross-Site Request Forgery (CSRF)
**Goal:** Ensure attackers cannot force a user's browser to perform actions (e.g., "Delete Host").
1.  **Identify Action:** Find a POST request that changes data (e.g., updating user settings).
2.  **Test Token Requirement:** Use **Burp Repeater** to send the request without the `sid` or CSRF token parameter.
3.  **Verify Origin:** Change the `Origin` header to a different domain.
*   **Validation:**
      **PASS** if the server returns a `403 Forbidden`.
      **FAIL** if the action completes based solely on the session cookie.

### 6. Logout Functionality
**Goal:** Ensure the session is destroyed on the server, not just in the browser.
1.  **Copy Token:** Capture a valid, active `zbx_session` cookie.
2.  **Perform Logout:** Click "Logout" in the UI.
3.  **Replay Attack:** In **Burp Repeater**, use the "old" cookie to request a private dashboard.
*   **Validation:**
      **PASS** if the server returns `401 Unauthorized`.
      **FAIL** if the dashboard still loads (indicating the session is still "alive" on the server).

### 7. Hard Session Timeout (15-Minute Inactivity)
**Goal:** Automatically terminate sessions left unattended.
1.  **Set Idle Time:** Log in and wait exactly 16 minutes without clicking anything.
2.  **Resume Activity:** Refresh the page or resend a request.
*   **Validation:**
      **PASS** if you are redirected to the login screen.
      **FAIL** if the session is still active (e.g., your 5-day persistence finding).

### 8. Session Variable Overloading (Session Puzzling)
**Goal:** Ensure variables from different application modules don't bleed into each other.
1.  **Multi-Module Action:** Start a sensitive process in Tab A (e.g., Password Reset).
2.  **Context Switch:** In Tab B, perform a different action (e.g., viewing a different user's profile).
3.  **Complete Original Action:** Finish the process in Tab A.
*   **Validation:**
      **PASS** if the action completes correctly for the intended user/target.
      **FAIL** if Tab A's action "puzzles" with the data from Tab B.

### 9. Session Hijacking (Environment Binding)
**Goal:** Prevent a stolen cookie from being used in a different environment.
1.  **Capture Token:** Copy your active cookie.
2.  **Switch Context:** Open a different browser (e.g., Firefox) or change your IP (e.g., using a VPN).
3.  **Injected Access:** Manually add the cookie to the new browser and attempt to access the app.
*   **Validation:**
      **PASS** if access is denied because the server detects a change in IP or User-Agent.
      **FAIL** if you gain access (Hijacking successful).

### 10. Concurrent User Sessions
**Goal:** Enforce a "Single Sign-on" rule where one login kills previous logins.
1.  **Initial Login:** Log in on Browser 1.
2.  **Concurrent Login:** Log in as the same user on Browser 2.
3.  **Verify Browser 1:** Refresh the page on Browser 1.
*   **Validation:**
      **PASS** if Browser 1 is automatically logged out.
      **FAIL** if both sessions remain active simultaneously.
