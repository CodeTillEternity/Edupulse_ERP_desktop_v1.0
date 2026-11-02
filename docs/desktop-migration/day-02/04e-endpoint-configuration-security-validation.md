# EduPulse ERP — Endpoint Configuration Security Validation Plan
## Work Package 04E — Verification Test Matrix & Acceptance Gates

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 02)
**Target Platform:** Windows 10 / Windows 11 Enterprise (Tauri v2 + Spring Boot 3.3.4)
**Author:** Antigravity AI Engineering Assistant (Security Baseline)
**Date:** 2 November 2026
**Document Status:** Approved Quality Assurance & Verification Specification

---

## 1. Executive Summary & Verification Methodology

This specification defines the formal **Endpoint Configuration Security Validation Plan** for EduPulse Windows Desktop Edition. It provides eleven concrete, automated and manual test cases covering URL validation, TLS certificate enforcement, Group Policy overrides, server switching isolation, and fault recovery.

```
+----------------------------------------------------------------------------------------------------+
|                                    TEST LAYER & TOOLING SPECIFICATION                              |
+------------------------------------+---------------------------------------------------------------+
| Layer 1: Rust Configuration Unit   | Cargo Test + mockito / wiremock HTTP mock harnesses           |
| Layer 2: Frontend Setup Wizard     | Vitest + React Testing Library + Mock Tauri Core              |
| Layer 3: System & Registry Target  | PowerShell Pester + Windows Registry test fixtures (HKLM/HKCU)|
| Layer 4: Network & TLS Integration | Real Windows 10/11 VMs + Self-signed & BadSSL testbeds        |
+------------------------------------+---------------------------------------------------------------+
```

---

## 2. Comprehensive Test Matrix (11 Core Test Areas)

---

### TC-CONF-01: Malformed and Invalid URL Rejection
- **Target Component:** Rust URI Parser & Frontend Normalizer
- **Objective:** Verify that invalid URL strings cannot be saved into configuration.
- **Preconditions:** Initial Setup Wizard open.
- **Execution Steps:**
  1. Test the following inputs sequentially:
     - `ftp://erp.campus.edu` (Illegal scheme)
     - `javascript:alert(1)` (Script injection)
     - `https://erp.campus.edu:99999` (Out-of-bounds port)
     - `https://` (Missing host)
     - `https://erp campus edu:8081` (Embedded space)
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** All invalid inputs are rejected at the UI layer and Rust parser with `ConfigError::InvalidUrlSyntax`. Setup Wizard displays clear validation error. Zero files written to disk.
  - **FAIL:** Any malformed string is persisted to `config.json`.

---

### TC-CONF-02: Plaintext HTTP Rejection in Production Builds
- **Target Component:** Protocol Enforcement Engine (`src-tauri/`)
- **Objective:** Verify that unencrypted `http://` schemes are rejected in production release builds.
- **Preconditions:** Application compiled with `--release`.
- **Execution Steps:**
  1. Enter `http://erp.campus.edu:8081` into the Setup Wizard.
  2. Attempt to save configuration.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Pre-flight probe is aborted immediately; returns `ConfigError::InsecureHttpForbidden`. UI alerts: *"Unencrypted HTTP connections are forbidden in production. Please use HTTPS."*
  - **FAIL:** Application establishes unencrypted HTTP connection.

---

### TC-CONF-03: Invalid, Expired, and Self-Signed TLS Certificate Rejection
- **Target Component:** Native TLS Engine (Windows Schannel)
- **Objective:** Verify that invalid, expired, or untrusted TLS certificates are rejected without user-facing bypass options.
- **Preconditions:** Mock HTTPS test servers configured with:
  - Test Server A: Self-signed certificate (untrusted root).
  - Test Server B: Expired certificate.
  - Test Server C: Hostname mismatch certificate (CN: `other.com`).
- **Execution Steps:**
  1. Direct Setup Wizard to Test Server A, then B, then C.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** In all three cases, the native connection probe fails with `NetworkError::TlsCertificateInvalid`. No "Proceed Anyway" or certificate ignore buttons exist. Connection is rejected.
  - **FAIL:** Application ignores TLS errors or establishes connection.

---

### TC-CONF-04: DNS Resolution Failure & Off-Campus Split-Horizon Guidance
- **Target Component:** Network Failure Handler
- **Objective:** Verify that DNS failures provide clear, actionable guidance to the user.
- **Preconditions:** Workstation disconnected from campus LAN; server set to `https://erp.internal.campus.local:8081`.
- **Execution Steps:**
  1. Launch `edupulse.exe`.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** DNS failure is caught within 2.5s. Application renders non-blocking modal: *"Unable to resolve campus server hostname. If you are working remotely, please connect to your institution's VPN."*
  - **FAIL:** Application crashes, hangs indefinitely, or surfaces cryptic `getaddrinfo` socket errors.

---

### TC-CONF-05: Atomic Server URL Switching & Credential Eviction
- **Target Component:** Multi-Tenant Isolation Bridge
- **Objective:** Verify that switching the server URL completely purges stored tokens, offline cache, and in-memory state.
- **Preconditions:** User is logged in to `https://college-a.edu`; `vault.dat` exists; offline cache holds 100 student records.
- **Execution Steps:**
  1. Navigate to Settings -> Server Configuration.
  2. Change URL to `https://college-b.edu` and confirm switch.
  3. Inspect `%APPDATA%\com.edupulse.erp\vault.dat`.
  4. Inspect offline SQLite cache database.
  5. Inspect Axios default headers and Zustand store.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** `vault.dat` is wiped with random bytes and unlinked. SQLite table `offline_cache` has 0 rows. Zustand state has `token: null`. Axios header `Authorization` is unset. Application routes cleanly to `/login`. College A tokens are never transmitted to College B.
  - **FAIL:** Residual tokens or cached records remain on disk after server switch.

---

### TC-CONF-06: Configuration File Corruption & Automatic Recovery
- **Target Component:** Rust Configuration Loader
- **Objective:** Verify application resilience when `config.json` contains corrupted syntax or truncated bytes.
- **Preconditions:** Application closed.
- **Execution Steps:**
  1. Open `%APPDATA%\com.edupulse.erp\config.json`.
  2. Overwrite file with invalid JSON: `{ "server": { "base_url": "broken... `.
  3. Launch `edupulse.exe`.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Corrupted file is detected, backed up to `config.json.corrupt.{timestamp}`, and the Initial Setup Wizard is presented cleanly. Application does NOT panic or exit abruptly.
  - **FAIL:** Application crashes on startup.

---

### TC-CONF-07: Unauthorized Configuration File Modification (ACL Tampering)
- **Target Component:** Windows NTFS Security Checks
- **Objective:** Verify that `config.json` cannot be modified by unauthorized standard users across Windows accounts.
- **Preconditions:** Multi-user Windows workstation (User Alice and User Bob).
- **Execution Steps:**
  1. Bob attempts to overwrite Alice's `%APPDATA%\com.edupulse.erp\config.json`.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Windows NTFS ACL denies write access (`ACCESS_DENIED`).
  - **FAIL:** Bob modifies Alice's configuration.

---

### TC-CONF-08: Windows Group Policy (HKLM) Precedence Enforcement
- **Target Component:** Precedence Hierarchy Engine
- **Objective:** Verify that machine-level GPO in `HKLM` unconditionally overrides user settings and CLI flags.
- **Preconditions:** Set registry key `HKLM\Software\Policies\EduPulse\ServerUrl = "https://gpo.campus.edu:8081"` and `LockConfiguration = 1`. Set `%APPDATA%\config.json` to `https://user.campus.edu:8081`.
- **Execution Steps:**
  1. Launch `edupulse.exe --server-url="https://cli.campus.edu:8081"`.
  2. Inspect resolved base URL in React application.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Application connects to `https://gpo.campus.edu:8081`. Settings page displays lock icon and disables URL input. CLI flag is ignored and logged to Event Log (Event 1004).
  - **FAIL:** CLI flag or local user file overrides the machine GPO.

---

### TC-CONF-09: Backend Health Degradation (HTTP 503 DOWN Handling)
- **Target Component:** Health Probe Monitor
- **Objective:** Verify application response when the Spring Boot server is reachable but storage is offline (`status: DOWN`).
- **Preconditions:** Mock `/api/v1/health` returning HTTP 503 `{"status": "DOWN", "error": "Storage unmounted"}`.
- **Execution Steps:**
  1. Run pre-flight setup probe or background monitor.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Application flags server degradation. Displays warning banner: *"Campus server is reachable, but backend storage services are undergoing maintenance."* Application prevents logins until health status returns to `UP`.
  - **FAIL:** Application attempts regular API calls and fails with unhandled 500 errors.

---

### TC-CONF-10: Automatic Background Reconnection & Cache Revalidation
- **Target Component:** Background Network Monitor
- **Objective:** Verify automatic session recovery after a temporary network blackout.
- **Preconditions:** User logged in; Wi-Fi network disabled for 60 seconds, then re-enabled.
- **Execution Steps:**
  1. Disconnect network: observe persistent offline banner.
  2. Reconnect network: observe background recovery.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** Background probe detects network recovery within 5 seconds of link up. Offline banner is automatically dismissed. Active TanStack Query views are re-fetched without requiring a manual page refresh.
  - **FAIL:** Application remains locked in offline mode until manually restarted.

---

### TC-CONF-11: Multi-Institution Credential Isolation Verification
- **Target Component:** End-to-End Security Boundary
- **Objective:** Verify that credentials from Institution 1 cannot be replayed against Institution 2.
- **Preconditions:** Testbed with two independent Spring Boot instances (Server 1 and Server 2).
- **Execution Steps:**
  1. Authenticate to Server 1.
  2. Switch URL to Server 2.
  3. Attempt to invoke any protected API on Server 2 using previous session state.
- **Pass / Fail Acceptance Criteria:**
  - **PASS:** All previous tokens are wiped before connecting to Server 2. User is presented with a fresh login prompt. Zero Authorization headers sent to Server 2.
  - **FAIL:** Any token issued by Server 1 is transmitted in an HTTP request to Server 2.

---

## 3. Acceptance Gates & Verification Checklist

```
+----------------------------------------------------------------------------------------------------+
|                                    QUALITY ACCEPTANCE GATES                                        |
+-------------------+---------------------------------------------------+----------------------------+
| Gate ID           | Quality Verification Metric                       | Required Pass Threshold    |
+-------------------+---------------------------------------------------+----------------------------+
| GATE-CONF-01      | RFC 3986 URL parsing & canonicalization suite     | 100% Passing (0 failures)  |
| GATE-CONF-02      | Insecure plaintext HTTP rejection in release      | 100% Rejection             |
| GATE-CONF-03      | Zero-bypass TLS certificate validation test       | 100% Rejection of BadSSL   |
| GATE-CONF-04      | GPO HKLM override precedence compliance           | 100% Precedence Compliance |
| GATE-CONF-05      | Forensic token eviction on server switch          | 0 residual tokens on disk  |
| GATE-CONF-06      | Corrupt configuration automatic recovery test     | Clean reset (Zero crashes) |
| GATE-CONF-07      | Pre-flight connectivity probe latency             | < 3,000 ms to result       |
+-------------------+---------------------------------------------------+----------------------------+
```

---

## 4. Conclusion

This validation plan guarantees that the configurable backend endpoint architecture for EduPulse Windows Desktop Edition maintains enterprise-grade security, fault tolerance, and institutional tenant isolation. Proceed immediately to final review and commit execution.
