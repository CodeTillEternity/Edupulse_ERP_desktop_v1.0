# EduPulse ERP — Day 02, Work Package 10
# Deliverable 10b: Architecture Decision Register

## 1. Executive Summary & Register Overview

This Architecture Decision Register (ADR) formalizes the foundational technical decisions established during Week 00, Day 02 for **EduPulse ERP Windows Desktop Edition**.

To maintain technical honesty and prevent premature assumptions, every architectural decision is evaluated against actual codebase reality and assigned one of four definitive statuses:

```
+-----------------------------------------------------------------------------------+
|                        DECISION CLASSIFICATION STATUSES                           |
|                                                                                   |
|  [ ACCEPTED ]                                                                     |
|  Fully evaluated, verified against codebase capabilities, and approved.           |
|                                                                                   |
|  [ PROVISIONALLY ACCEPTED ]                                                       |
|  Approved in principle, but depends on external remediation or operational setup. |
|                                                                                   |
|  [ REQUIRES PROOF OF CONCEPT (POC) ]                                              |
|  Technically feasible, but requires empirical bench testing before locking design.|
|                                                                                   |
|  [ OPEN ]                                                                         |
|  Architecture trade-offs identified, but final policy decision remains unresolved.|
+-----------------------------------------------------------------------------------+
```

---

## 2. Architecture Decision Records (ADR-01 through ADR-16)

### ADR-01: Desktop Framework Selection — Tauri v2 over Electron
- **Status:** **ACCEPTED**
- **Context:** EduPulse requires a desktop edition to run on administrative and classroom Windows PCs across educational institutions. Many workstations are low-spec PCs (4 GB to 8 GB RAM, dual-core CPUs). Electron imposes an idle baseline of ~150–200 MB RAM and an installer size exceeding 100 MB.
- **Decision:** Adopt **Tauri v2** with Rust native core and Windows Evergreen WebView2 runtime.
- **Alternatives Considered:** Electron 33+, Flutter Desktop, React Native for Windows, Native C# (.NET MAUI / WPF).
- **Consequences:** Reduces memory footprint to ~35–45 MB and installer size to ~10–15 MB. Leverages the existing Windows OS-installed WebView2 runtime. Requires Rust toolchain proficiency for native Windows capabilities.
- **Security Implications:** Eliminates bundled Node.js runtime attack surface in the frontend; isolates JavaScript from raw OS calls via strictly gated IPC.
- **Implementation Dependencies:** Rust 1.80+, Node.js 20 LTS, `@tauri-apps/cli` v2.

---

### ADR-02: React Frontend Codebase Reuse & Native Shell Boundary
- **Status:** **ACCEPTED**
- **Context:** The project maintains an active web frontend (`frontend-v2/`) built with React 19.2, TypeScript 5.7, and Vite 8.3. Duplicating UI logic for desktop would double maintenance overhead.
- **Decision:** Directly bundle the production build (`frontend-v2/dist`) into the Tauri application shell. Abstract native services behind an environment-aware API adapter.
- **Alternatives Considered:** Rewrite UI in native Rust/Iced/Slint, separate desktop React codebase, maintain a shared component library.
- **Consequences:** Maximum code reuse (~95% identical code). Frontend detects desktop mode via `window.__TAURI_INTERNALS__`.
- **Security Implications:** Requires strict Content Security Policy (CSP) and capability declarations in Tauri to prevent untrusted web scripts from invoking native IPC.
- **Implementation Dependencies:** Unified API adapter in `frontend-v2/src/services/api.ts`.

---

### ADR-03: Spring Boot Sole Business Logic Authority & Downstream Client Invariant
- **Status:** **ACCEPTED**
- **Context:** Institutional ERP systems manage sensitive financial transactions, fee receipts, grading records, and role-based permissions that must never be bypassed.
- **Decision:** Java 21 + Spring Boot 3.3.4 remains the **sole authoritative business logic and transactional authority**. The desktop client is strictly a downstream projection with **zero offline write authority in release v1**.
- **Alternatives Considered:** Distributed two-way database replication (SymmetricDS, CouchDB/PouchDB), local business logic execution in Rust.
- **Consequences:** Guarantees zero split-brain data corruption. Drastically simplifies initial desktop implementation and compliance auditing.
- **Security Implications:** Local data can be wiped, corrupted, or inspected without compromising institutional ledger integrity.
- **Implementation Dependencies:** Spring Boot 3.3.4 REST endpoints on backend.

---

### ADR-04: PostgreSQL 17 as Authoritative System of Record (SoR)
- **Status:** **ACCEPTED**
- **Context:** EduPulse ERP manages 54 relational tables across 9 Flyway migrations. The backend uses Spring Data JPA / Hibernate 6.5 with HikariCP.
- **Decision:** Maintain **PostgreSQL 17** as the sole authoritative transactional database. Desktop clients never communicate directly with PostgreSQL; all interactions occur via Spring Boot REST APIs.
- **Alternatives Considered:** Exposing PostgreSQL port 5432 over VPN to desktop clients, migrating to document databases.
- **Consequences:** Protects database connection pool from campus WAN drops and prevents credential exposure in client binaries.
- **Security Implications:** Database credentials remain strictly on the backend server.
- **Implementation Dependencies:** PostgreSQL 17 server instance, Flyway V1–V9 migration pipeline.

---

### ADR-05: Local Encrypted Cache Engine — SQLCipher
- **Status:** **ACCEPTED**
- **Context:** Desktop clients require offline access to student rosters, timetables, and circulars during campus network drops. Local data at rest on Windows workstations must be protected against physical theft.
- **Decision:** Implement **SQLCipher** (SQLite 3 with 256-bit AES encryption via `rusqlite` bundled) at `%LOCALAPPDATA%\com.edupulse.erp\data\cache.db`.
- **Alternatives Considered:** Plain SQLite, IndexedDB in WebView2, encrypted flat JSON files, RocksDB.
- **Consequences:** Delivers sub-millisecond query performance, ACID reliability, and complete cryptographic confidentiality at rest.
- **Security Implications:** Full page-level encryption prevents offline extraction of student PII from stolen laptops.
- **Implementation Dependencies:** Rust crate `rusqlite` with `bundled-sqlcipher` feature.

---

### ADR-06: Windows DPAPI for Credential and Database Master Key Storage
- **Status:** **PROVISIONALLY ACCEPTED**
- **Context:** Browser `localStorage` is vulnerable to XSS exfiltration. Plaintext configuration files leak secrets.
- **Decision:** Protect persistent refresh tokens and the 256-bit SQLCipher master encryption key using Windows **Data Protection API (DPAPI)** via `CryptProtectData` (`CurrentUser` scope with application entropy salt).
- **Alternatives Considered:** Windows Credential Manager, raw registry keys, hardcoded application secret, TPM-only hardware keys.
- **Consequences:** Cryptographic keys and tokens are decrypted strictly in memory at runtime and bound to the logged-in Windows OS user.
- **Security Implications & Provisional Condition:** DPAPI user scope decrypts for any process running under that Windows account. If multiple staff share a generic Windows login (e.g. `staff-pc`), DPAPI does not isolate them. **Remediation Condition:** Desktop must implement an explicit "Shared PC Mode" that purges the DPAPI vault on application logout.
- **Implementation Dependencies:** Windows native integration via `windows-sys` or `dpapi` crate.

---

### ADR-07: Configurable Backend Server Endpoint & Fallback Hierarchy
- **Status:** **ACCEPTED**
- **Context:** The same desktop binary must support cloud-hosted multi-tenant SaaS backends (`https://api.edupulse.org`) and on-premise private campus servers (`https://erp.campus.edu`) without recompilation.
- **Decision:** Implement a deterministic four-tier configuration fallback chain:
  1. CLI Argument: `--backend-url <URL>`
  2. Enterprise Registry / GPO: `HKLM\SOFTWARE\Policies\EduPulse\BackendUrl`
  3. Local Configuration File: `%APPDATA%\com.edupulse.erp\config.json`
  4. Embedded Fallback: `https://api.edupulse.org`
- **Alternatives Considered:** Hardcoded backend URL, build-time environment variable (`VITE_API_URL`) baking separate installers per campus.
- **Consequences:** Single distributable installer serves all deployment models.
- **Security Implications:** Enforces strict HTTPS/TLS 1.3; rejects unencrypted HTTP URLs (except `localhost` during development).
- **Implementation Dependencies:** Rust configuration loader module.

---

### ADR-08: Desktop Origin Scheme and Strict Backend CORS Policy
- **Status:** **ACCEPTED**
- **Context:** Tauri v2 on Windows serves web assets from `http://tauri.localhost` (or `https://tauri.localhost` when configured). The existing Spring Boot backend currently uses an insecure wildcard pattern (`allowedOriginPatterns("*")` with `allowCredentials(true)`).
- **Decision:** Standardize desktop origin to `https://tauri.localhost` (via `app.windows.useHttpsScheme = true`). Update Spring Boot CORS configuration to explicitly whitelist registered production domains and the exact Tauri desktop origin.
- **Alternatives Considered:** Leaving backend CORS as wildcard `*`, proxying all API requests through Rust native HTTP client.
- **Consequences:** Eliminates CORS vulnerability while maintaining direct, high-performance WebView2-to-Spring Boot HTTP communication.
- **Security Implications:** Closes unauthorized cross-origin credentialed request attack vector.
- **Implementation Dependencies:** Spring Boot `CorsConfig.java` refactoring.

---

### ADR-09: In-Memory Short-Lived Access Tokens and Persistent Refresh Token Rotation
- **Status:** **PROVISIONALLY ACCEPTED**
- **Context:** Web frontend currently stores JWT tokens in `localStorage`. Desktop requires high resistance to memory and disk extraction.
- **Decision:** Short-lived access tokens (15-min TTL) are stored **strictly in JavaScript/React memory**. Persistent refresh tokens (7-day TTL) are stored in the DPAPI vault and rotated on every use.
- **Alternatives Considered:** Storing access tokens in DPAPI, keeping both tokens in memory only (requires login on every app restart).
- **Consequences:** Seamless user experience across app restarts without exposing access tokens on disk.
- **Security Implications & Provisional Condition:** **Blocked by existing backend authentication defects:**
  1. Backend token blacklist is held in an in-memory `ConcurrentHashMap` and lost on server restart.
  2. Backend active session map is keyed only by username, terminating multi-device logins.
  3. Backend logout endpoint does not revoke refresh tokens.
  4. Backend lacks refresh-token family reuse invalidation.
  **Remediation Condition:** Implementation is blocked until backend authentication remediation (SEC-01 through SEC-06) is completed.
- **Implementation Dependencies:** Spring Boot backend auth service remediation.

---

### ADR-10: Strict Read-Only Offline Access in Release v1
- **Status:** **ACCEPTED**
- **Context:** Implementing two-way offline write synchronization with conflict resolution (grades, attendance, fees) requires distributed consensus logic and introduces severe financial/academic corruption risks.
- **Decision:** In Release v1, offline capability is **strictly read-only**. Teachers and administrators can view cached rosters, timetables, and circulars. All create, update, and delete actions require live backend connectivity.
- **Alternatives Considered:** Client-side optimistic write queues with replay, two-way master-master replication.
- **Consequences:** Accelerates time-to-market, ensures 100% data integrity, and eliminates split-brain risks.
- **Security Implications:** Zero risk of offline tampering with student balances or grade books.
- **Implementation Dependencies:** UI offline detection and action disabling in React.

---

### ADR-11: Dual-Phase Deletion Synchronization and Reconciliation
- **Status:** **PROVISIONALLY ACCEPTED**
- **Context:** EduPulse PostgreSQL database entities do not implement soft deletes; records are hard-deleted. Naive delta sync queries (`WHERE updated_at > :since`) cannot detect deleted rows.
- **Decision:** Adopt a dual-phase deletion synchronization architecture:
  - *Phase 1 (Short-Term Workaround):* Module-level cryptographic SHA-256 hash check (`GET /api/v1/sync/{module}/reconciliation-hash`). If local hash diverges from server, client fetches active IDs and prunes missing rows.
  - *Phase 2 (Long-Term Architecture):* Introduce an authoritative `deletion_tombstones` table in PostgreSQL populated by JPA entity listeners.
- **Alternatives Considered:** Full database re-download on every sync, polling individual records.
- **Consequences:** Solves the zombie record problem without delaying desktop client development.
- **Security Implications & Provisional Condition:** Phase 1 requires empirical performance validation (POC) on backend hash calculation for large datasets (> 50,000 records).
- **Implementation Dependencies:** Backend sync endpoints.

---

### ADR-12: Institution-Specific Single-Tenant Model & Endpoint Switching Sanitization
- **Status:** **ACCEPTED**
- **Context:** Database audit confirmed that EduPulse is architected as an institution-specific database (tables lack `tenant_id` columns; global unique constraints exist).
- **Decision:** The desktop client connects to a single institution backend at a time. Switching backend URLs triggers **mandatory cryptographic cache shredding** (wiping the SQLCipher database and DPAPI key) before connecting to the new endpoint.
- **Alternatives Considered:** Multi-tenant local cache with `tenant_id` scoping in SQLite.
- **Consequences:** Prevents any possibility of institutional data leakage across tenant boundaries.
- **Security Implications:** Zero residual data contamination between institutions.
- **Implementation Dependencies:** Rust cache manager purge protocol.

---

### ADR-13: Dual Enterprise Packaging — WiX MSI and NSIS
- **Status:** **ACCEPTED**
- **Context:** Educational institutions require both centralized IT deployment (Active Directory GPO / Microsoft Intune) and direct download by individual faculty on personal laptops.
- **Decision:** Support two distinct Windows installer targets:
  - **WiX Toolset v3/v4 (.msi):** Per-machine enterprise installer (`Program Files\EduPulse ERP\`) supporting silent flags and `BACKEND_URL` command-line properties.
  - **NSIS (.exe):** Per-user standalone installer (`%LOCALAPPDATA%\Programs\EduPulse ERP\`) requiring zero administrative privileges.
- **Alternatives Considered:** MSIX / Windows Store package, standalone ZIP archive.
- **Consequences:** Maximizes enterprise adoption while maintaining ease of use for faculty.
- **Security Implications:** MSI enforces secure per-machine installation directory ACLs.
- **Implementation Dependencies:** WiX Toolset v3.14/v4, NSIS 3.x, Tauri bundler config.

---

### ADR-14: Independent Desktop CI/CD Build and Release Pipeline
- **Status:** **ACCEPTED**
- **Context:** Desktop releases must not trigger unnecessary web deployments, and web releases must not block desktop distribution.
- **Decision:** Maintain a dedicated GitHub Actions workflow (`.github/workflows/desktop-ci.yml`) triggered on desktop version tags (`desktop-v*`) and desktop path changes, running on Windows Server 2022 runners.
- **Alternatives Considered:** Merging desktop build into existing `backend-ci.yml` and `frontend-ci.yml`.
- **Consequences:** Decoupled release cadences and isolated artifact generation.
- **Security Implications:** Desktop signing keys and release secrets are isolated to desktop workflow environments.
- **Implementation Dependencies:** GitHub Actions Windows runner, Tauri action.

---

### ADR-15: Windows Code Signing via Azure Trusted Signing / Authenticode
- **Status:** **PROVISIONALLY ACCEPTED**
- **Context:** Unsigned Windows binaries trigger Microsoft Defender SmartScreen blocking screens ("Windows protected your PC"), undermining institutional trust.
- **Decision:** Sign all desktop executables, DLLs, and installers using standard Authenticode or **Azure Trusted Signing** (cloud HSM-backed code signing).
- **Alternatives Considered:** Physical USB hardware token EV certificate, distributing self-signed binaries.
- **Consequences:** Seamless installation without SmartScreen warnings; automated CI/CD signing without physical tokens.
- **Provisional Condition:** Requires educational organization identity verification and Azure Trusted Signing subscription setup.
- **Implementation Dependencies:** Azure Trusted Signing dlib / SignTool integration in CI.

---

### ADR-16: In-App Auto-Updates via Tauri Plugin with Ed25519 Signatures
- **Status:** **ACCEPTED**
- **Context:** Desktop software requires rapid distribution of critical security patches and feature updates without requiring manual re-installation by users.
- **Decision:** Implement **`tauri-plugin-updater`** using cryptographic **Ed25519** digital signatures. Update metadata is pulled over HTTPS, and payloads are verified prior to execution.
- **Alternatives Considered:** Custom updater script, Squirrel.Windows, external updater executable.
- **Consequences:** Native, secure auto-update mechanism built into Tauri runtime.
- **Security Implications:** Compromised CDN cannot inject malicious updates without the private Ed25519 signing key.
- **Implementation Dependencies:** Ed25519 keypair generation, GitHub Releases / static update server.

---

## 3. Prioritized Unresolved-Decision & Proof of Concept Register

The following engineering decisions require empirical validation or policy determination prior to production freeze:

| ID | Title | Category | Current Status | Risk / Impact | Blocking Phase |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **POC-01** | **WebView2 Origin Scheme & Custom Headers** | Technical POC | Requires POC | Need to verify if `useHttpsScheme = true` (`https://tauri.localhost`) properly handles custom `Authorization` headers in WebView2 without origin issues. | Phase 0 |
| **POC-02** | **MSI `BACKEND_URL` Property Ingestion** | Packaging POC | Requires POC | Need to validate that WiX MSI custom action writes `BACKEND_URL` to registry where Rust can read it without requiring runtime elevation. | Phase 1 |
| **POC-03** | **SHA-256 Deletion Hash Performance** | Performance POC | Requires POC | Benchmark Spring Boot backend performance computing SHA-256 reconciliation hashes over 50,000+ student records. | Phase 5 |
| **OPEN-01** | **WebView2 Distribution on Air-Gapped Networks** | Policy / Arch | Open | Decide whether to require Evergreen WebView2 download or provide an optional Fixed Version Runtime bundle (+150 MB) for air-gapped campuses. | Phase 6 |
| **OPEN-02** | **Multi-User Shared Windows Account Isolation** | Security Policy | Open | Determine whether "Shared PC Mode" (purge cache on logout) should be forced by default across all institutional installations. | Phase 3 |
