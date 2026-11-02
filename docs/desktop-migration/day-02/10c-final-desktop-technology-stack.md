# EduPulse ERP — Day 02, Work Package 10
# Deliverable 10c: Final Desktop Technology Stack Baseline

## 1. Executive Summary & Technology Governance

This document establishes the authoritative technology stack baseline for the **EduPulse ERP Windows Desktop Edition**. It catalogues all verified existing technologies currently deployed in the repository (`frontend-v2/` and `backend/`) and rigorously defines the proposed native desktop dependencies required for Tauri v2, Rust, Windows DPAPI, and SQLCipher.

To avoid premature assumptions, every component is explicitly classified as **Verified Existing** or **Proposed Desktop Dependency**, accompanied by compatibility constraints and pre-implementation verification requirements.

```
+-----------------------------------------------------------------------------------+
|                        TECHNOLOGY BASELINE CLASSIFICATION                         |
|                                                                                   |
|  [ VERIFIED EXISTING ]                                                            |
|  Directly verified in repository configuration files (pom.xml, package.json, CI)  |
|                                                                                   |
|  [ PROPOSED DESKTOP DEPENDENCY ]                                                  |
|  Required for native desktop integration, evaluated during Day 02 architecture    |
|                                                                                   |
|  [ UNVERIFIED / PROOF OF CONCEPT REQUIRED ]                                       |
|  Requires bench validation before finalizing release builds                       |
+-----------------------------------------------------------------------------------+
```

---

## 2. Frontend Technology Stack (`frontend-v2/`)

The desktop application bundles the production build produced by the existing `frontend-v2/` workspace without duplicating UI code:

| Technology | Role / Purpose | Classification | Verified Version | Compatibility & Security Notes | Pre-Implementation Verification |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **React** | Core component rendering engine | **Verified Existing** | `^19.2.8` | High performance, React Server Component (RSC) compatible. | Verify zero hydration warnings in WebView2 local environment. |
| **React DOM** | DOM binding for React | **Verified Existing** | `^19.2.8` | Standard browser/WebView2 DOM mount. | Verify compatibility with Tauri webview container. |
| **Vite** | Build tool and dev server | **Verified Existing** | `^8.3.0` | Next-gen bundler; ultra-fast HMR and tree-shaken production output. | Confirm `base: './'` or relative asset pathing for `tauri.localhost`. |
| **TypeScript** | Type-safe application development | **Verified Existing** | `~6.0.2` | Strict mode typing across all views and models. | Run `npx tsc -b` clean before desktop bundle packaging. |
| **Tailwind CSS** | Utility-first styling framework | **Verified Existing** | `^3.4.19` | Responsive layouts, CSS variables, dark/light theme support. | Ensure PostCSS and Autoprefixer compile cleanly into static CSS. |
| **Axios** | HTTP client for REST APIs | **Verified Existing** | `^1.20.0` | Manages backend HTTP requests, interceptors, and timeout handling. | Configure desktop interceptors for dynamic backend URL injection. |
| **Zustand** | In-memory client state management | **Verified Existing** | `^5.0.15` | Holds active session, navigation state, and in-memory access token. | Verify that access tokens remain in-memory and are never mirrored to disk. |
| **TanStack Query** | Server state caching & mutation | **Verified Existing** | `^5.104.1` | Network query caching, optimistic UI updates, retry logic. | Configure offline pause-and-resume behavior during network drops. |
| **React Router** | Client-side view routing | **Verified Existing** | `^7.18.4` | Single-page application navigation across the 23 ERP domains. | Configure browser history mode compatible with WebView2 local origin. |
| **Lucide React** | Application icon library | **Verified Existing** | `^1.52.0` | Comprehensive icon set for ERP modules. | Tree-shaken during build; zero runtime overhead. |
| **Sonner** | Toast notifications | **Verified Existing** | `^2.0.8` | User alerts for sync status, connectivity changes, and errors. | Ensure toasts render above modal dialogs in desktop resolution. |
| **@tauri-apps/api** | Frontend IPC bridge to Rust | **Proposed Desktop** | `^2.x` | Exposes `invoke()`, event listening, and window controls to React. | Add as desktop dependency; wrap behind abstract API adapter. |

---

## 3. Desktop Shell & Native Host Stack

The native host layer is compiled into a standalone Windows binary (`EduPulse.exe`):

| Technology | Role / Purpose | Classification | Target Version | Compatibility & Security Notes | Pre-Implementation Verification |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Tauri Core** | Desktop application framework | **Proposed Desktop** | `^2.x` (v2.2+) | Lightweight, memory-efficient alternative to Electron. | Initialize Tauri in root using `npm run tauri init`. |
| **Rust Toolchain** | Native runtime compiler | **Proposed Desktop** | `1.80+ (stable)` | Targets `x86_64-pc-windows-msvc`. Memory-safe systems programming. | Verify MSVC C++ build tools installed on developer and CI machines. |
| **Microsoft WebView2** | Embedded browser rendering engine | **Verified OS Runtime** | Evergreen (120+) | Native Windows component; evergreen auto-updates via Windows Update. | Validate bootstrapper download behavior on restricted campus networks. |
| **windows-sys** | Win32 API system bindings | **Proposed Desktop** | `^0.59` | Low-level Win32 calls for DPAPI, registry, and process monitoring. | Verify minimal feature flags (`Win32_Security_Cryptography`, etc.). |
| **Tokio** | Asynchronous Rust runtime | **Proposed Desktop** | `^1.40` | Powers background sync worker, health checks, and non-blocking I/O. | Multi-threaded runtime; ensure thread pool does not starve UI events. |
| **tracing / tracing-appender** | Structured diagnostic logging | **Proposed Desktop** | `^0.1 / ^0.2` | Rolling local diagnostic logs (`%LOCALAPPDATA%\...\logs\`). | Implement automatic PII sanitization in log formatter. |
| **tauri-plugin-updater** | In-app auto-update framework | **Proposed Desktop** | `^2.x` | Pulls signed update manifests and binaries over HTTPS. | Generate Ed25519 signing keypair and securely store private key in CI. |

---

## 4. Security & Cryptographic Stack

| Technology | Role / Purpose | Classification | Target Version | Compatibility & Security Notes | Pre-Implementation Verification |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Windows DPAPI** | Hardware/OS-bound secret vault | **Proposed Desktop** | Win32 Native | `CryptProtectData` binds ciphertext to the logged-in Windows user account. | Verify entropy salt and `CurrentUser` scope implementation. |
| **SQLCipher** | Encrypted local database engine | **Proposed Desktop** | `4.5+` | SQLite 3 with 256-bit AES page encryption via `rusqlite` (`bundled-sqlcipher`). | Test compilation on Windows MSVC without OpenSSL linker collisions. |
| **zeroize** | Memory-scrubbing trait | **Proposed Desktop** | `^1.8` | Automatically wipes encryption keys and plaintext secrets from memory on drop. | Verify compiler does not optimize away zeroization routines. |
| **Ed25519** | Digital signature scheme for updates | **Proposed Desktop** | RFC 8032 | Cryptographically verifies desktop update payloads before installation. | Test signature verification failure with intentionally tampered binary. |
| **Azure Trusted Signing** | Enterprise Windows code signing | **Proposed Desktop** | Cloud HSM | Authenticode signing; eliminates Microsoft SmartScreen untrusted warnings. | Validate Azure tenant onboarding and SignTool integration in GitHub Actions. |
| **Tauri Capabilities** | Granular IPC command permissions | **Proposed Desktop** | v2 Schema | Declarative JSON configuration restricting which IPC commands webview can call. | Audit `src-tauri/capabilities/default.json` for least-privilege access. |

---

## 5. Backend Technology Stack (`backend/`)

The backend is an enterprise Spring Boot application serving authoritative REST endpoints:

| Technology | Role / Purpose | Classification | Verified Version | Compatibility & Security Notes | Pre-Implementation Verification |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Java Development Kit** | Enterprise backend execution environment | **Verified Existing** | `21 (LTS)` | Eclipse Temurin distribution. Modern virtual threads and language features. | Confirmed in `pom.xml` (`<java.version>21</java.version>`) and CI. |
| **Spring Boot** | Core application framework | **Verified Existing** | `3.3.4` | Modern enterprise microservice stack; native Jakarta EE 10 baseline. | Confirmed in `pom.xml` parent POM. |
| **Spring Security** | Authentication & authorization framework | **Verified Existing** | `6.3.3` | Enforces RBAC, JWT validation filters, and endpoint security rules. | Remediate in-memory token blacklist and username-only session map. |
| **JJWT (Java JWT)** | JWT parsing and signature verification | **Verified Existing** | `0.11.5` | `jjwt-api`, `jjwt-impl`, `jjwt-jackson`. Generates and verifies HMAC tokens. | Verified in `pom.xml`. Plan upgrade to `0.12.x` in future maintenance. |
| **Spring Data JPA** | Object-relational mapping repository | **Verified Existing** | `3.3.4` (Hibernate 6.5) | Manages 54 entity mappings and relational queries. | Audit entities to ensure no unmanaged N+1 queries during batch sync. |
| **Springdoc OpenAPI** | REST API documentation & Swagger UI | **Verified Existing** | `2.6.0` | Automatically documents `/api/v1/*` endpoints. | Ensure synchronization endpoints are documented in OpenAPI schema. |
| **Lombok** | Boilerplate code reduction | **Verified Existing** | `1.18.34` | Generates getters, setters, builders, and loggers. | Verified in `pom.xml`. |

---

## 6. Database & Persistence Stack

| Technology | Role / Purpose | Classification | Verified Version | Compatibility & Security Notes | Pre-Implementation Verification |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PostgreSQL** | Authoritative System of Record (SoR) | **Verified Existing** | `17` | Relational database hosting 54 production tables. Enforces ACID integrity. | Verified in `.github/workflows/ci.yml` (`image: postgres:17`). |
| **Flyway Core** | Database schema migration engine | **Verified Existing** | `10.17+` | Manages incremental migration scripts `V1` through `V9`. | Verified in `pom.xml`. Enforce zero-downtime expand/contract rules. |
| **Flyway PostgreSQL** | PostgreSQL-specific Flyway extension | **Verified Existing** | `10.17+` | Enables advanced PostgreSQL DDL migration features. | Verified in `pom.xml`. |
| **HikariCP** | High-performance JDBC connection pool | **Verified Existing** | `5.1.0` | Default pool in Spring Boot 3.3.4 (`maximum-pool-size: 10`). | Tune pool size for concurrent desktop background delta sync workers. |
| **H2 Database** | In-memory database for testing | **Verified Existing** | Spring Boot default | Embedded database used during local development and fast unit testing. | Scope runtime/test only; never deployed to production. |

---

## 7. Build, Packaging, and CI/CD Pipeline

| Technology | Role / Purpose | Classification | Verified / Target | Compatibility & Security Notes | Pre-Implementation Verification |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Node.js** | Frontend build environment | **Verified Existing** | `20 / 22 LTS` | Runs Vite build and TypeScript compilation. Verified in CI. | Ensure Node.js 20+ compatibility across all build scripts. |
| **Apache Maven** | Backend build and dependency manager | **Verified Existing** | `3.9+` | Compiles Spring Boot backend and executes integration test suites. | Verified in CI (`mvn clean test`, `mvn verify`). |
| **Cargo** | Rust package manager and build tool | **Proposed Desktop** | `1.80+` | Compiles native Tauri desktop binary and links C-based SQLCipher. | Test build in isolated environment without internet access. |
| **WiX Toolset** | Windows enterprise MSI installer builder | **Proposed Desktop** | `v3.14 / v4` | Produces per-machine `.msi` installers with Active Directory GPO support. | Verify WiX binary integration with Tauri CLI packaging pipeline. |
| **NSIS** | Per-user Windows installer builder | **Proposed Desktop** | `3.x` | Produces lightweight per-user `.exe` installers requiring zero admin rights. | Test installation into `%LOCALAPPDATA%\Programs\` without UAC prompt. |
| **GitHub Actions** | Automated CI/CD build matrix | **Verified Existing** | Runner `windows-2022` | Dedicated desktop workflow (`.github/workflows/desktop-ci.yml`). | Secure CI secrets (code signing certificates, API tokens) in environments. |

---

## 8. Incompatible or Unverified Version Combinations & Pre-Implementation Checklist

```
+-----------------------------------------------------------------------------------+
|               PRE-IMPLEMENTATION COMPATIBILITY & VERIFICATION MATRIX              |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|  [ ISSUE 1: JJWT 0.11.5 vs JJWT 0.12.x ]                                         |
|  - Finding: pom.xml declares JJWT 0.11.5 (earlier docs conjectured 0.12.6).       |
|  - Impact: JJWT 0.11.5 uses Jwts.parserBuilder() rather than Jwts.parser().      |
|  - Action: Retain 0.11.5 during initial desktop integration to prevent breaking  |
|    existing backend auth services; schedule upgrade in Phase 0.                  |
|                                                                                   |
|  [ ISSUE 2: React 19.2.8 & Vite 8.3 Asset Paths in WebView2 ]                     |
|  - Finding: Default Vite config generates absolute paths (/assets/index.js).     |
|  - Impact: In custom webview origins (tauri.localhost), absolute paths may fail. |
|  - Action: Configure base: './' or explicit origin baseURL in vite.config.ts.    |
|                                                                                   |
|  [ ISSUE 3: SQLCipher MSVC Linker Collisions on Windows ]                         |
|  - Finding: rusqlite with bundled-sqlcipher requires OpenSSL or bundled crypto.   |
|  - Impact: Can trigger C runtime linker collisions (MSVCRT vs LIBCMT).           |
|  - Action: Validate cargo build --features bundled-sqlcipher on clean Windows.   |
|                                                                                   |
|  [ ISSUE 4: WiX MSI Property Ingestion ]                                          |
|  - Finding: Passing BACKEND_URL via msiexec does not automatically configure app.|
|  - Action: Implement WiX registry write custom action to HKLM\SOFTWARE\Policies  |
|    or an application configuration file bridge.                                   |
+-----------------------------------------------------------------------------------+
```
