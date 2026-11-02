# EduPulse ERP — Day 02, Work Package 08
# Deliverable 08b: Tauri IPC and Native Privilege Security Controls

## 1. Executive Summary and Threat Context

The Inter-Process Communication (IPC) bridge connects the untrusted React 19 presentation layer (running within the Microsoft Edge WebView2 rendering engine) to the privileged Tauri Rust native shell.

In hybrid desktop applications, the IPC bridge is the primary boundary between web vulnerabilities (such as Cross-Site Scripting / XSS or malicious npm dependencies) and native operating system privileges. If the IPC surface is over-permissioned, a minor DOM-based XSS vulnerability can escalate into arbitrary command execution, local file system compromise, or host takeover.

---

## 2. Complete Inventory of Exposed Tauri Commands

EduPulse ERP enforces an absolute minimum attack surface. The table below represents the complete, closed inventory of all native commands exposed to the WebView:

| Command Category | Command Identifier | Permitted Window | Input Validation & Constraints | Privileged Operation Executed |
| :--- | :--- | :--- | :--- | :--- |
| **Authentication** | `vault_store_refresh_token` | `"main"` | String (JWT format validated via regex) | Calls DPAPI `CryptProtectData` to persist token. |
| **Authentication** | `vault_get_refresh_token` | `"main"` | None | Calls DPAPI `CryptUnprotectData` and returns token. |
| **Authentication** | `vault_clear_credentials` | `"main"` | None | Overwrites and unlinks DPAPI credential file. |
| **Authentication** | `vault_verify_offline_pin` | `"main"` | String (6-8 numeric digits) | Compares Argon2id hash inside DPAPI session snapshot. |
| **Offline Storage**| `query_offline_cache` | `"main"` | Struct `CacheQueryArgs` (sanitized enum, pagination) | Executes parameterized `SELECT` on SQLCipher. |
| **Offline Storage**| `get_sync_status` | `"main"` | None | Queries `sync_metadata` table for sync timestamps. |
| **Offline Storage**| `trigger_manual_sync` | `"main"` | None (Rate-limited to 1 per 60 seconds) | Signals background sync thread to request deltas. |
| **Configuration** | `get_endpoint_config` | `"main"` | None | Reads Registry policy or `%APPDATA%\endpoint.json`. |
| **Configuration** | `set_endpoint_config` | `"main"` | Validated HTTPS URL (String) | Rejects if policy locked; writes to `endpoint.json`. |
| **Configuration** | `verify_endpoint_health` | `"main"` | Validated HTTPS URL (String) | Executes 3.0s timeout probe to `/api/v1/health`. |
| **Diagnostics** | `get_system_diagnostics`| `"main"` | None | Compiles sanitized OS, WebView2, and DB metadata. |
| **Diagnostics** | `export_diagnostic_package`| `"main"` | None | Compresses sanitized logs; uses native Save Dialog. |
| **Diagnostics** | `log_frontend_event` | `"main"` | Struct `FrontendLogPayload` (safe fields only) | Forwards UI error boundary events to Rust logger. |
| **Window** | `window_minimize` | `"main"` | None | Native Win32 `ShowWindow(SW_MINIMIZE)`. |
| **Window** | `window_maximize` | `"main"` | None | Native Win32 window toggle. |
| **Window** | `window_close` | `"main"` | None | Triggers graceful application shutdown sequence. |

### Prohibited Native Capabilities:
- **NO `fs:*` Capabilities:** Generic file read, write, copy, or delete commands are completely excluded.
- **NO `shell:*` Capabilities:** Arbitrary process spawning, command execution (`cmd.exe`, `powershell.exe`), or shell execution is strictly blocked.
- **NO Direct SQL Execution:** The frontend cannot pass raw SQL strings. All queries are pre-compiled prepared statements.

---

## 3. Declarative Capabilities and Window Scoping in Tauri v2

In Tauri v2, permissions are scoped declaratively via capabilities files located in `src-tauri/capabilities/`.

```json
{
  "$schema": "https://raw.githubusercontent.com/tauri-apps/tauri/dev/crates/tauri-utils/schema.json",
  "identifier": "default-capability",
  "description": "Default least-privilege capability set for EduPulse ERP",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "core:window:allow-minimize",
    "core:window:allow-maximize",
    "core:window:allow-close",
    "core:window:allow-set-title",
    "dialog:allow-save"
  ]
}
```

### 3.1 Security Enforcement Rules

1. **Window Scoping:** All native commands and capabilities are restricted strictly to the primary window titled `"main"`. If any dialog or modal spawns an auxiliary window context, that secondary window possesses zero native IPC capabilities.
2. **Remote Navigation Lockdown:** If the WebView navigates away from the local packaged scheme (`https://tauri.localhost`) to an external web URL, Tauri v2's security architecture immediately revokes all IPC access. External web content cannot invoke native Rust functions.
3. **Content Security Policy (CSP):**
   ```text
   default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self' https:; font-src 'self' data:; object-src 'none'; frame-src 'none';
   ```
   The CSP forbids `unsafe-eval`, external scripts, and insecure `http:` network egress.

---

## 4. Blast-Radius Analysis: Containing Frontend XSS

A fundamental design requirement is that **a catastrophic DOM XSS vulnerability in React must not compromise host integrity or database encryption keys**.

```mermaid
flowchart TD
    subgraph Attacker Vector
        XSS[Attacker Injects Malicious JavaScript into React DOM]
    end

    subgraph What Attacker CANNOT Do
        C1[CANNOT Execute Shell or Spawn OS Processes: No shell:execute capability]
        C2[CANNOT Read Host Filesystem: No fs:read capability]
        C3[CANNOT Steal SQLCipher AES Key: Key is isolated in Rust memory]
        C4[CANNOT Write Directly to SQLite: Prepared statements only]
        C5[CANNOT Alter Backend URL if GPO Locked: Protected in HKLM Registry]
    end

    subgraph What Attacker COULD Attempt
        A1[Attempt to call vault_get_refresh_token via IPC]
        A2[Attempt to query cached student records via IPC]
    end

    subgraph Containment & Defenses
        D1[Strict Token Rotation: Backend invalidates token family on reuse]
        D2[Role Snapshot Guard: Attacker can only query records permitted to active user]
    end

    XSS -.->|Blocked| C1
    XSS -.->|Blocked| C2
    XSS -.->|Blocked| C3
    XSS -.->|Blocked| C4
    XSS -.->|Blocked| C5

    XSS --> A1
    XSS --> A2
    A1 --> D1
    A2 --> D2

    style Attacker Vector fill:#ffebee,stroke:#c62828,stroke-width:2px
    style What Attacker CANNOT Do fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style What Attacker COULD Attempt fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style Containment & Defenses fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
```

### Exploit Resistance Details:
1. **Database Key Inaccessibility:** The 256-bit AES-256 SQLCipher encryption key is generated, decrypted, and held exclusively within the Rust process heap. There is no IPC command to request, return, or export the key.
2. **Strict Token Rotation Defense:** Even if an XSS script calls `vault_get_refresh_token` and exfiltrates the refresh token over the network, Spring Boot's single-use token rotation ensures that when either the legitimate user or the attacker attempts to refresh, the backend detects reuse and permanently terminates the entire token family.
3. **No Dynamic SQL Execution:** An attacker cannot execute SQL injection through the IPC bridge because `query_offline_cache` maps input parameters to pre-compiled positional SQLite queries (`SELECT ... WHERE student_id = ?`).

---

## 5. Input Validation and IPC Boundary Contracts

All IPC arguments crossing the WebView-to-Rust boundary undergo strict two-tier schema validation:

```
+-----------------------------------------------------------------------------------+
|                        TWO-TIER IPC VALIDATION ARCHITECTURE                       |
|                                                                                   |
|   Tier 1: Client-Side TypeScript & Zod Validation                                 |
|   - Ensures parameters conform to expected formats before dispatching IPC call.   |
|   - Immediate developer feedback and friendly UI error states.                    |
|                                                                                   |
|   Tier 2: Native Rust Serde Strongly-Typed Deserialization                        |
|   - Rust command handlers reject malformed JSON, unexpected types, or extra keys. |
|   - Bounds and Range Checking: Page numbers capped at 10,000; Page sizes max 500. |
|   - URL Validation: Must parse as valid HTTPS URL; cleartext http:// rejected.    |
|   - String Sanitization: Strips null bytes (\0) to prevent native buffer overflows.|
+-----------------------------------------------------------------------------------+
```

---

## 6. Six Mandatory Least-Privilege Rules

To maintain long-term architectural integrity, the development team must adhere to six immutable IPC rules:

1. **Rule 1 (Zero Key Exposure):** Under no circumstances shall cryptographic keys (AES keys, DPAPI master keys, Minisign private keys) ever be passed into or out of the IPC channel.
2. **Rule 2 (No Unrestricted Filesystem Access):** All file reads and writes must be handled within Rust native services targeting explicitly sandboxed application paths (`%APPDATA%`, `%LOCALAPPDATA%`).
3. **Rule 3 (No Arbitrary Shell Execution):** The application shall not enable or bundle `shell:execute` or `shell:open` capabilities for arbitrary command lines.
4. **Rule 4 (Minimal Command Surface):** Every newly proposed Tauri command must undergo an architecture security review. If an operation can be performed within standard web browser semantics, it must not be added to Rust.
5. **Rule 5 (Input Bounds Enforcement):** All Rust IPC handlers must validate arguments independently of frontend assertions. Never trust client-supplied IDs, limits, or URLs.
6. **Rule 6 (No Authorization Duplication):** Rust IPC handlers enforce local snapshot boundaries for offline queries, but never attempt to simulate or overwrite central Spring Boot business logic.
