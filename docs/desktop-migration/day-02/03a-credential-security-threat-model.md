# EduPulse ERP — Credential Security Threat Model
## Work Package 03A — Desktop Authentication Threat Model & Risk Analysis

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 02)
**Target Platform:** Windows 10 / Windows 11 Enterprise (Tauri v2 + React 19 + Spring Boot 3.3.4)
**Author:** Antigravity AI Engineering Assistant (Security Baseline)
**Date:** 2 November 2026
**Document Status:** Approved Security Architecture Specification

---

## 1. Executive Summary & Methodology

### 1.1 Context
In the current EduPulse web application, user authentication is managed via JSON Web Tokens (JWT). The client receives a short-lived **access token (15-minute expiration)** and a long-lived **refresh token (7-day expiration)** upon successful login. In the web version, both tokens are persisted in plaintext inside browser `localStorage` (`useAuthStore.ts`, key: `auth-storage`).

Transitioning EduPulse to a Windows desktop client via **Tauri v2** alters the security boundaries and attack surfaces. While eliminating traditional browser extension injection and cross-origin tab snooping, desktop execution introduces new host-level threats: process memory inspection, Windows DPAPI key derivation boundaries, local filesystem harvesting, and shared workstation contamination.

### 1.2 Threat Modeling Framework
This document applies the **STRIDE** methodology (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) combined with NIST SP 800-63B (Digital Identity Guidelines) to systematically evaluate 12 primary threat vectors against EduPulse desktop credentials.

```
+----------------------------------------------------------------------------------------------------+
|                                    TRUST BOUNDARY ARCHITECTURE                                     |
+----------------------------------------------------------------------------------------------------+
| [Zone 1: External Network / Internet]                                                             |
|       │                                                                                            |
|       ▼ (TLS 1.3 / HTTPS - Bearer JWT)                                                             |
| [Zone 2: Sandboxed Webview (Edge WebView2)]                                                        |
|       • React 19 DOM & Components                                                                  |
|       • In-Memory Zustand State (Access Token: 15m)                                                |
|       • Axios HTTP Client & Interceptors                                                           |
|       │                                                                                            |
|       ▼ (Tauri IPC Bridge - Typed Commands / Capabilities)                                         |
| [Zone 3: Trusted Native Process (Rust Core - edupulse.exe)]                                        |
|       • Command Router & Permission Scopes                                                         |
|       • DPAPI Cryptographic Bridge                                                                 |
|       │                                                                                            |
|       ▼ (Win32 CryptProtectData / CryptUnprotectData)                                              |
| [Zone 4: Windows Security Subsystem (Host OS)]                                                     |
|       • User Logon Master Key & Local LSA Secrets                                                  |
|       • %APPDATA%\com.edupulse.erp\vault.dat (Ciphertext Blob)                                    |
+----------------------------------------------------------------------------------------------------+
```

---

## 2. In-Depth Threat Analysis (12 Core Vectors)

---

### Threat 01: Credential Theft at Rest (Static File Harvesting)
- **STRIDE Category:** Information Disclosure (I)
- **Attack Scenario:** A malicious user, compromised local utility, or unauthorized IT staff with physical or file-level access to the machine navigates to the application data directory (`%APPDATA%\com.edupulse.erp\`) or browser profile directories to extract stored authentication tokens.
- **Potential Impact:** **High.** Attacker extracts long-lived refresh tokens (7-day lifespan), gains persistent unauthorized access to the institutional ERP, and impersonates faculty, student, or administrative accounts.
- **Existing Protection (Web):** **None.** Web version stores tokens in plaintext in browser `localStorage`. Any script or local tool reading LevelDB files can extract them immediately.
- **Proposed Mitigation (Desktop):**
  1. Access tokens are **never written to disk**; they exist solely in volatile JavaScript memory.
  2. Refresh tokens are encrypted before disk write using **Windows DPAPI (`CryptProtectData`)** with a cryptographic salt/entropy parameter.
  3. The resulting ciphertext file (`vault.dat`) can only be decrypted by the Windows Data Protection subsystem under the exact same logged-in Windows user account.
- **Residual Risk:** **Low.** If an attacker gains full code execution under the same Windows user session, they can invoke `CryptUnprotectData` via their own malicious process. (Mitigated by Threat 07).

---

### Threat 02: Cross-Site Scripting (XSS) in Presentation Layer
- **STRIDE Category:** Elevation of Privilege (E) / Information Disclosure (I)
- **Attack Scenario:** An attacker injects malicious JavaScript via unsanitized ERP user inputs (e.g., student admission bio, forum announcement, course syllabus markdown, or financial remarks).
- **Potential Impact:** **Critical in Web; Moderate in Desktop.** In web, XSS steals tokens from `localStorage` and exfiltrates them via `fetch()`. In Electron, XSS often leads to Remote Code Execution (RCE).
- **Existing Protection (Web):** React's automatic JSX string escaping; basic input sanitization in certain forms.
- **Proposed Mitigation (Desktop):**
  1. **Absence of Node.js:** Unlike Electron, Tauri v2 has no Node.js runtime in the renderer. XSS cannot execute system binaries or access the filesystem directly.
  2. **No Long-Lived Tokens in DOM:** Refresh tokens are held by the native Rust vault. The DOM/JavaScript context holds only the 15-minute access token in a private closure.
  3. **Strict Content Security Policy (CSP):** Enforced in `tauri.conf.json`: `default-src 'self'; script-src 'self'; connect-src 'self' https://<configured-server>; object-src 'none'`.
  4. **Tauri Capability Boundary:** The IPC command `secure_retrieve_token` is scoped strictly to internal auth refresh pipelines and protected against unauthorized script invocations.
- **Residual Risk:** **Low to Moderate.** While an active XSS script runs, it could use the active in-memory access token to perform unauthorized REST actions within its 15-minute validity window.

---

### Threat 03: Malicious Frontend Third-Party Dependencies (Supply Chain)
- **STRIDE Category:** Tampering (T) / Information Disclosure (I)
- **Attack Scenario:** A compromised npm package in `frontend-v2/node_modules` (e.g. a transitive dependency of an icon set, chart library, or utility) executes malicious code during runtime, looking to harvest keys or exfiltrate session data.
- **Potential Impact:** **High.** Silent exfiltration of session data, unauthorized API requests, or credential theft.
- **Existing Protection (Web):** `npm audit` and static linting (`oxlint`).
- **Proposed Mitigation (Desktop):**
  1. **IPC Capability Allowlist:** Tauri v2 requires explicit declarations for which windows and scripts can invoke native commands. Third-party code cannot call unmapped Rust commands.
  2. **Network Origin Lockdown:** CSP restricts outgoing HTTP connections to the verified institutional server endpoint only. Malicious dependencies cannot transmit data to external command-and-control (C2) servers.
  3. **Dependency Pinning & Lockfile Integrity:** Strict verification of `package-lock.json` with hash validation in CI.
- **Residual Risk:** **Low.** Third-party scripts could still intercept volatile DOM state or Axios responses before they are rendered, requiring continuous dependency hygiene.

---

### Threat 04: Renderer Process Compromise (Chromium Sandbox Escape)
- **STRIDE Category:** Elevation of Privilege (E)
- **Attack Scenario:** An advanced adversary exploits a zero-day vulnerability in Microsoft Edge WebView2 (Chromium rendering engine) to escape the webview sandbox and attempt native OS access.
- **Potential Impact:** **Critical.** Potential execution of native code within the user's security context.
- **Existing Protection (Web):** Browser vendor security patching.
- **Proposed Mitigation (Desktop):**
  1. **Microsoft Edge Evergreen Servicing:** Operating-system-level automatic updates to WebView2 independent of the application, ensuring immediate patching of Chromium CVEs.
  2. **Tauri Isolation Pattern:** Optional cryptographically signed iframe middleman that validates and encrypts IPC messages between WebView2 and the Rust backend.
  3. **Least Privilege Process:** The WebView2 child processes execute with low-integrity Windows security tokens.
- **Residual Risk:** **Very Low.** Dependent on Microsoft's zero-day response time for Edge WebView2.

---

### Threat 05: Inter-Process Communication (IPC) Abuse & Message Spoofing
- **STRIDE Category:** Tampering (T) / Elevation of Privilege (E)
- **Attack Scenario:** A compromised renderer script attempts to forge Tauri IPC requests to trigger privileged operations, such as dumping all stored credentials, clearing security configuration, or redirecting backend traffic.
- **Potential Impact:** **High.** Unauthorized credential extraction or denial of service.
- **Existing Protection (Web):** N/A (single process web runtime).
- **Proposed Mitigation (Desktop):**
  1. **Tauri v2 Permissions & Scopes:** Strict JSON capabilities defined in `src-tauri/capabilities/default.json`. Commands accept only strongly-typed structures validated by Serde in Rust.
  2. **No Arbitrary Key Queries:** The native IPC command `secure_retrieve_token` takes **zero parameters** and returns only the singular refresh token for the active institutional session. An attacker cannot request arbitrary registry keys or filesystem paths.
  3. **Zero Shell/Exec Primitives:** The application does not expose `tauri-plugin-shell` execution capabilities to the webview.
- **Residual Risk:** **Low.** The attack surface is restricted to the specific pre-compiled Rust commands.

---

### Threat 06: Local Filesystem Attacks & Symlink Manipulation
- **STRIDE Category:** Tampering (T) / Information Disclosure (I)
- **Attack Scenario:** An attacker or malicious local application creates symbolic links (symlinks) or junction points targeting `%APPDATA%\com.edupulse.erp\vault.dat` to overwrite critical system files or induce the application to write decrypted secrets into an unprivileged folder.
- **Potential Impact:** **Moderate.** Local privilege escalation, file corruption, or credential overwrite.
- **Existing Protection (Web):** N/A (browser manages storage isolation).
- **Proposed Mitigation (Desktop):**
  1. **Direct Path Canonicalization:** Rust storage routines use canonical path verification (`std::fs::canonicalize`) before reading or writing to `vault.dat`.
  2. **Rejection of Reparse Points:** Rust checks file metadata using `windows-rs` to ensure target paths are not symbolic links or directory junctions.
  3. **Restricted File ACLs:** The vault file is created with Windows NTFS Access Control Lists restricting read/write permissions exclusively to the creating Windows User SID and `NT AUTHORITY\SYSTEM`.
- **Residual Risk:** **Very Low.** NTFS permissions prevent cross-user symlink hijacking.

---

### Threat 07: Malware Running Under the Same Windows User Account
- **STRIDE Category:** Information Disclosure (I) / Elevation of Privilege (E)
- **Attack Scenario:** Malware (e.g., an infostealer Trojan) is executed by the user on their workstation. Because the malware runs under the exact same Windows user security context (`User SID`), it can theoretically call `CryptUnprotectData` on the ciphertext stored in `vault.dat`.
- **Potential Impact:** **Critical.** The malware successfully decrypts the 7-day refresh token and exfiltrates it to an external server.
- **Existing Protection (Web):** Plaintext `localStorage` is completely vulnerable to simple file grabbers.
- **Proposed Mitigation (Desktop):**
  1. **DPAPI Optional Entropy:** EduPulse passes an application-specific cryptographic secondary entropy string (`pOptionalEntropy`) derived from an installation-specific secret compiled or provisioned into the binary, preventing generic DPAPI decryptor tools from decrypting the file without knowing the entropy.
  2. **Backend Token Rotation & Invalidation:** Every refresh token is single-use. If malware uses a stolen refresh token, the next legitimate user refresh detects reuse and **instantly revokes the entire token family**.
  3. **Short-Lived Refresh Tokens:** In high-risk institutional deployments, refresh token duration can be reduced from 7 days to 24 hours.
- **Residual Risk:** **Moderate.** Fundamental OS principle: in Windows user-space security, any process running under a user's context can ultimately access data belonging to that user. The defense relies on token rotation and backend detection.

---

### Threat 08: Shared Institutional Computer Contamination
- **STRIDE Category:** Information Disclosure (I) / Spoofing (S)
- **Attack Scenario:** Multiple staff members, faculty, or lab students share a single physical computer running under a generic shared Windows login (e.g. `LabUser` or `Admin`). User A logs in, completes work, and closes the desktop app window without explicitly clicking "Logout". User B opens the app later.
- **Potential Impact:** **High.** User B opens EduPulse and automatically inherits User A's session, gaining access to confidential student records, grading sheets, and financial accounts.
- **Existing Protection (Web):** Closing the browser window does not clear `localStorage`; session remains active until expiration.
- **Proposed Mitigation (Desktop):**
  1. **Configurable "Shared Computer Mode":** A dedicated toggle on the login screen ("This is a shared computer").
  2. **Volatile Session Enforcement:** When Shared Computer Mode is active, the refresh token is **never written to DPAPI disk storage**; it is kept strictly in volatile memory. Closing the window completely destroys the session.
  3. **Inactivity Session Lockout:** Frontend detects 15 minutes of user inactivity (no mouse/keyboard events), clears in-memory tokens, and prompts for password re-entry.
- **Residual Risk:** **Low to Moderate.** Relies on user compliance to toggle Shared Mode or allow inactivity timers to elapse.

---

### Threat 09: Windows Account Switching & Multi-User Isolation
- **STRIDE Category:** Information Disclosure (I) / Spoofing (S)
- **Attack Scenario:** On a multi-user Windows workstation with separate OS user accounts (e.g., `Domain\ProfSmith` and `Domain\StudentJones`), Student Jones attempts to read Prof Smith's EduPulse credentials from disk.
- **Potential Impact:** **Critical.** Student impersonates faculty member.
- **Existing Protection (Web):** Browser user profiles are separated by directory permissions, but generic browsers often share session state if profiles are not used.
- **Proposed Mitigation (Desktop):**
  1. **Windows DPAPI Security Boundary:** `CryptProtectData` with `CRYPTPROTECT_UI_FORBIDDEN` encrypts data using a Master Key derived from the user’s Windows login password and SID.
  2. **Cryptographic Impossibility:** Even if Student Jones copies Prof Smith's `vault.dat` file, `CryptUnprotectData` executed under Student Jones's account will fail with error `SEC_E_DECRYPT_FAILURE` (`0x8009000D`).
- **Residual Risk:** **Zero.** Strictly enforced by the Windows NT kernel and Local Security Authority Subsystem Service (LSASS).

---

### Threat 10: Token Replay & Stolen Token Reuse
- **STRIDE Category:** Spoofing (S) / Repudiation (R)
- **Attack Scenario:** An attacker intercepts or extracts an access token or refresh token and replays it from a different machine or IP address to make unauthorized API calls.
- **Potential Impact:** **High.** Unauthorized transactions executed under legitimate credentials.
- **Existing Protection (Web):** Short-lived access token (15 mins); Spring Boot IP logging.
- **Proposed Mitigation (Desktop):**
  1. **Strict Token Rotation (Existing Backend):** Every time `/auth/refresh` is called, the provided refresh token is immediately blacklisted and a new refresh token is issued.
  2. **Token Reuse Detection:** If a previously rotated refresh token is presented again, the Spring Boot `SessionService` flags a token replay attack and **immediately invalidates all active sessions for that user**.
  3. **IP & User-Agent Context Binding:** Backend logs and correlates client IP address on token issuance and refresh.
- **Residual Risk:** **Low.** Stolen access tokens can only be replayed for up to 15 minutes before expiring.

---

### Threat 11: Refresh Token Theft via Memory Inspection
- **STRIDE Category:** Information Disclosure (I)
- **Attack Scenario:** An attacker uses process debugging tools (e.g., Process Hacker, Cheat Engine, or mini-dumps) to inspect the RAM of `edupulse.exe` or WebView2 processes to read unencrypted JWT tokens.
- **Potential Impact:** **High.** Attacker extracts active access or refresh tokens from heap memory.
- **Existing Protection (Web):** Browser heap is unencrypted in memory.
- **Proposed Mitigation (Desktop):**
  1. **Memory Scrubbing in Rust:** Refresh token plaintext in the Rust layer is held inside zeroizing memory structures (using the `zeroize` crate) and securely overwritten with zeros immediately after being passed to the DPAPI encryption API or Axios client.
  2. **Short Exposure Window:** The refresh token exists in memory for only a few milliseconds during startup or token refresh cycles.
  3. **Anti-Debugging / Integrity Checks:** Production builds compile with release optimizations and strip debug symbols.
- **Residual Risk:** **Low.** Reading process memory requires administrative or debug privileges (`SeDebugPrivilege`) on the host machine.

---

### Threat 12: Network Interception (Man-in-the-Middle)
- **STRIDE Category:** Tampering (T) / Information Disclosure (I)
- **Attack Scenario:** An adversary on the local campus network (rogue Wi-Fi access point, ARP poisoning) intercepts traffic between the desktop client and the Spring Boot backend to capture credentials in transit.
- **Potential Impact:** **Critical.** Plaintext capture of passwords, access tokens, and refresh tokens.
- **Existing Protection (Web):** HTTPS/TLS.
- **Proposed Mitigation (Desktop):**
  1. **Enforced TLS 1.3:** Desktop Axios client and Tauri probe reject insecure `http://` schemes for production institutional endpoints.
  2. **Strict Certificate Validation:** The native layer uses the Windows Root Certificate Store to validate server TLS certificates, preventing spoofed self-signed certificates.
  3. **Optional Institutional CA Certificate Pinning:** For high-security institutional campuses, the desktop client can be compiled or configured with the institution’s private root CA certificate.
- **Residual Risk:** **Low.** Standard TLS 1.3 guarantees confidentiality and integrity.

---

## 3. Threat Summary & Risk Matrix

```
+----+-----------------------------+---------------+-------------------+--------------------+---------------+
| ID | Threat Vector               | STRIDE Category| Potential Impact  | Proposed Mitigation| Residual Risk |
+----+-----------------------------+---------------+-------------------+--------------------+---------------+
| 01 | Static Credential Theft     | Info Disc (I) | High              | Windows DPAPI      | Low           |
| 02 | XSS Presentation Attack     | Elev Priv (E) | Moderate          | CSP + No Node.js   | Low           |
| 03 | Malicious npm Dependencies  | Tampering (T) | High              | Scoped Capabilities| Low           |
| 04 | WebView2 Sandbox Escape     | Elev Priv (E) | Critical          | Evergreen Updates  | Very Low      |
| 05 | IPC Command Abuse           | Tampering (T) | High              | Serde Typed Comms  | Low           |
| 06 | Filesystem / Symlink Hijack | Tampering (T) | Moderate          | Canonical ACLs     | Very Low      |
| 07 | Same-Account Malware        | Info Disc (I) | Critical          | DPAPI Entropy+Rot. | Moderate      |
| 08 | Shared Computer Leaving     | Spoofing (S)  | High              | Shared Mode + Lock | Low           |
| 09 | Windows Account Switching   | Info Disc (I) | Critical          | DPAPI User Binding | Zero          |
| 10 | Token Replay & Reuse        | Spoofing (S)  | High              | Token Family Rot.  | Low           |
| 11 | RAM Memory Scraping         | Info Disc (I) | High              | Zeroize Rust Heap  | Low           |
| 12 | Man-in-the-Middle (MitM)    | Info Disc (I) | Critical          | Enforced TLS 1.3   | Low           |
+----+-----------------------------+---------------+-------------------+--------------------+---------------+
```

---

## 4. Conclusion & Actionable Security Directives

1. **Deprecate `localStorage` Persistence:** Completely remove token storage from browser `localStorage` in the desktop distribution.
2. **Implement Dual-Token Storage Boundary:** Access token strictly in volatile JavaScript memory (Zustand); refresh token strictly encrypted via Windows DPAPI.
3. **Enforce Single-Use Token Rotation:** Leverage the existing Spring Boot `SessionService.revokeToken()` to invalidate refresh tokens upon every refresh transaction.
4. **Enforce Shared Computer Controls:** Implement explicit session timeout and shared workstation modes to protect multi-user institutional environments.

Proceed to **Work Package 03B: Token Lifecycle Design**.
