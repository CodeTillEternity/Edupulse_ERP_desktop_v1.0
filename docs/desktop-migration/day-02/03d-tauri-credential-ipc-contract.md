# EduPulse ERP — Tauri Credential IPC Security Contract
## Work Package 03D — Native IPC Interface & Capability Specification

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 02)
**Target Platform:** Windows 10 / Windows 11 Enterprise (Tauri v2 + Rust Core)
**Author:** Antigravity AI Engineering Assistant (Security Baseline)
**Date:** 2 November 2026
**Document Status:** Approved Engineering Architecture Specification

---

## 1. Executive Summary & Security Directives

In Tauri v2, Inter-Process Communication (IPC) is the critical trust boundary connecting the untrusted presentation layer (Microsoft Edge WebView2 executing React 19) with the privileged host operating system layer (Rust native process).

This specification establishes the strongly-typed, audited IPC security contract governing all authentication credential operations.

```
+----------------------------------------------------------------------------------------------------+
|                                    CORE IPC SECURITY DIRECTIVES                                    |
+------------------------------------+---------------------------------------------------------------+
| Window Invocation Scoping          | Restricted strictly to "main" application window.             |
| Argument Validation                | Pre-execution schema validation via Serde in Rust.           |
| Cryptographic Memory Scrubbing     | Plaintext token buffers wrapped in zeroize::Zeroizing.        |
| Secret Path Immutability           | Storage path is hardcoded in Rust; zero client path inputs.   |
| Zero Logging of Credentials        | Strict prohibition on logging raw JWT tokens or ciphertext.   |
| Capability-Based Allowlist         | Declared in src-tauri/capabilities/credential.json.           |
| Shell / Filesystem Isolation       | Arbitrary filesystem and shell execution APIs are forbidden.  |
+------------------------------------+---------------------------------------------------------------+
```

---

## 2. Tauri v2 Capability & Permission Configuration

Under Tauri v2's Capability System, every IPC command must be explicitly granted through a capability JSON manifest. The application defines `src-tauri/capabilities/credential.json`:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "edupulse-credential-capability",
  "description": "Grants main window access to encrypted DPAPI credential storage",
  "windows": [
    "main"
  ],
  "permissions": [
    "core:default",
    "edupulse:allow-secure-store-token",
    "edupulse:allow-secure-retrieve-token",
    "edupulse:allow-secure-clear-token",
    "edupulse:allow-check-credential-status"
  ]
}
```

*Security Enforcement:* Any script attempting to call an unpermitted command receives an immediate IPC invocation rejection (`IPC_PERMISSION_DENIED`), preventing unvetted third-party npm scripts from accessing the native vault.

---

## 3. Strongly-Typed Native Command Contracts

The native Rust layer implements four dedicated, strongly-typed commands.

---

### 3.1 Command 1: `secure_store_token`
Encrypts and persists a newly issued or rotated refresh token using Windows DPAPI.

#### Rust Command Signature & Types:
```rust
#[derive(serde::Deserialize)]
pub struct StoreTokenRequest {
    pub refresh_token: String,
    pub institution_id: Option<String>,
}

#[derive(serde::Serialize)]
pub struct StoreTokenResponse {
    pub success: bool,
    pub token_id: String,
    pub stored_at: String, // ISO 8601 UTC timestamp
}

#[tauri::command]
pub async fn secure_store_token(
    window: tauri::Window,
    payload: StoreTokenRequest,
) -> Result<StoreTokenResponse, CredentialError>;
```

#### Input Validation Rules:
1. **Window Identity:** Must originate from `window.label() == "main"`.
2. **Payload Non-Empty:** `refresh_token.trim().is_empty()` returns `CredentialError::EmptyToken`.
3. **JWT Structure Verification:** The string must contain exactly two period characters (`.`), conforming to standard three-segment JWT structure (`header.payload.signature`).
4. **Length Restriction:** `refresh_token.len()` must be between 100 bytes and 4,096 bytes. Payloads exceeding 4KB are rejected to prevent memory exhaustion.
5. **Memory Management:** The incoming token string is immediately moved into a `zeroize::Zeroizing<String>` buffer, ensuring that once encrypted, the plaintext is zeroized upon leaving scope.

---

### 3.2 Command 2: `secure_retrieve_token`
Decrypts and returns the persisted refresh token for silent session restoration or token refresh.

#### Rust Command Signature & Types:
```rust
#[derive(serde::Serialize)]
pub struct RetrieveTokenResponse {
    pub refresh_token: String,
    pub has_credential: bool,
}

#[tauri::command]
pub async fn secure_retrieve_token(
    window: tauri::Window,
) -> Result<RetrieveTokenResponse, CredentialError>;
```

#### Input Validation & Security Rules:
1. **Zero Input Parameters:** The command accepts **no arguments**. The client cannot specify custom file paths, keys, or user IDs. The Rust core retrieves strictly the single active `vault.dat` belonging to the current Windows user.
2. **Window Verification:** Restricted to `window.label() == "main"`.
3. **Vault Existence Check:** If `vault.dat` does not exist, returns `RetrieveTokenResponse { refresh_token: String::new(), has_credential: false }` without throwing an exception, allowing clean initial login routing.

---

### 3.3 Command 3: `secure_clear_token`
Forensically overwrites and deletes the stored refresh token during user logout.

#### Rust Command Signature & Types:
```rust
#[tauri::command]
pub async fn secure_clear_token(
    window: tauri::Window,
) -> Result<(), CredentialError>;
```

#### Operational Execution Steps:
1. Rust opens `%APPDATA%\com.edupulse.erp\vault.dat`.
2. Overwrites the file with cryptographically secure random bytes (`rand::fill`).
3. Calls `File::sync_all()` to flush buffers to physical disk.
4. Overwrites the file with zeros (`0x00`).
5. Calls `File::sync_all()` a second time.
6. Calls `std::fs::remove_file("vault.dat")`.
7. Returns `Ok(())`.

---

### 3.4 Command 4: `check_credential_status`
Queries whether a valid credential exists without decrypting the token into memory.

#### Rust Command Signature & Types:
```rust
#[derive(serde::Serialize)]
pub struct CredentialStatusResponse {
    pub has_stored_credential: bool,
    pub vault_size_bytes: u64,
    pub last_modified: Option<String>,
}

#[tauri::command]
pub async fn check_credential_status(
    window: tauri::Window,
) -> Result<CredentialStatusResponse, CredentialError>;
```

#### Security Benefit:
Enables the React frontend during cold launch to determine whether to show a "Resuming Session..." loading skeleton or navigate directly to `/login`, without unnecessarily decrypting the token if the user is already determined to be logged out.

---

## 4. Error Handling & Sanitized Error Enums

Native errors are mapped to a strictly sanitized enum to prevent internal system path disclosures:

```rust
#[derive(Debug, thiserror::Error, serde::Serialize)]
#[serde(tag = "code", content = "message")]
pub enum CredentialError {
    #[error("Caller unauthorized to invoke credential operations")]
    UnauthorizedCaller,

    #[error("Provided authentication token is malformed")]
    InvalidTokenFormat,

    #[error("Token string cannot be empty")]
    EmptyToken,

    #[error("No stored credential vault found on workstation")]
    VaultNotFound,

    #[error("Operating system failed to decrypt stored credentials")]
    DecryptionFailed,

    #[error("Local filesystem error encountered during vault operation")]
    IoFailure,

    #[error("Vault ciphertext is corrupted or was modified externally")]
    CorruptVault,
}
```

*Security Rule:* Raw Windows `HRESULT` error codes (e.g. `0x8009000D`) and internal Windows filesystem paths (e.g. `C:\Users\...\AppData\Roaming\...`) are **never serialized to the webview**.

---

## 5. Strict Logging Restrictions & Audit Policy

To prevent inadvertent credential leakage through application logging systems or crash reports:

```
+----------------------------------------------------------------------------------------------------+
|                                    STRICT AUDIT & LOGGING POLICY                                   |
+------------------------------------+---------------------------------------------------------------+
| Prohibited Item                    | Compliance Rule                                               |
+------------------------------------+---------------------------------------------------------------+
| Full JWT Strings                   | NEVER logged under any log level (Trace, Debug, Info, Warn).  |
| Partial Token Slices (Prefix/Suffix)| NEVER logged (tokens contain base64 signatures).             |
| DPAPI Raw Ciphertext Hex/Base64    | NEVER logged to files, stdout, or Windows Event Log.          |
| User Passwords                     | NEVER logged in frontend or native layers.                    |
+------------------------------------+---------------------------------------------------------------+
| Permitted Item                     | Approved Logging Pattern                                      |
+------------------------------------+---------------------------------------------------------------+
| Token JTI (UUID)                   | Allowed: log::info!("Token registered. JTI: {}", tokenId);    |
| Operation Latency                  | Allowed: log::debug!("DPAPI protect completed in {}ms", ms);   |
| Payload Byte Size                  | Allowed: log::debug!("Encrypted payload size: {} bytes", sz); |
+------------------------------------+---------------------------------------------------------------+
```

---

## 6. TypeScript Client Bridge Specification

The frontend encapsulates IPC calls inside a strongly-typed service (`frontend-v2/src/services/credentialBridge.ts`):

```typescript
import { invoke } from '@tauri-apps/api/core';

export interface StoreTokenResponse {
  success: boolean;
  tokenId: string;
  storedAt: string;
}

export interface RetrieveTokenResponse {
  refreshToken: string;
  hasCredential: boolean;
}

export interface CredentialStatusResponse {
  hasStoredCredential: boolean;
  vaultSizeBytes: number;
  lastModified?: string;
}

export const credentialBridge = {
  async storeRefreshToken(refreshToken: string): Promise<StoreTokenResponse> {
    return invoke<StoreTokenResponse>('secure_store_token', {
      payload: { refresh_token: refreshToken }
    });
  },

  async retrieveRefreshToken(): Promise<string | null> {
    const res = await invoke<RetrieveTokenResponse>('secure_retrieve_token');
    return res.hasCredential ? res.refreshToken : null;
  },

  async clearRefreshToken(): Promise<void> {
    return invoke<void>('secure_clear_token');
  },

  async checkStatus(): Promise<CredentialStatusResponse> {
    return invoke<CredentialStatusResponse>('check_credential_status');
  }
};
```

Proceed to **Work Package 03E: Session Revocation and Logout Security**.
