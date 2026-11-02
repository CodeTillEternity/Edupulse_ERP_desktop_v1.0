# EduPulse ERP — Day 02, Work Package 10
# Deliverable 10d: Phased Desktop Implementation Roadmap

## 1. Executive Summary & Delivery Strategy

This roadmap defines the phased implementation strategy for **EduPulse ERP Windows Desktop Edition**. It translates the architectural blueprints established in WP01 through WP09 into an actionable, dependency-linked engineering progression across eight distinct phases (Phase 0 through Phase 7).

To maintain engineering integrity, development milestones are governed by **observable technical exit criteria** rather than arbitrary calendar dates.

```
+-----------------------------------------------------------------------------------+
|                        PHASED IMPLEMENTATION PROGRESSION                          |
|                                                                                   |
|  [ Phase 0: Implementation Preparation & Backend Auth Remediation ]               |
|      |                                                                            |
|  [ Phase 1: Desktop Shell & React Frontend Integration ]                          |
|      |                                                                            |
|  [ Phase 2: Dynamic Backend Connectivity & API Compatibility ]                    |
|      |                                                                            |
|  [ Phase 3: Native Security, DPAPI Vault & Secure IPC ]                           |
|      |                                                                            |
|  [ Phase 4: Offline Foundation & Encrypted SQLCipher Cache ]                      |
|      |                                                                            |
|  [ Phase 5: Keyset Synchronization & Deletion Reconciliation ]                    |
|      |                                                                            |
|  [ Phase 6: Enterprise Packaging, Code Signing & Updates ]                        |
|      |                                                                            |
|  [ Phase 7: Verification, Hardening & Institutional Pilot ]                       |
+-----------------------------------------------------------------------------------+
```

---

## 2. Phase 0: Implementation Preparation & Backend Remediation

### 2.1 Objective
Establish developer and CI build environments, conduct mandatory technical proofs of concept, and remediate critical backend authentication security defects that block desktop credential persistence.

### 2.2 Tasks
- **Task 0.1 (Backend Auth Remediation):**
  - Replace in-memory `TokenBlacklistService` with persistent Redis or PostgreSQL table.
  - Update session tracking to key by `(username, deviceId/sessionId)` rather than username alone.
  - Add refresh token invalidation in `/api/v1/auth/logout`.
  - Implement `credentials_updated_at` claim validation in `JwtAuthenticationFilter` to invalidate active tokens on password change.
  - Implement refresh-token family reuse detection and immediate child revocation.
  - Refactor `CorsConfig.java` to eliminate wildcard `*` with credentials and explicitly whitelist `https://tauri.localhost`.
- **Task 0.2 (Technical POCs):**
  - Execute POC-01: Validate WebView2 HTTPS origin scheme (`https://tauri.localhost`) and custom authorization header propagation.
  - Execute POC-02: Test Windows DPAPI `CryptProtectData` round-trip in standalone Rust console binary.
- **Task 0.3 (Toolchain Setup):** Ensure MSVC C++ Build Tools, Rust 1.80+, Node.js 20+, and Cargo are configured on developer and Windows CI runners.

### 2.3 Dependencies
- Access to backend repository and CI environments.
- Windows development workstation.

### 2.4 Expected Deliverables
- Remediated Spring Boot authentication service with passing unit/integration tests.
- Standalone DPAPI Rust proof-of-concept crate.
- Validated CORS and WebView2 communication test report.

### 2.5 Technical Risks
- Refactoring backend session maps could impact existing web clients if not tested against web integration suites.

### 2.6 Acceptance Criteria & Exit Conditions
- All six backend security defects (SEC-01 through SEC-06) pass automated integration tests.
- Backend CORS configuration successfully responds to preflight `OPTIONS` requests from `https://tauri.localhost` with HTTP 200 and explicit `Access-Control-Allow-Origin`.

---

## 3. Phase 1: Desktop Shell & Frontend Integration

### 3.1 Objective
Initialize the Tauri v2 desktop application shell, configure WebView2 host parameters, and package the existing React 19.2 frontend into a launchable Windows desktop executable.

### 3.2 Tasks
- **Task 1.1:** Initialize Tauri v2 project structure (`src-tauri/`) in repository root.
- **Task 1.2:** Configure `src-tauri/tauri.conf.json`:
  - Set `app.windows.useHttpsScheme = true` (`https://tauri.localhost`).
  - Configure CSP to restrict script-src and connect-src.
  - Define minimum window dimensions (1280x800) and window decorations.
- **Task 1.3:** Configure `frontend-v2/vite.config.ts` to output compatible static assets for Tauri bundling.
- **Task 1.4:** Implement runtime environment detector (`isDesktop()`) in `frontend-v2/src/utils/environment.ts`.
- **Task 1.5:** Build and launch debug and release binaries (`npm run tauri dev` and `npm run tauri build`).

### 3.3 Dependencies
- Phase 0 toolchain setup complete.

### 3.4 Expected Deliverables
- Working `src-tauri/` project directory.
- Compiled `EduPulse.exe` launching into the React ERP dashboard.

### 3.5 Technical Risks
- Relative asset pathing issues or React Router history mode incompatibilities inside WebView2.

### 3.6 Acceptance Criteria & Exit Conditions
- `EduPulse.exe` launches cleanly on Windows 10/11 without blank screens, script errors, or console exceptions.
- Memory usage at idle remains under 50 MB RAM.

---

## 4. Phase 2: Dynamic Backend Connectivity & API Compatibility

### 4.1 Objective
Implement dynamic backend URL resolution, connectivity health checks, and secure HTTP communication between the desktop webview and remote Spring Boot endpoints.

### 4.2 Tasks
- **Task 2.1:** Implement Rust configuration manager reading the 4-tier fallback hierarchy (CLI flag -> Registry/GPO -> `config.json` -> default).
- **Task 2.2:** Expose `get_backend_url` and `test_backend_connection` Tauri IPC commands.
- **Task 2.3:** Update frontend Axios client to dynamically fetch the configured backend base URL at application boot.
- **Task 2.4:** Build UI "Server Configuration & Connection Test" screen allowing administrators to change and validate backend URLs.
- **Task 2.5:** Implement automated background network ping checking `/api/v1/health` every 60 seconds.

### 4.3 Dependencies
- Phase 1 desktop shell operational.
- Remediated backend CORS configuration.

### 4.4 Expected Deliverables
- Rust configuration management module.
- Frontend connection status indicator and settings modal.

### 4.5 Technical Risks
- Institutional firewalls or proxy servers blocking direct Axios requests.

### 4.6 Acceptance Criteria & Exit Conditions
- Application successfully switches between cloud and local test backend URLs without restarting or recompiling the binary.
- Network disconnection immediately updates the UI status badge from "Online" to "Disconnected".

---

## 5. Phase 3: Native Security, DPAPI Vault & Secure IPC

### 5.1 Objective
Eliminate insecure `localStorage` token storage in desktop mode by implementing a hardware/OS-bound DPAPI credential vault in Rust and securing all IPC channels.

### 5.2 Tasks
- **Task 3.1:** Implement Rust DPAPI vault module using `CryptProtectData` and `CryptUnprotectData` with application entropy salt.
- **Task 3.2:** Expose secure IPC commands: `store_credentials`, `get_refresh_token`, `clear_credentials`.
- **Task 3.3:** Integrate `zeroize` trait to scrub decrypted token buffers from memory upon completion.
- **Task 3.4:** Define strict Tauri capability files (`capabilities/default.json`) restricting IPC command invocation.
- **Task 3.5:** Refactor frontend authentication store (Zustand):
  - Store short-lived access token (15-min TTL) in memory only.
  - Delegate persistent refresh token storage to Rust DPAPI vault.
  - Implement proactive token refresh interceptor using decrypted DPAPI refresh token.
- **Task 3.6:** Implement "Shared PC Mode" settings toggle that clears DPAPI credentials and cache on application logout.

### 5.3 Dependencies
- Phase 0 backend auth fixes.
- Phase 2 backend connectivity.

### 5.4 Expected Deliverables
- Rust DPAPI credential vault module.
- In-memory authentication state manager in React.
- IPC capability security definitions.

### 5.5 Technical Risks
- IPC latency during token refresh causing request queue stalls.

### 5.6 Acceptance Criteria & Exit Conditions
- Inspection of Windows AppData, registry, and browser storage confirms zero plaintext tokens or passwords at rest.
- Closing and reopening the application restores user session using the protected refresh token without prompting for credentials.

---

## 6. Phase 4: Offline Foundation & Encrypted SQLCipher Cache

### 6.1 Objective
Embed and configure SQLCipher 256-bit AES database in the desktop client to provide high-speed, secure local caching for operational ERP datasets.

### 6.2 Tasks
- **Task 4.1:** Add `rusqlite` with `bundled-sqlcipher` to `src-tauri/Cargo.toml`.
- **Task 4.2:** Implement database initialization lifecycle:
  - Generate 256-bit master key via OS CSPRNG on first launch; encrypt via DPAPI into `cache.key`.
  - Open `%LOCALAPPDATA%\com.edupulse.erp\data\cache.db` with WAL mode and `temp_store = MEMORY`.
  - Apply startup integrity check via `PRAGMA quick_check;`.
- **Task 4.3:** Implement local schema migrations (`rusqlite_migration`) for `sync_metadata`, `cached_students`, `cached_timetables`, and `cached_announcements`.
- **Task 4.4:** Expose read-only cached query IPC commands to frontend:
  - `query_cached_students`, `query_cached_timetable`, `query_cached_notices`.
- **Task 4.5:** Implement UI offline mode detection: when offline, route view queries to local cache commands and render "Offline Read-Only" banner.

### 6.3 Dependencies
- Phase 3 DPAPI vault available for master key protection.

### 6.4 Expected Deliverables
- Embedded SQLCipher database module in Rust.
- Local schema migration scripts.
- Offline-aware UI views for timetables, rosters, and announcements.

### 6.5 Technical Risks
- C-compiler / MSVC linker issues compiling SQLCipher C amalgamation on Windows.

### 6.6 Acceptance Criteria & Exit Conditions
- Disconnecting network adapter allows user to continue browsing student rosters, timetables, and circulars with < 20 ms render times.
- Attempting to view the raw `cache.db` file in external hex editors reveals 100% encrypted ciphertext.

---

## 7. Phase 5: Keyset Synchronization & Deletion Reconciliation

### 7.1 Objective
Implement one-way downstream synchronization between PostgreSQL and SQLCipher, resolving the hard-delete gap through cryptographic hash reconciliation and keyset pagination.

### 7.2 Tasks
- **Task 5.1 (Backend):** Implement synchronization endpoints in Spring Boot:
  - `GET /api/v1/sync/{module}?since={ts}&limit={limit}` (Keyset paginated delta sync).
  - `GET /api/v1/sync/{module}/reconciliation-hash` (SHA-256 hash across active IDs and updated_at).
  - `GET /api/v1/sync/{module}/active-id-manifest` (Compact active ID list for pruning).
- **Task 5.2 (Desktop):** Implement background Tokio sync worker in Rust:
  - Immediate SQLite transaction wrapping for upserts.
  - Keyset cursor progression and HTTP ETag evaluation.
  - Periodic 15-minute background polling and manual refresh trigger.
- **Task 5.3 (Deletion Reconciliation):** Implement Phase 1 set difference pruning: if server hash diverges, fetch active ID manifest and execute `DELETE FROM cached_{table} WHERE id NOT IN (...)`.
- **Task 5.4:** Implement storage quota manager enforcing 250 MB ceiling with LRU eviction of expired notices and old attendance.

### 7.3 Dependencies
- Phase 4 SQLCipher local database operational.
- Backend synchronization controllers deployed.

### 7.4 Expected Deliverables
- Spring Boot sync API module.
- Rust background synchronization engine.
- Deletion reconciliation sub-routine.

### 7.5 Technical Risks
- High memory usage on backend when generating active ID manifests for large student populations (> 20,000 records).

### 7.6 Acceptance Criteria & Exit Conditions
- Creating a student on the web backend propagates to the desktop cache within 15 minutes (or immediately on manual sync).
- Hard-deleting a student on the backend removes the student from the desktop cache on the subsequent reconciliation cycle.
- Abrupt network termination during sync does not corrupt the local SQLite database.

---

## 8. Phase 6: Enterprise Packaging, Code Signing & Updates

### 8.1 Objective
Produce production-ready, digitally signed Windows installers (.msi and .exe), establish automated CI/CD release workflows, and enable cryptographically verified in-app updates.

### 8.2 Tasks
- **Task 6.1:** Configure WiX Toolset in Tauri bundler to produce per-machine enterprise `.msi` supporting silent install (`/qn`) and registry property ingestion (`BACKEND_URL`).
- **Task 6.2:** Configure NSIS bundler to produce per-user standalone `.exe` installer.
- **Task 6.3:** Integrate Microsoft SignTool with Azure Trusted Signing or institutional Authenticode certificate in CI/CD.
- **Task 6.4:** Configure `tauri-plugin-updater` with Ed25519 signing keypair.
- **Task 6.5:** Create dedicated GitHub Actions workflow (`.github/workflows/desktop-ci.yml`) triggered on desktop version tags.
- **Task 6.6:** Implement structured local diagnostic logging using `tracing-appender` with automatic PII sanitization.

### 8.3 Dependencies
- Phases 1 through 5 complete.
- Code signing certificate procured.

### 8.4 Expected Deliverables
- Verified `.msi` and `.exe` installer binaries.
- Automated GitHub Actions desktop release workflow.
- Update server manifest generator.

### 8.5 Technical Risks
- SmartScreen false positives on initial releases until certificate builds reputation.

### 8.6 Acceptance Criteria & Exit Conditions
- MSI installs silently via `msiexec /i EduPulse.msi /qn BACKEND_URL=https://erp.campus.edu` and successfully configures the backend endpoint.
- Binaries pass Windows Defender scan with zero detections and valid digital signatures.

---

## 9. Phase 7: Verification, Hardening & Institutional Pilot

### 9.1 Objective
Execute comprehensive security penetration testing, performance benchmarking, and a multi-workstation institutional pilot across participating campus environments.

### 9.2 Tasks
- **Task 7.1 (Security Hardening):** Conduct comprehensive penetration testing (DPAPI dump attempts, memory inspection, IPC fuzzing, CORS verification, XSS injection probes).
- **Task 7.2 (Performance Benchmarking):** Verify cold boot time (< 2.5 seconds), memory footprint (< 60 MB RAM), and query latency (< 15 ms).
- **Task 7.3 (Network Resilience):** Test desktop client behavior under high latency (500 ms), 50% packet loss, and abrupt Wi-Fi disconnection.
- **Task 7.4 (Institutional Pilot):** Deploy `.msi` to 25 staff workstations across two pilot campuses; monitor diagnostic logs and user feedback for two weeks.
- **Task 7.5 (Production Readiness Review):** Review all pilot telemetry, resolve critical bug reports, and issue final production release sign-off.

### 9.3 Dependencies
- Phase 6 signed installers available.
- Pilot campus infrastructure provisioned.

### 9.4 Expected Deliverables
- Security penetration test report.
- Performance benchmark scorecard.
- Institutional pilot sign-off documentation.

### 9.5 Technical Risks
- Local workstation antivirus software flagging auto-update background worker.

### 9.6 Acceptance Criteria & Exit Conditions
- Zero high or critical security vulnerabilities detected.
- Zero data corruption or unrecoverable crash incidents during the two-week institutional pilot.
- Formal sign-off by institutional IT administrators.

---

## 10. Critical Path Analysis

The critical implementation path is strictly sequential across core architectural boundaries:

```
[ Phase 0: Backend Auth Fixes ]
              │
              ▼
[ Phase 1: Tauri Shell Initialization ]
              │
              ▼
[ Phase 2: Dynamic Backend URL & CORS ]
              │
              ▼
[ Phase 3: DPAPI Vault & Token In-Memory State ]
              │
              ▼
[ Phase 4: SQLCipher Local Database & Schema ]
              │
              ▼
[ Phase 5: Backend Keyset Sync & Deletion Hash APIs ]
              │
              ▼
[ Phase 6: MSI Enterprise Packaging & Code Signing ]
              │
              ▼
[ Phase 7: Institutional Pilot & Release Sign-Off ]
```

**Critical Path Bottlenecks:**
1. **Backend Auth Remediation (Phase 0):** Blocks Phase 3; desktop cannot securely manage refresh tokens until backend fixes token revocation and session tracking.
2. **Backend Sync APIs (Phase 5):** Blocks complete offline readiness; desktop cannot keep cache fresh without backend delta endpoints.
3. **Code Signing Setup (Phase 6):** Blocks enterprise institutional pilot; unsigned installers will be blocked by campus Windows Defender policies.
