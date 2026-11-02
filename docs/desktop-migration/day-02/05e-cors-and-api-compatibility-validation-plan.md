# EduPulse ERP — CORS & API Compatibility Validation Plan
## Work Package 05E — Verification Test Matrix, Preflight Scenarios & Regression Gates

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 02)
**Target Platform:** Windows 10 / Windows 11 Enterprise (Tauri v2 + Spring Boot 3.3.4)
**Author:** Antigravity AI Engineering Assistant (Security Baseline)
**Date:** 2 November 2026
**Document Status:** Approved Quality Assurance & Verification Specification

---

## 1. Executive Summary & Verification Methodology

This validation plan provides a rigorous, executable test harness to guarantee that the Spring Boot 3.3.4 backend maintains full backward compatibility with the existing web application while providing seamless, secure connectivity for the Tauri v2 Windows Desktop Edition.

Thirteen concrete test cases define exact preconditions, execution commands/steps, expected HTTP response headers, and measurable pass/fail acceptance criteria.

```
+----------------------------------------------------------------------------------------------------+
|                                    TEST HARNESS & LAYER MAPPING                                    |
+------------------------------------+---------------------------------------------------------------+
| Layer 1: Spring Boot MockMvc / CORS| SpringBootTest + MockMvc (HTTP Origin & Preflight Verification)|
| Layer 2: Frontend Web Regression   | Playwright Web Testbed (Verifying web app builds & logs in)   |
| Layer 3: WebView2 Desktop Target   | Tauri Driver / Playwright for Desktop on Windows 10/11         |
| Layer 4: Protocol & Security Wire  | Curl / HTTPie / Wireshark Protocol Header Scans               |
+------------------------------------+---------------------------------------------------------------+
```

---

## 2. Comprehensive Test Matrix (13 Core Test Areas)

---

### TC-CORS-01: Existing Web Application Zero-Regression Verification
- **Target Component:** Browser Web Application (`frontend-v2/`)
- **Objective:** Verify that desktop compatibility enhancements introduce zero regressions into the existing web application.
- **Preconditions:** Spring Boot running on port 8081; web dev server on port 5173.
- **Execution Steps:**
  1. Launch web application in Google Chrome: `http://localhost:5173`.
  2. Log in as an administrator.
  3. Navigate to Student Directory, Departments, and Finance Dashboard.
  4. Inspect browser developer console and network tab.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** 100% of API requests return HTTP 200/201. Zero CORS policy errors. Zero layout or navigation regressions.
  - **FAIL:** Console displays `CORS policy: No 'Access-Control-Allow-Origin' header` or UI components fail to mount.

---

### TC-CORS-02: Tauri WebView2 Preflight OPTIONS Verification
- **Target Component:** Spring Security `CorsFilter`
- **Objective:** Verify that preflight `OPTIONS` requests from `https://tauri.localhost` (configured via `app.windows.useHttpsScheme = true`) are resolved with correct CORS headers before reaching authentication filters.
- **Preconditions:** Spring Boot server listening on port 8081.
- **Execution Steps:**
  Execute curl preflight simulation:
  ```bash
  curl -i -X OPTIONS http://localhost:8081/api/v1/students \
    -H "Origin: https://tauri.localhost" \
    -H "Access-Control-Request-Method: POST" \
    -H "Access-Control-Request-Headers: Authorization, Content-Type"
  ```
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Returns `HTTP/1.1 200 OK` or `204 No Content`. Contains headers:
    - `Access-Control-Allow-Origin: https://tauri.localhost`
    - `Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS`
    - `Access-Control-Allow-Headers: Authorization, Content-Type`
    - `Access-Control-Allow-Credentials: true`
    - `Access-Control-Max-Age: 3600`
  - **FAIL:** Returns HTTP 401 Unauthorized, HTTP 403 Forbidden, or omits `Access-Control-Allow-Origin`.

---

### TC-CORS-03: Unauthorized Origin Rejection in Production Profile
- **Target Component:** Backend `CorsConfigurationSource`
- **Objective:** Verify that an unauthorized external website cannot execute cross-origin requests against the institutional ERP.
- **Preconditions:** Spring Boot running under `production` profile with explicit origins.
- **Execution Steps:**
  ```bash
  curl -i -X OPTIONS https://erp.campus.edu:8081/api/v1/students \
    -H "Origin: https://malicious-phishing-site.com" \
    -H "Access-Control-Request-Method: GET"
  ```
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Response omits `Access-Control-Allow-Origin` or returns `HTTP 403 Forbidden`. The browser/WebView2 blocks the script from reading response data.
  - **FAIL:** Response reflects `Access-Control-Allow-Origin: https://malicious-phishing-site.com`.

---

### TC-CORS-04: Credentialed Requests without Wildcard Collision
- **Target Component:** Spring Security CORS Spec Compliance
- **Objective:** Verify that `withCredentials: true` operates cleanly with explicit origins and never triggers browser wildcard rejections.
- **Preconditions:** Desktop client making authenticated call with `Authorization: Bearer <token>`.
- **Execution Steps:**
  1. Trigger `apiClient.get('/api/v1/departments')` from inside Tauri window.
  2. Inspect response headers in WebView2 DevTools.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** `Access-Control-Allow-Origin` matches `https://tauri.localhost` explicitly. `Access-Control-Allow-Credentials` is `true`. Zero browser console warnings regarding wildcard origins.
  - **FAIL:** Header returns `*` alongside credentials, triggering browser CORS violation.

---

### TC-CORS-05: Authorization & Custom Header Exposure
- **Target Component:** Spring MVC Response Header Handler
- **Objective:** Verify that custom response headers (`Content-Disposition`, `Authorization`) are readable by frontend JavaScript.
- **Preconditions:** User requests student fee receipt download (`GET /api/v1/finance/receipts/101/pdf`).
- **Execution Steps:**
  1. Dispatch GET request via Axios in WebView2.
  2. Inspect `response.headers['content-disposition']` in JavaScript.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** `response.headers['content-disposition']` contains `"attachment; filename=..."`. JavaScript parses filename successfully.
  - **FAIL:** Header is present in network stream but stripped by WebView2 because it was omitted from `Access-Control-Expose-Headers`.

---

### TC-CORS-06: Dual-Token Silent Refresh Flow in WebView2
- **Target Component:** Axios Interceptor (`api.ts`) + `AuthController.refresh`
- **Objective:** Verify that when a 15-minute access token expires, WebView2 executes a preflight and refresh call silently without user logout.
- **Preconditions:** Desktop client running with expired mock access token and valid DPAPI refresh token.
- **Execution Steps:**
  1. User triggers an action (e.g. view student profile).
  2. Monitor network log for 401 interception and subsequent `/auth/refresh` dispatch.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** 401 error is intercepted; preflight and POST to `/auth/refresh` succeed; new access token is stored in Zustand; original request replays and resolves with 200 OK. User observes zero disruption.
  - **FAIL:** User is redirected to `/login` or preflight to `/auth/refresh` fails with CORS error.

---

### TC-CORS-07: Explicit User Logout & Token Blacklist Trapping
- **Target Component:** `AuthController.logout` + Session Service
- **Objective:** Verify that user logout invalidates tokens and purges the native DPAPI vault.
- **Preconditions:** Desktop client is authenticated.
- **Execution Steps:**
  1. User clicks "Logout".
  2. Attempt to replay the logged-out token via curl:
     `curl -i -H "Authorization: Bearer <old_token>" http://localhost:8081/api/v1/students`
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Curl request returns `HTTP 401 Unauthorized { "auth_error_code": "TOKEN_REVOKED" }`. `%APPDATA%\com.edupulse.erp\vault.dat` is completely deleted from client disk.
  - **FAIL:** Token continues to work after logout.

---

### TC-CORS-08: TLS Certificate Enforcement (Zero-Bypass Verification)
- **Target Component:** Native Rust Probe & WebView2 Network Engine
- **Objective:** Verify that invalid or self-signed TLS certificates are rejected without user-facing bypass options.
- **Preconditions:** Backend running with expired or self-signed certificate.
- **Execution Steps:**
  1. Launch desktop app configured to point to invalid certificate endpoint.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Connection probe fails immediately with `TlsCertificateInvalid`. WebView2 aborts connection. Zero "Proceed Anyway" overrides available.
  - **FAIL:** Application establishes connection despite invalid certificate.

---

### TC-CORS-09: Backend Unavailable (HTTP 503) & Dynamic Reconnection
- **Target Component:** Background Health Probe & UI Banner
- **Objective:** Verify client behavior when backend storage volume is unmounted.
- **Preconditions:** Spring Boot returning 503 from `/api/v1/health` with `{"status": "DOWN"}`.
- **Execution Steps:**
  1. Monitor desktop UI state.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Application displays persistent amber warning banner: *"Campus storage services undergoing maintenance."* Background probe polls with exponential backoff.
  - **FAIL:** Client hangs or crashes.

---

### TC-CORS-10: Dual Endpoint Versioning Parity (`/auth/*` vs `/api/v1/*`)
- **Target Component:** Spring Web Gateway & URL Routing
- **Objective:** Verify that both unversioned auth endpoints and versioned domain endpoints resolve under the dynamically injected desktop base URL.
- **Preconditions:** Desktop base URL configured to `https://erp.campus.edu:8081`.
- **Execution Steps:**
  1. Execute `/auth/login` (Auth DTO).
  2. Execute `/api/v1/students` (Domain DTO).
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Both endpoints resolve successfully against the same host and port.
  - **FAIL:** Routing errors or 404s due to path concatenation defects.

---

### TC-CORS-11: Multi-Institution Server Switching Isolation
- **Target Component:** Multi-Tenant Credential Barrier
- **Objective:** Verify that switching the server URL purges all tokens and cached tables before connecting to the new server.
- **Preconditions:** Desktop logged into College A (`https://college-a.edu:8081`).
- **Execution Steps:**
  1. Switch server configuration to College B (`https://college-b.edu:8081`).
  2. Inspect outgoing headers on the first request to College B.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** No `Authorization` header is sent to College B. `vault.dat` is wiped. User is presented with a fresh login prompt.
  - **FAIL:** College A Bearer token is transmitted to College B.

---

### TC-CORS-12: Production vs. Development Origin Enforcement
- **Target Component:** Environment Profile Resolver
- **Objective:** Verify that `http://localhost:5173` is accepted in `dev` profile and rejected in `prod` profile.
- **Preconditions:** Test against server running under `dev` profile, then `prod` profile.
- **Execution Steps:**
  1. Execute curl preflight with `Origin: http://localhost:5173`.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Under `dev`, returns `Access-Control-Allow-Origin: http://localhost:5173`. Under `prod`, returns no allow origin header.
  - **FAIL:** `prod` profile permits arbitrary local origins.

---

### TC-CORS-13: Packaged Windows Application WebView Origin Validation
- **Target Component:** Packaged Desktop Executable (`edupulse.exe`)
- **Objective:** Verify that the packaged Windows desktop application executes under `window.location.origin === "https://tauri.localhost"` (validating `app.windows.useHttpsScheme = true`), and emits an `Origin` header matching the backend CORS whitelist.
- **Preconditions:** Built release executable `edupulse.exe` deployed on Windows 10/11 workstation; Spring Boot server configured with `https://tauri.localhost` in CORS allowed origins.
- **Execution Steps:**
  1. Launch packaged `edupulse.exe`.
  2. In WebView2 DevTools console, evaluate `window.location.origin`.
  3. Execute `window.isSecureContext` in console.
  4. Dispatch an API call: `apiClient.get('/api/v1/health')`.
  5. Inspect outgoing request headers in Network tab.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:**
    - `window.location.origin` strictly equals `"https://tauri.localhost"`.
    - `window.isSecureContext` evaluates to `true`.
    - Outgoing request carries header `Origin: https://tauri.localhost`.
    - Spring Boot responds with `Access-Control-Allow-Origin: https://tauri.localhost`.
    - 0 CORS policy warnings or console errors.
  - **FAIL:** `window.location.origin` is `"http://tauri.localhost"` (indicating `useHttpsScheme` was omitted), or Spring Boot rejects the preflight/request due to origin mismatch.

---

## 3. Acceptance Gates & Verification Checklist

```
+----------------------------------------------------------------------------------------------------+
|                                    QUALITY ACCEPTANCE GATES                                        |
+-------------------+---------------------------------------------------+----------------------------+
| Gate ID           | Quality Verification Metric                       | Required Pass Threshold    |
+-------------------+---------------------------------------------------+----------------------------+
| GATE-CORS-01      | Existing web application regression suite         | 100% Passing (0 failures)  |
| GATE-CORS-02      | Preflight OPTIONS response latency                | < 50 ms round-trip         |
| GATE-CORS-03      | Wildcard credentialed origin elimination          | 0 wildcard CORS warnings   |
| GATE-CORS-04      | Token rotation single-use replay trapping         | 100% Replay Rejection      |
| GATE-CORS-05      | Zero-bypass TLS certificate enforcement           | 100% BadSSL Rejection      |
| GATE-CORS-06      | Server switch credential purge verification       | 0 residual tokens on disk  |
| GATE-CORS-07      | Packaged Windows WebView origin validation        | window.location.origin === |
|                   | (useHttpsScheme = true verification)              | "https://tauri.localhost"  |
+-------------------+---------------------------------------------------+----------------------------+
```

---

## 4. Conclusion

This validation plan ensures complete API and CORS compatibility between the Spring Boot backend, the web portal, and the new Windows desktop client. All requirements for Work Package 05 are fulfilled.
