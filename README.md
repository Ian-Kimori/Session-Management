# Session-Management

# Session Management Audit

# Universal Web Application Methodology

Applies to any web application. Tested with Burp Suite Pro. All steps are self-contained and repeatable.

---

## Prerequisites

- Burp Suite configured as browser proxy
- A test account with known credentials
- A second test account (for tests 8 and 10)
- Browser with cookie editing capability (DevTools or extension)
- Two separate browser profiles available

---

## 1. Token Randomness and Entropy

**Goal:** Confirm session IDs cannot be guessed, predicted, or forged.

1. Log in and capture the session cookie from the login response in Burp HTTP History.
2. Right-click the login request → Send to Sequencer.
3. Set the token location to the exact cookie value field.
4. Click **Start live capture**. Collect a minimum of 2,000 tokens (5,000 preferred).
5. Let Burp auto-analyse. In the results, check the **Overall result** effective entropy — it must exceed 100 bits.
6. Review the FIPS 140-2 test results — all sub-tests should pass.
7. Separately, copy 5–10 raw token values. If they are Base64-encoded, decode them and inspect for structured data such as timestamps, sequential counters, or user IDs embedded in the payload.
8. Take one decoded token, modify a single byte (e.g. flip a digit in a user ID field), re-encode it, and inject it via Burp Repeater as your Cookie header.

**Pass:** Entropy exceeds 100 bits effective randomness, all FIPS tests pass, and the server rejects any manually modified token with 401 or 403.

**Fail:** Entropy is low or the token contains predictable structure, or the server accepts a modified token without rejecting it.

---

## 2. Cookie Security Attributes

**Goal:** Prevent session theft via XSS (HttpOnly) and network interception (Secure).

1. In Burp HTTP History, locate the POST login response — the one containing the `Set-Cookie` header.
2. Inspect the header for the `HttpOnly` flag. Its absence means JavaScript (e.g. via XSS) can read the cookie with `document.cookie`.
3. Inspect for the `Secure` flag. Its absence means the cookie is transmitted over plain HTTP connections.
4. Inspect for `SameSite=Lax` or `SameSite=Strict`. Its absence weakens CSRF protection for cross-origin requests.
5. If the application sets multiple cookies, repeat for every security-relevant one — not just session tracking cookies.
6. Confirm in browser DevTools (Application → Cookies) that all flags are reflected correctly.

**Pass:** All three flags — `HttpOnly`, `Secure`, and `SameSite=Lax` or `Strict` — are present on every session cookie.

**Fail:** Any flag is missing, particularly `HttpOnly` or `Secure` on a production HTTPS application.

---

## 3. Session Fixation

**Goal:** Confirm the server issues a brand-new session token upon successful authentication.

> You do not need to decode the cookie. You are comparing the raw cookie string value before and after login. If the raw value changes, the server issued a new token. Decoding is only relevant for entropy analysis (test 1) and payload inspection (test 4).

1. Open the application in a fresh browser tab with no existing session. Navigate to the login page.
2. In Burp HTTP History, identify the earliest server response that sets a session cookie. Note the exact raw cookie value — this is your pre-authentication token.
3. Log in with valid credentials and observe the POST login response in Burp.
4. Check whether the `Set-Cookie` header in the login response issues a new cookie value, or whether no new cookie is set (implying the old one is reused).
5. Compare the raw cookie string from step 2 against what is now active. If they are identical, the server has not rotated the token.
6. **Advanced:** Before logging in, manually plant a custom session ID in the cookie via DevTools or Burp (e.g. set `sessionid=ATTACKER_CHOSEN_VALUE`). Submit the login form. If the server accepts and continues using your planted value post-authentication, session fixation is confirmed.

**Pass:** The raw session token value is completely different after login compared to before.

**Fail:** The same token from the pre-login phase is still active post-authentication. An attacker who pre-sets a known session ID can hijack the session after the victim logs in.

---

## 4. Exposed Session Variables

**Goal:** Ensure the session token is confined to the Cookie header and does not leak elsewhere.

1. After logging in, navigate through 8–10 distinct pages — profile, settings, admin panels, dashboards, report views.
2. At each page, inspect the URL bar for session token or user identifier appearing as a query parameter (e.g. `?sid=...`, `?user_id=...`, `?token=...`).
3. In Burp HTTP History, check the `Referer` header on outgoing requests to any third-party resources — CDN assets, analytics scripts, external fonts. Verify the session token does not appear in the Referer URL.
4. In browser DevTools → Application, check `localStorage`, `sessionStorage`, and `IndexedDB` for any session token values.
5. Decode the cookie payload if it is Base64 or JWT-encoded. Verify it does not contain cleartext sensitive data such as passwords, internal IP addresses, or plaintext role names.
6. In Burp HTTP History, search for the session token value across all recorded requests to confirm it only ever travels in the `Cookie` header.

**Pass:** Token appears exclusively in the `Cookie` header. No exposure in URLs, Referer headers, or client-side storage.

**Fail:** Token appears in any URL parameter, leaks to third parties via Referer, or is stored in localStorage.

---

## 5. Cross-Site Request Forgery (CSRF)

**Goal:** Confirm that state-changing actions require a valid, server-validated CSRF token.

1. Identify a state-changing POST request — e.g. updating a profile, deleting a record, changing a password. Capture it in Burp.
2. Send the request to Burp Repeater. Remove the CSRF token parameter entirely from the request body (e.g. delete `sid=XXXX` or `_csrf=XXXX`). Send the request.
3. In a separate Repeater tab, keep the CSRF token present but change the `Origin` header to `https://evil.com`. Send the request.
4. In another tab, keep the CSRF token present but change the `Referer` header to `https://evil.com/attack.html`. Send the request.
5. If any state-changing action is reachable via GET request, test it from an unauthenticated tab with no CSRF token.
6. Use Burp's CSRF PoC generator (right-click → Engagement tools → Generate CSRF PoC) to produce a self-submitting HTML form. Open it in a second browser profile that is logged in as the victim and observe whether the action executes.

**Pass:** Server returns 403 Forbidden when the CSRF token is absent or when the Origin header is untrusted.

**Fail:** The action completes with a missing or invalid CSRF token — the server is relying solely on the session cookie for authorisation.

---

## 6. Logout and Server-Side Invalidation

**Goal:** Confirm the session is destroyed server-side upon logout, not merely cleared in the browser.

1. Log in and capture the active session cookie value from Burp HTTP History.
2. Copy the raw cookie string in full.
3. Click **Logout** in the application UI. Confirm you are redirected to the login page.
4. Open Burp Repeater. Paste a request to a protected resource (dashboard, admin panel, API endpoint) and inject the copied cookie in the `Cookie` header.
5. Send the request. Observe the response status and body.
6. Additionally, open a completely fresh browser profile (no cookies), manually inject the copied cookie, and attempt to access a protected page. This removes any possibility the browser is contributing to the block.

**Pass:** Server returns 401 or 403, or redirects to the login page. The old token is completely invalidated server-side.

**Fail:** Protected content loads successfully with the post-logout cookie. The session exists indefinitely on the server regardless of logout.

---

## 7. Inactivity Timeout

**Goal:** Confirm sessions are automatically terminated after a defined period of inactivity.

1. Log in and note your session cookie value.
2. Leave the browser completely idle — no clicks, no scrolling, no background requests. Wait for the documented idle timeout period (commonly 15–30 minutes).
3. After the timeout period has elapsed, do not interact with the browser. Instead, use Burp Repeater directly to send a credentialed request to a protected resource — this bypasses any browser-side re-authentication prompt.
4. Observe the response. A 401 or login redirect confirms the timeout fired.
5. If the application sends periodic keep-alive AJAX requests, use Burp's Match and Replace or a Proxy rule to block them. Retest to confirm suppressing keep-alives allows the timeout to trigger.
6. Document the actual observed timeout duration. If it exceeds the security policy (e.g. policy states 15 minutes but the session remains valid for hours or days), record the discrepancy as a finding.

**Pass:** After the inactivity period, any request with the session cookie returns 401 or redirects to login.

**Fail:** Session remains active beyond the documented policy timeout, particularly if it never expires.

---

## 8. Session Puzzling / Variable Overloading

**Goal:** Confirm the server resolves action targets from explicit request parameters, not from a shared session variable that can be overwritten by concurrent activity.

1. Open two browser tabs in the same session (same browser profile, same cookies).
2. In Tab A: begin a sensitive multi-step action — e.g. editing User A's profile, initiating a password reset for User A, or starting a mass update on Host Group A. Do **not** complete the action yet.
3. In Tab B: navigate to a related but different context — e.g. open User B's profile, or navigate to Host Group B's mass update screen. Allow the page to fully load.
4. Switch back to Tab A. Enable Burp intercept. Complete the original action (click Submit or Save).
5. In the intercepted POST body, identify which user or object ID is present — does it reference Tab A's target or has it been replaced by Tab B's?
6. Forward the request. In the application, verify which user or object was actually modified.

**Pass:** Action applies to the object explicitly named in Tab A's POST body. The server uses request-body IDs as the authoritative source.

**Fail:** Action applies to Tab B's last-loaded object. The server resolved the target from a session variable that was silently overwritten by Tab B's navigation.

---

## 9. Session Hijacking / Environment Binding

**Goal:** Determine whether a stolen cookie is usable from a different browser or IP address.

1. Log in from your primary browser and copy the active session cookie from Burp.
2. Open a completely different browser (e.g. Firefox if you used Chrome) — this changes the User-Agent string.
3. Inject the copied cookie into the new browser via DevTools (Application → Cookies → manually add the name/value pair) or a cookie editor extension.
4. Navigate to a protected page. Note whether access is granted.
5. Separately, repeat from the same browser but with a different source IP — use a VPN, mobile hotspot, or configure a Burp upstream proxy to route through another address.
6. Observe whether the server detects the environment change and invalidates the session.

**Pass:** Server detects the changed User-Agent or IP and forces re-authentication.

**Fail:** Session is fully accepted from the new environment. Note that this outcome is expected on most applications — IP binding breaks legitimate mobile users. Document it as a compensating-control gap rather than a standalone critical finding, and note whether sensitive operations require step-up authentication.

---

## 10. Concurrent Session Control

**Goal:** Confirm whether the application enforces a single active session per user.

1. Log in as a test user from Browser Profile 1. Note the session cookie.
2. In a completely separate browser profile (or an incognito window), log in as the same user. This establishes a second concurrent session.
3. Return to Browser Profile 1. Send any request to a protected resource — either by refreshing the page or replaying via Burp Repeater.
4. Observe whether Browser Profile 1's session was invalidated by the second login.
5. Repeat in reverse — log out from Browser Profile 2 and check whether Browser Profile 1 is also affected.
6. Document the total number of concurrent sessions the application permits, and whether the user receives any notification such as "You were signed out because a new login was detected."

**Pass:** The first session is automatically terminated when a second login occurs for the same account.

**Fail:** Both sessions remain active simultaneously — no concurrent session enforcement is in place.

---

## Quick Reference

| # | Test | Primary Tool | Key Evidence |
|---|------|-------------|--------------|
| 1 | Token randomness | Burp Sequencer | Entropy > 100 bits |
| 2 | Cookie attributes | Burp HTTP History | HttpOnly, Secure, SameSite in Set-Cookie |
| 3 | Session fixation | Browser DevTools + Burp | Raw cookie value changes on login |
| 4 | Variable exposure | Burp Search + DevTools | Token only in Cookie header |
| 5 | CSRF | Burp Repeater + PoC generator | 403 on missing/invalid token |
| 6 | Logout invalidation | Burp Repeater | 401 after logout with old cookie |
| 7 | Inactivity timeout | Burp Repeater + timer | 401 after idle period |
| 8 | Session puzzling | Two browser tabs + Burp | POST body IDs match action target |
| 9 | Hijacking / binding | Second browser + VPN | Session rejected in new environment |
| 10 | Concurrent sessions | Two browser profiles | First session killed by second login |
