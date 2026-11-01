# Work Package 04B — Authentication & Security Architecture Audit

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 01)  
**Target Module:** `backend/src/main/java/com/erp/auth/`, `security/` & `frontend-v2/src/config/api.ts`  
**Date of Audit:** 1 November 2026  
**Auditor:** Antigravity AI Engineering Assistant (Pair Programming Baseline)

---

## 1. Executive Summary

Security in EduPulse is built on an end-to-end stateless JSON Web Token (JWT) architecture backed by Spring Security on the server and Zustand with Axios interceptors on the client. Role-Based Access Control (RBAC) is enforced at both the API level and the frontend router level across seven discrete institutional roles.

The authentication engine features dual-token issuance (short-lived access tokens and longer-lived refresh tokens), automated silent refresh queues, client IP logging, and session revocation. In migrating to a Windows desktop runtime, specific security adaptations are required to transition from vulnerable browser `localStorage` to OS-level encrypted credential stores (e.g., Windows DPAPI / Credential Manager).

---

## 2. End-to-End Authentication Lifecycle & Flow

```
[User / Desktop Client]             [Spring Security Filter]              [Auth Service / DB]
        │                                      │                                  │
   1.   │── POST /auth/login (username, pass) ─┼─────────────────────────────────>│
        │                                      │                                  │
   2.   │                                      │                           Verify password
        │                                      │                           (BCryptPasswordEncoder)
        │                                      │                                  │
   3.   │                                      │                           Generate JWT Pair:
        │                                      │                           - Access Token (15m)
        │                                      │                           - Refresh Token (7d)
        │                                      │                           Include roles & perms
        │                                      │                                  │
   4.   │<── 200 OK (Tokens, User, Roles) ─────┼──────────────────────────────────│
        │
   5.   │── Store in Zustand auth-storage
        │   (Currently localStorage in web)
        │
   6.   │── GET /api/v1/students ──────────────>│                                  │
        │   [Authorization: Bearer <token>]    │                                  │
   7.   │                                      │── JwtAuthenticationFilter        │
        │                                      │   Verify signature & expiry      │
        │                                      │   Populate SecurityContext       │
        │                                      │                                  │
   8.   │                                      │── Role / Permission Check ──────>│ Dispatch Controller
        │                                      │                                  │
   9.   │<── 200 OK (Protected Data) ──────────┼──────────────────────────────────│
        │
  10.   │── [Token Expires -> 401 Error] ──────>│                                  │
        │                                      │                                  │
  11.   │── POST /auth/refresh (refreshToken) ─┼─────────────────────────────────>│
        │                                      │                           Verify refresh token
        │                                      │                           Rotate refresh token
  12.   │<── 200 OK (New Access + Refresh) ────┼──────────────────────────────────│
        │
  13.   │── Drain failedQueue & retry requests │                                  │
        │
  14.   │── POST /auth/logout ─────────────────┼─────────────────────────────────>│
        │   Clear local tokens & session state │                           Invalidate session
```

---

## 3. Token Generation & JWT Specifications

### 3.1 Token Properties & Signing
- **Library:** JJWT (`io.jsonwebtoken:jjwt-api:0.11.5`).
- **Signature Algorithm:** HMAC-SHA256 (`Keys.hmacShaKeyFor(secret.getBytes())`).
- **Secret Management:** Externalized via `${JWT_SECRET}` in production, eliminating hardcoded keys.
- **Token Claims Payload:**
  - `sub`: Principal username.
  - `role`: Primary role name (e.g. `ROLE_ADMIN`, `ROLE_FACULTY`).
  - `permissions`: List of granular permission strings.
  - `iat`: Epoch issuance timestamp.
  - `exp`: Epoch expiration timestamp.

### 3.2 Token Lifecycles & Expiration
- **Access Token:** Short lifespan (~15 minutes to 1 hour depending on profile), designed to minimize exposure if intercepted.
- **Refresh Token:** Extended lifespan (~7 days), stored in the database or validated with signature and user state. Token rotation is enforced during refresh.

---

## 4. Role-Based Access Control (RBAC) Matrix

The system enforces 7 distinct institutional roles and 17 granular permission scopes defined in `com.erp.auth.model.Role`:

| Role Name | Authority String | Granular Permissions Granted | Typical Operational Scope |
| :--- | :--- | :--- | :--- |
| **`SUPER_ADMIN`** | `ROLE_SUPER_ADMIN` | All permissions (`EnumSet.allOf(Permission.class)`) | Multi-tenant setup, campus provisioning, global overrides |
| **`ADMIN`** | `ROLE_ADMIN` | `USER_*` (CRUD), `STUDENT_MANAGE`, `STUDENT_PROFILE_*`, `ATTENDANCE_READ`, `FEES_READ`, `INSTITUTION_READ` | Campus administrators, registrars, HR staff |
| **`FACULTY`** | `ROLE_FACULTY` | `ATTENDANCE_TAKE`, `ATTENDANCE_READ`, `ATTENDANCE_UPDATE`, `STUDENT_PROFILE_READ`, `INSTITUTION_READ` | Professors, lecturers, teaching assistants |
| **`STUDENT`** | `ROLE_STUDENT` | `STUDENT_PROFILE_READ`, `STUDENT_ATTENDANCE_READ`, `STUDENT_FEES_READ` | Enrolled students (self-service profile & dues) |
| **`PARENT`** | `ROLE_PARENT` | `STUDENT_PROFILE_READ`, `STUDENT_ATTENDANCE_READ`, `STUDENT_FEES_READ` | Parents/guardians monitoring student progress |
| **`FINANCE`** | `ROLE_FINANCE` | `FEES_MANAGE`, `FEES_READ`, `FEES_CREATE`, `STUDENT_PROFILE_READ` | Bursar's office, accountants, fee collection clerks |
| **`APPLICANT`** | `ROLE_APPLICANT`| `STUDENT_PROFILE_READ` | Prospective students tracking admission status |

---

## 5. Client-Side Security & Token Handling

### 5.1 Current Web Implementation
- **Storage:** Handled by Zustand (`useAuthStore.ts`) persisting to `localStorage` under key `auth-storage`.
- **Interceptors (`src/config/api.ts`):**
  - Outgoing HTTP calls inject `Authorization: Bearer <token>`.
  - On `401 Unauthorized`, requests are paused and queued in `failedQueue`.
  - Dispatches `POST /auth/refresh`. On success, tokens are updated and queued requests retry automatically.
  - On failure or missing refresh token, `logout()` is invoked and the browser redirects to `/login`.

### 5.2 Password Security & Hashing
- Backend implements `BCryptPasswordEncoder` configured in `PasswordConfig.java`.
- Passwords are never stored in plaintext.
- User password changes are enforced through `POST /auth/change-password` requiring current password verification.

---

## 6. Desktop-Specific Security Requirements

The transition from browser to desktop introduces unique operational security characteristics that must be addressed:

### 6.1 Token Storage Hardening (Blocker for Plaintext Storage)
- *Risk:* In a browser, `localStorage` is isolated per origin. In packaged desktop apps (Tauri or Electron), `localStorage` is stored in SQLite files or LevelDB folders on the user's local disk (`%APPDATA%`), vulnerable to malware and local forensic extraction.
- *Desktop Mitigation:* Tokens must be moved out of webview `localStorage` into the native OS vault:
  - **Windows:** Windows Data Protection API (DPAPI) / Windows Credential Manager via native plugin (e.g. `tauri-plugin-stronghold` or Electron `safeStorage`).

### 6.2 Cross-Origin Resource Sharing (CORS) & Webview Origin
- *Observation:* Desktop webviews do not run on `http://localhost:5173`. They run on custom URI schemes:
  - Tauri: `tauri://localhost` or `http://tauri.localhost`
  - Electron: `file://` or custom protocol `app://`
- *Backend Requirement:* `CorsConfig.java` in the backend must be updated to permit the desktop origin protocol to prevent preflight rejections.

### 6.3 Secure Desktop Session & Auto-Lock
- Desktop ERPs in institutional terminals (faculty rooms, cash counters) require inactivity auto-lock (e.g., 15 minutes of user inactivity locks the screen until password/PIN is re-entered).
