# EduPulse ERP — Desktop System Architecture Design
## Work Package 02 — Target Architecture Specification for Windows Desktop Edition (Tauri v2)

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 02)  
**Target Platform:** Windows 10 / Windows 11 Enterprise (x64 / ARM64)  
**Recommended Framework:** Tauri v2 (with Microsoft Edge WebView2)  
**Author:** Antigravity AI Engineering Assistant (Pair Programming Baseline)  
**Date:** 2 November 2026  
**Document Status:** Authoritative Architectural Design Document  

---

## 1. Executive Summary & Guiding Principles

### 1.1 Context and Vision
EduPulse ERP is evolving from a browser-hosted Single Page Application (SPA) into a robust, high-performance, enterprise-grade Windows desktop application. The product contains **259 planned modules across 23 institutional domains**, serving higher education institutions ranging from single-campus colleges to large multi-tenant university systems.

This document establishes the definitive target architecture for the EduPulse Windows Desktop Edition using **Tauri v2** as the native desktop shell.

```
+----------------------------------------------------------------------------------------------------+
|                                    CORE ARCHITECTURAL PRINCIPLES                                   |
+----------------------------------------------------------------------------------------------------+
| 1. Complete Scope Preservation  | All 259 planned modules and 23 domains remain intact.            |
| 2. Maximum Frontend Reuse       | 100% reuse of frontend-v2/ (React 19.2 + Vite 8.3 + TS).           |
| 3. Single Business Authority    | Spring Boot 3.3.4 remains the sole business logic engine.        |
| 4. System of Record             | PostgreSQL 15+ remains the sole transactional source of truth.   |
| 5. Zero DB Credential Exposure  | Desktop clients never hold direct database credentials or ports. |
| 6. Zero Rust Logic Duplication  | Rust handles OS interop, security, and caching; zero ERP rules.  |
| 7. Secure Credential Boundary   | Refresh tokens stored encrypted via Windows DPAPI, not web store.|
| 8. Dynamic Server Configuration | Campus server endpoint is dynamically configurable by admins.   |
| 9. Read-Only Offline Initial    | Initial offline capabilities are strictly read-only and cached.  |
| 10. Enterprise IT Native        | Native MSI and GPO deployment support for institutional sysadmins.|
+----------------------------------------------------------------------------------------------------+
```

---

## 2. High-Level System Architecture

The EduPulse Desktop Application operates as a multi-tier, decoupled architecture consisting of an isolated client-side presentation layer, a native operating system integration layer, an encrypted OS credential vault, and a centralized enterprise backend.

### 2.1 Complete Architectural Topology Diagram

```mermaid
graph TD
    subgraph ClientHost ["Client Workstation (Windows 10 / 11 Workstation)"]
        subgraph TauriShell ["Tauri v2 Native Desktop Container (edupulse.exe)"]
            subgraph WebviewZone ["Isolated Presentation Layer (Microsoft Edge WebView2)"]
                ReactSPA["React 19.2 SPA (frontend-v2)"]
                ReactRouter["React Router v7 (Layout & Workspaces)"]
                QueryCache["TanStack Query v5 (Data Cache)"]
                ZustandMem["Zustand Store (In-Memory Access Token)"]
                AxiosClient["Axios HTTP Client (Dynamic BaseURL)"]
                ReactSPA --> ReactRouter
                ReactRouter --> QueryCache
                QueryCache --> AxiosClient
                ZustandMem --> AxiosClient
            end

            subgraph NativeCore ["Trusted Native Layer (Rust Core / src-tauri)"]
                TAOEngine["TAO Window & Event Engine"]
                WRYEngine["WRY Webview Controller"]
                CapEngine["Capability & Permissions Engine"]
                ConfigMgr["Configuration Manager (config.json)"]
                VaultBridge["DPAPI Cryptographic Vault Bridge"]
                OfflineEngine["Encrypted Offline Cache (SQLite / rusqlite)"]
                UpdateMgr["Cryptographic Updater (Ed25519)"]
                
                TAOEngine --- WRYEngine
                WRYEngine <-->|"Custom Scheme / IPC IPC Router"| CapEngine
                CapEngine --> ConfigMgr
                CapEngine --> VaultBridge
                CapEngine --> OfflineEngine
                CapEngine --> UpdateMgr
            end
        end

        subgraph WinSubsystem ["Windows Operating System Services"]
            DPAPI["Windows DPAPI (CryptProtectData / CryptUnprotectData)"]
            WinCred["Windows Credential Manager (wincred)"]
            AppDir["Local AppData (%APPDATA%/EduPulse/)"]
            VaultBridge <--> DPAPI
            VaultBridge <--> WinCred
            ConfigMgr <--> AppDir
            OfflineEngine <--> AppDir
        end
    end

    subgraph NetworkBoundary ["Institutional Network / Cloud Boundary (TLS 1.3)"]
        HTTPSChannel["HTTPS / REST API (/api/v1/*, /auth/*)"]
        WSSChannel["Secure WebSockets (/ws/* for Notifications)"]
    end

    subgraph BackendTier ["Enterprise Backend Tier (On-Premise Server or Cloud)"]
        subgraph SpringBootApp ["EduPulse Core Application (Spring Boot 3.3.4 / Java 21)"]
            TomcatGate["Tomcat Web Gateway (Port 8081)"]
            SecurityFilter["Spring Security (JwtAuthenticationFilter)"]
            RESTControllers["39 REST Controllers (85+ Endpoints)"]
            BizServices["50 Business Services (@Transactional)"]
            JPARepos["45 Spring Data JPA Repositories"]
            TomcatGate --> SecurityFilter
            SecurityFilter --> RESTControllers
            RESTControllers --> BizServices
            BizServices --> JPARepos
        end

        subgraph DatabaseTier ["Persistence Tier (System of Record)"]
            Flyway["Flyway Migration Engine (V1 - V9)"]
            PostgreSQL[("PostgreSQL 15+ Database (54 Tables)")]
            Flyway --> PostgreSQL
            JPARepos --> PostgreSQL
        end
    end

    AxiosClient <== "Direct HTTPS (Bearer JWT)" ==> HTTPSChannel
    HTTPSChannel <==> TomcatGate
    ReactSPA -.->|"Tauri invoke() / IPC"| CapEngine
```

---

## 3. Layered Component Responsibilities & Division of Concerns

To guarantee long-term maintainability and prevent architectural rot, strict boundaries are enforced across all layers of the system:

```
+-------------------+-----------------------------------------------------------------------------------+
| System Component  | Core Architectural Responsibilities & Constraints                                 |
+-------------------+-----------------------------------------------------------------------------------+
| React Frontend    | - Renders all 259 ERP functional modules, workspaces, navigation, and charts.     |
| (frontend-v2/)    | - Manages client-side view state, form validation (Zod), and UI reactivity.       |
|                   | - Maintains short-lived Access Token in volatile JS memory (Zustand).             |
|                   | - Initiates REST API calls via Axios directly to Spring Boot backend.             |
|                   | - Interacts with Tauri native layer strictly through typed IPC commands.          |
|                   | - STRICT PROHIBITION: Zero storage of long-term credentials in localStorage.      |
+-------------------+-----------------------------------------------------------------------------------+
| Tauri Shell       | - Native OS lifecycle management: windowing, menus, system tray, crash reporting. |
| & Rust Core       | - Secure credential encryption/decryption via Windows DPAPI and Credential Vault. |
| (src-tauri/)      | - Configuration management: persistent storage of campus server URL.              |
|                   | - Network reachability probe and TLS certificate health checks.                   |
|                   | - Cryptographically verified background application auto-updates (Ed25519).       |
|                   | - Local encrypted read-only caching for offline emergency viewing.                |
|                   | - STRICT PROHIBITION: Zero duplication of Spring Boot ERP business logic.         |
+-------------------+-----------------------------------------------------------------------------------+
| Spring Boot       | - Sole authority for all institutional business rules, validations, and workflows.|
| Backend           | - Stateless JWT issuance, role-based access control (RBAC), and session auditing. |
| (backend/)        | - Database transaction management (@Transactional), entity validation, and logic. |
|                   | - Exposes 39 REST controllers and 85+ endpoints over HTTPS.                       |
|                   | - STRICT PROHIBITION: Zero awareness of client packaging (Tauri vs Web vs Mobile)|
+-------------------+-----------------------------------------------------------------------------------+
| PostgreSQL        | - Sole system of record for institutional data (students, finances, grades, etc.).|
| Database          | - Schema management and versioning strictly via Flyway migrations (V1 to V9+).    |
|                   | - Enforces relational integrity, foreign keys, unique constraints, and indexes.   |
|                   | - STRICT PROHIBITION: Never exposed directly to desktop client workstations.      |
+-------------------+-----------------------------------------------------------------------------------+
```

---

## 4. Frontend-to-Native IPC Protocol (Tauri Commands & Events)

Communication between the React presentation layer and the native Rust layer occurs via Tauri’s asynchronous, message-passing IPC bridge. All commands are strictly typed and audited against the **Tauri Capability System** (`src-tauri/capabilities/default.json`).

### 4.1 Native Command Inventory

```
+-----------------------------------+-----------------------+---------------------------------------------------+
| Command Name                      | Arguments             | Return Type / Functionality                       |
+-----------------------------------+-----------------------+---------------------------------------------------+
| `get_server_config`               | None                  | `ServerConfigResponse { url, isConfigured }`      |
| `save_server_config`              | `{ serverUrl: str }`  | `Result<ServerConfigResponse, ConfigError>`       |
| `test_server_connectivity`        | `{ serverUrl: str }`  | `Result<ConnectivityResult, NetworkError>`        |
| `secure_store_token`              | `{ refreshToken: str}`| `Result<(), SecurityError>` (DPAPI encrypt)       |
| `secure_retrieve_token`           | None                  | `Result<String, SecurityError>` (DPAPI decrypt)   |
| `secure_clear_token`              | None                  | `Result<(), SecurityError>` (Wipes DPAPI entry)   |
| `get_system_info`                 | None                  | `SystemInfo { osVersion, webviewVersion, arch }`  |
| `check_for_updates`               | None                  | `UpdateInfo { available, version, notes }`        |
| `install_update`                  | None                  | `Result<(), UpdateError>` (Triggers restart)      |
| `query_offline_cache`             | `{ entity, key }`     | `Result<JsonValue, CacheError>` (Encrypted SQLite)|
| `sync_offline_cache`              | `{ dataset: Json }`   | `Result<(), CacheError>` (Batch cache update)     |
+-----------------------------------+-----------------------+---------------------------------------------------+
```

### 4.2 Frontend Invocation Contract & Types
The desktop wrapper in `frontend-v2/src/services/desktopBridge.ts` encapsulates all Tauri commands, ensuring safe fallbacks if accessed from a standard browser:

```typescript
// Interface definitions for Tauri IPC communication
export interface ServerConfig {
  serverUrl: string;
  isConfigured: boolean;
  institutionName?: string;
}

export interface ConnectivityResult {
  reachable: boolean;
  statusCode: number;
  latencyMs: number;
  serverVersion?: string;
}

// Runtime detection
export const isDesktop = (): boolean => {
  return typeof window !== 'undefined' && '__TAURI_INTERNALS__' in window;
};

// Safe bridge execution
export const desktopBridge = {
  async getServerConfig(): Promise<ServerConfig> {
    if (!isDesktop()) return { serverUrl: '', isConfigured: false };
    const { invoke } = await import('@tauri-apps/api/core');
    return invoke<ServerConfig>('get_server_config');
  },

  async saveServerConfig(serverUrl: string): Promise<void> {
    if (!isDesktop()) return;
    const { invoke } = await import('@tauri-apps/api/core');
    return invoke('save_server_config', { serverUrl });
  },

  async testConnectivity(serverUrl: string): Promise<ConnectivityResult> {
    if (!isDesktop()) return { reachable: true, statusCode: 200, latencyMs: 0 };
    const { invoke } = await import('@tauri-apps/api/core');
    return invoke<ConnectivityResult>('test_server_connectivity', { serverUrl });
  },

  async storeRefreshToken(refreshToken: string): Promise<void> {
    if (!isDesktop()) return;
    const { invoke } = await import('@tauri-apps/api/core');
    return invoke('secure_store_token', { refreshToken });
  },

  async retrieveRefreshToken(): Promise<string | null> {
    if (!isDesktop()) return null;
    const { invoke } = await import('@tauri-apps/api/core');
    try {
      return await invoke<string>('secure_retrieve_token');
    } catch {
      return null;
    }
  },

  async clearRefreshToken(): Promise<void> {
    if (!isDesktop()) return;
    const { invoke } = await import('@tauri-apps/api/core');
    return invoke('secure_clear_token');
  }
};
```

---

## 5. Frontend-to-Backend API Communication Architecture

The desktop application communicates with the Spring Boot backend using standard HTTPS REST calls via Axios. Because the frontend runs inside Microsoft Edge WebView2, it can communicate directly over the network to the configured server endpoint without proxying through Rust for standard API transactions.

### 5.1 API Request Lifecycle Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User as Campus Staff / Student
    participant UI as React UI Component
    participant Axios as Axios Client (apiClient)
    participant Store as Zustand (Volatile Memory)
    participant Bridge as Desktop Bridge (Tauri IPC)
    participant RustVault as Rust Native DPAPI Vault
    participant Gateway as Spring Boot Tomcat (Port 8081)
    participant SecFilter as Spring Security Filter

    User->>UI: Trigger Action (e.g. Load Student Records)
    UI->>Axios: GET /api/v1/students?page=0&size=20
    
    rect rgb(240, 248, 255)
        Note over Axios,Store: Request Interceptor Phase
        Axios->>Store: Read state.token (In-Memory Access Token)
        Store-->>Axios: Return "eyJhbGciOiJIUzI1Ni..."
        Axios->>Axios: Attach Header: Authorization: Bearer <token>
    end

    Axios->>Gateway: HTTPS GET /api/v1/students
    Gateway->>SecFilter: JwtAuthenticationFilter.doFilterInternal()
    
    alt Token is Valid & Unexpired
        SecFilter->>SecFilter: Validate HMAC-SHA256 signature
        SecFilter->>SecFilter: Populate SecurityContext(Principal, Roles)
        SecFilter-->>Gateway: Proceed to StudentController
        Gateway-->>Axios: 200 OK (Paginated Student JSON)
        Axios-->>UI: Render Student Table
    else Token Expired (HTTP 401 Unauthorized)
        Gateway-->>Axios: 401 Unauthorized (Token Expired)
        
        rect rgb(255, 245, 238)
            Note over Axios,RustVault: Silent Token Refresh Flow
            Axios->>Axios: Intercept 401 & Enqueue concurrent requests
            Axios->>Bridge: retrieveRefreshToken()
            Bridge->>RustVault: invoke('secure_retrieve_token')
            RustVault-->>Bridge: Plaintext Refresh Token (Decrypted via DPAPI)
            Bridge-->>Axios: Return Refresh Token
            
            Axios->>Gateway: POST /auth/refresh { refreshToken }
            Gateway-->>Axios: 200 OK { accessToken: "new_jwt", refreshToken: "rot_jwt" }
            
            Axios->>Store: Update in-memory Access Token
            Axios->>Bridge: storeRefreshToken(rot_jwt)
            Bridge->>RustVault: invoke('secure_store_token', { rot_jwt })
            RustVault-->>Bridge: Encrypted & Persisted
            
            Axios->>Axios: Replay failedQueue with new Access Token
            Axios->>Gateway: HTTPS GET /api/v1/students (Retried)
            Gateway-->>Axios: 200 OK (Data)
            Axios-->>UI: Render Student Table
        end
    end
```

---

## 6. Configurable Institutional Backend Server URL Architecture

Unlike a public web SaaS where the frontend URL is statically hardcoded, an enterprise desktop client deployed in colleges and universities must connect to varied institutional hostnames (e.g., `https://erp.citycollege.edu:8081` or `https://cloud.edupulse.org`).

### 6.1 Server Discovery & Resolution Workflow
1. **Configuration Persistence:** The active backend server URL is persisted in the native Windows user profile:
   `%APPDATA%\com.edupulse.erp\config.json`
2. **Initial Launch Gate:** If `config.json` does not exist or `serverUrl` is blank, the desktop shell immediately renders the **Institutional Server Setup Screen** before showing the login view.
3. **Connectivity & Certificate Verification:**
   - When the user or administrator enters a campus URL, the Rust backend invokes a native HTTP probe (`reqwest` with Windows system root CA trust).
   - It performs an automated health check against `{serverUrl}/actuator/health` or `{serverUrl}/api/v1/auth/health`.
   - It verifies TLS certificate validity, latency, and expected response payloads before allowing the configuration to be saved.
4. **Active URL Injection:** Once verified, the URL is saved to `config.json` and communicated to `frontend-v2/src/config/api.ts`, dynamically setting `apiClient.defaults.baseURL`.
5. **Enterprise MDM / GPO Policy Override:**
   For mass institutional deployments via Active Directory Group Policy, the Rust configuration manager checks for a Windows Registry policy key:
   `HKLM\Software\Policies\EduPulse\ServerUrl`
   If present, the registry value takes precedence, locking the server endpoint so individual campus users cannot alter it.

---

## 7. Authentication & Secure Credential-Storage Boundaries

### 7.1 Threat Model of Token Storage
In browser web applications, storing tokens in `localStorage` exposes them to any malicious JavaScript executed via XSS or browser extensions. In a Windows desktop application, credentials must be safeguarded against both process-level memory inspection and local disk exfiltration:

```
+-----------------------------------+-----------------------------------+-----------------------------------+
| Security Dimension                | Access Token (15-Minute Expiry)   | Refresh Token (7-Day Expiry)      |
+-----------------------------------+-----------------------------------+-----------------------------------+
| Storage Location                  | Volatile JavaScript Memory        | Windows DPAPI / Credential Vault  |
| Storage Mechanism                 | Zustand In-Memory State           | Win32 CryptProtectData            |
| Persistence Across App Restarts   | NO (Purged when process exits)    | YES (Persists in encrypted form)  |
| Plaintext Disk Exposure           | NEVER written to disk             | NEVER written in plaintext        |
| Protection Scope                  | Process memory isolation          | Logged-in Windows User Account    |
| Mitigation Against Disk Theft     | 100% immune (non-persistent)      | 100% immune (unusable off-machine)|
+-----------------------------------+-----------------------------------+-----------------------------------+
```

### 7.2 Complete Authentication Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User as Campus Staff / Student
    participant UI as React Login Page
    participant Bridge as Desktop Bridge (IPC)
    participant Rust as Rust Native Core
    participant DPAPI as Windows DPAPI Vault
    participant Server as Spring Boot Auth Controller

    User->>UI: Enter Username & Password
    UI->>Server: POST /auth/login { username, password }
    
    alt Credentials Valid
        Server->>Server: BCrypt password verification
        Server->>Server: Generate JWT Access Token (15m) + Refresh Token (7d)
        Server-->>UI: 200 OK { token, refreshToken, user: { role, permissions } }
        
        UI->>UI: Set Access Token in Zustand (Volatile Memory)
        UI->>Bridge: storeRefreshToken(refreshToken)
        Bridge->>Rust: invoke('secure_store_token', { refreshToken })
        Rust->>DPAPI: CryptProtectData(refreshToken, Entropy)
        DPAPI-->>Rust: Encrypted Ciphertext Blob
        Rust->>Rust: Write ciphertext to %APPDATA%/EduPulse/vault.dat
        Rust-->>Bridge: Storage OK
        Bridge-->>UI: Storage OK
        
        UI->>UI: Navigate to /dashboard (Role Workspace)
    else Credentials Invalid
        Server-->>UI: 401 Unauthorized { message: "Invalid credentials" }
        UI->>User: Display Error Banner
    end

    Note over UI,Server: User explicitly clicks "Logout"
    User->>UI: Click Logout
    UI->>Server: POST /auth/logout (Revoke active session)
    UI->>UI: Wipe Zustand Memory (token = null, user = null)
    UI->>Bridge: clearRefreshToken()
    Bridge->>Rust: invoke('secure_clear_token')
    Rust->>Rust: Delete vault.dat / CryptUpdate(0)
    Rust-->>Bridge: Cleared
    UI->>UI: Navigate to /login
```

---

## 8. Application Startup and Initialization Flow

The desktop initialization sequence must handle initial configuration, credential restoration, and network validation before rendering the user interface:

### 8.1 Application Startup Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    actor User as User Launches Desktop Shortcut
    participant OS as Windows OS (Process Launch)
    participant Rust as Tauri Core (edupulse.exe)
    participant Config as config.json / GPO Registry
    participant DPAPI as Windows DPAPI Vault
    participant WV as WebView2 Renderer
    participant React as React 19 Frontend Mount
    participant Server as Institutional Server

    User->>OS: Execute edupulse.exe
    OS->>Rust: Initialize TAO Event Loop & WRY Window
    
    Rust->>Config: Read Server Configuration
    alt Server URL Not Configured
        Rust->>WV: Load index.html
        WV->>React: Mount Application
        React->>React: Detect isConfigured == false
        React->>User: Render "Institutional Server Setup Wizard"
    else Server URL Configured
        Rust->>Server: Fast Ping /actuator/health (Timeout: 2000ms)
        alt Server Reachable
            Rust->>WV: Load index.html (Inject active Server URL)
            WV->>React: Mount Application & Initialize Axios
            
            React->>Rust: invoke('secure_retrieve_token')
            Rust->>DPAPI: CryptUnprotectData(vault.dat)
            alt Valid Refresh Token Found
                DPAPI-->>Rust: Plaintext Refresh Token
                Rust-->>React: Return Refresh Token
                React->>Server: POST /auth/refresh { refreshToken }
                alt Refresh Successful
                    Server-->>React: 200 OK { accessToken, newRefreshToken }
                    React->>React: Store accessToken in Zustand
                    React->>Rust: invoke('secure_store_token', { newRefreshToken })
                    React->>React: Navigate directly to /dashboard
                else Refresh Failed / Token Revoked
                    React->>Rust: invoke('secure_clear_token')
                    React->>React: Navigate to /login
                end
            else No Refresh Token Found
                Rust-->>React: Return null
                React->>React: Navigate to /login
            end
        else Server Unreachable
            Rust->>WV: Load index.html (Offline/Unreachable state)
            WV->>React: Mount Application
            React->>React: Detect Server Unreachable
            React->>User: Render "Server Offline / Reconnect Banner" with Cached Mode option
        end
    end
```

---

## 9. Network Failure, Disconnection, and Recovery Architecture

In higher education environments, local campus Wi-Fi fluctuations and power outages are common. The desktop application implements comprehensive resilience mechanisms to handle connectivity drops gracefully:

### 9.1 Network Failure and Recovery Flow Diagram

```mermaid
flowchart TD
    StartRequest["Axios Outgoing API Request"] --> CheckNet{"Network Link Active?"}
    
    CheckNet -- Yes --> ExecReq["Execute HTTPS Call"]
    CheckNet -- No (navigator.onLine == false) --> TriggerOffline["Trigger Immediate Offline Handling"]
    
    ExecReq --> CheckResp{"Response Status"}
    
    CheckResp -- "200 - 299 OK" --> Success["Return Response to React Query"]
    
    CheckResp -- "401 Unauthorized" --> TriggerAuthRefresh["Trigger Dual-Token Refresh Flow"]
    
    CheckResp -- "502 / 503 / 504 / ECONNREFUSED" --> ServerDown["Server Outage / Gateway Failure"]
    CheckResp -- "Timeout (ECONNABORTED > 15s)" --> TimeoutFailure["Request Timeout"]
    
    ServerDown --> CheckRetry{"Retry Count < 3?"}
    TimeoutFailure --> CheckRetry
    TriggerOffline --> OfflineBanner["Display Persistent Reconnection Banner"]
    
    CheckRetry -- Yes --> BackoffDelay["Exponential Backoff (1s, 2s, 4s)"]
    BackoffDelay --> ExecReq
    
    CheckRetry -- No (Max Retries Exhausted) --> ShowErrorUI["Surface Non-Blocking Error Toast"]
    ShowErrorUI --> CheckCache{"Cached Offline Data Available?"}
    
    CheckCache -- Yes --> RenderCached["Render Cached View (Read-Only Mode)"]
    CheckCache -- No --> RenderEmpty["Render Friendly Offline Recovery View"]
    
    OfflineBanner --> BackgroundProbe["Background Polling Ping (/actuator/health every 10s)"]
    BackgroundProbe --> ProbeCheck{"Ping Successful?"}
    
    ProbeCheck -- No --> BackgroundProbe
    ProbeCheck -- Yes --> OnlineRestore["Dismiss Banner & Invalidate TanStack Queries"]
    OnlineRestore --> AutoSync["Silent Data Re-fetch & UI Refresh"]
```

---

## 10. Update and Installer Architecture

Desktop deployments in educational institutions require two distinct distribution channels: individual user installations and mass enterprise administrative installations.

```
+-----------------------------------+-----------------------------------+-----------------------------------+
| Distribution Dimension            | NSIS Interactive Installer (.exe) | WiX Enterprise Installer (.msi)   |
+-----------------------------------+-----------------------------------+-----------------------------------+
| Primary Target                    | Individual faculty & student PCs  | Institutional IT Administrators   |
| Deployment Mechanism              | Manual download & execution       | Active Directory GPO / Intune     |
| Administrative Privileges         | Per-User (no UAC) or Machine      | Per-Machine (Silent /qn flag)     |
| Auto-Update Channel               | Automated background updater      | Managed by Campus IT or Updater   |
| Cryptographic Verification        | Ed25519 signature check           | Authenticode & Ed25519 signature  |
| Payload Size                      | ~12 MB                            | ~14 MB                            |
+-----------------------------------+-----------------------------------+-----------------------------------+
```

### 10.1 Cryptographically Enforced Auto-Update Flow
1. **Signature Verification:** The Tauri core utilizes `tauri-plugin-updater`. Every update artifact (`.zip` / `.tar.gz`) is signed during CI/CD using an **Ed25519 private key**. The corresponding public key is hardcoded into `src-tauri/tauri.conf.json`.
2. **Update Manifest:** The desktop client periodically queries an update manifest JSON file hosted on a campus mirror or secure cloud storage:
   ```json
   {
     "version": "1.1.0",
     "notes": "EduPulse ERP 1.1.0: Enhanced fee reconciliation and examination seating modules.",
     "pub_date": "2026-11-15T10:00:00Z",
     "platforms": {
       "windows-x86_64": {
         "signature": "dW50cnVzdGVkIGNvbW1lbnQ6IHRhdXJpIHNpZ25hdHVyZQpSV1F5...",
         "url": "https://updates.edupulse.org/releases/EduPulse_1.1.0_x64.msi.zip"
       }
     }
   }
   ```
3. **Execution & Silent Restart:** If a new signature-verified version is detected, the client downloads the payload in the background, unpacks it to a staging directory, and prompts the user or silently replaces the binary upon application close.

---

## 11. Deployment Topologies: Cloud-Hosted vs. Institution-Hosted

EduPulse ERP supports two primary enterprise deployment topologies without requiring separate desktop application builds:

### 11.1 Deployment Topology Comparison Diagram

```mermaid
graph TD
    subgraph ClientFleet ["Distributed Desktop Fleet (Windows 10 / 11)"]
        PC1["Admin Workstation (Admissions)"]
        PC2["Faculty Laptop (Grading)"]
        PC3["Finance Office (Fee Collection)"]
        PC4["Library Terminal (Catalog)"]
    end

    subgraph TopologyCloud ["Topology A: Cloud-Hosted SaaS Model"]
        subgraph CloudProvider ["Cloud Infrastructure (AWS / Azure / Render)"]
            CloudLB["Cloud Load Balancer (HTTPS:443)"]
            CloudSpring["Spring Boot Cluster (Auto-scaled)"]
            CloudPostgres[("Managed PostgreSQL (Neon / Aurora)")]
            CloudLB --> CloudSpring
            CloudSpring --> CloudPostgres
        end
    end

    subgraph TopologyOnPrem ["Topology B: Institution-Hosted On-Premise LAN"]
        subgraph CampusDC ["Campus Data Center / Server Room"]
            CampusFirewall["Institutional Firewall / Gateway"]
            CampusSpring["Institutional Spring Boot Server (Port 8081)"]
            CampusPostgres[("Dedicated PostgreSQL 15+ Server")]
            CampusFirewall --> CampusSpring
            CampusSpring --> CampusPostgres
        end
    end

    PC1 -. "Configured to Cloud URL" .-> CloudLB
    PC2 -. "Configured to Cloud URL" .-> CloudLB
    PC3 == "Configured to LAN IP: 192.168.1.50" ==> CampusFirewall
    PC4 == "Configured to LAN IP: 192.168.1.50" ==> CampusFirewall
```

### 11.2 Architectural Differentiation
- **Topology A (Cloud-Hosted SaaS):**
  - Desktop client connects over public WAN via HTTPS (`https://cloud.edupulse.org`).
  - Managed SSL/TLS termination, automated database backups, multi-AZ high availability.
  - Ideal for institutions lacking dedicated on-premise IT infrastructure.
- **Topology B (Institution-Hosted On-Premise LAN):**
  - Desktop client connects over campus Gigabit LAN (`https://erp-server.campus.local:8081`).
  - Data remains strictly within institutional physical boundaries (critical for government and strict data-sovereignty mandates).
  - High throughput for bulk student document uploads and instantaneous report generation.

---

## 12. Initial Offline Read-Only Architecture

### 12.1 Design Constraints and Safety Rules
In higher education ERP systems, offline write transactions (such as fee collections, student enrollment, or grade submission) present severe data collision and financial reconciliation risks. Therefore, the initial desktop release enforces strict architectural boundaries:
- **Strictly Read-Only:** The offline mode allows viewing previously cached institutional data only. All write, update, and delete actions are disabled in the UI when offline.
- **Zero Local Write Queues (Phase 1):** No offline mutation queueing is permitted until Phase 2 bidirectional conflict-resolution protocols are formally designed and audited.

### 12.2 Local Caching Engine & Data Flow
1. **Engine:** Lightweight SQLite database embedded via `tauri-plugin-sql` or direct Rust `rusqlite` with table-level encryption (`SQLCipher` or DPAPI key derivation).
2. **Cache Target Datasets:**
   - Active user profile, roles, and granular permissions.
   - Academic calendar, semester timetables, and lecture schedules.
   - Student directories and course registration catalogs.
   - Historical fee receipts (PDF/JSON representations) for verification.
3. **Cache-on-Read Synchronization:**
   Whenever the user navigates through the ERP while online, successful TanStack Query responses for whitelisted cacheable endpoints are asynchronously forwarded to the native cache:
   ```
   React Query Success -> desktopBridge.syncOfflineCache(key, data) -> SQLite INSERT OR REPLACE
   ```
4. **Offline Read-Through:**
   If a network request fails and the application is offline, the React data hook falls back to `desktopBridge.queryOfflineCache(key)`, rendering the view with an explicit **"Cached Read-Only Data"** indicator and timestamp.

---

## 13. Trust Boundaries, Security Controls & Threat Mitigation

### 13.1 Trust Boundary Mapping
The architecture enforces five discrete security zones with strict isolation between each boundary:

```
[Zone 1: External Network]
          │
     (TLS 1.3 HTTPS)
          ▼
[Zone 2: Sandboxed Webview]  ◄──(Tauri Capabilities)──►  [Zone 3: Trusted Rust Core]
(React DOM, JS Heap Memory)                                  │
                                                    (Win32 CryptProtectData)
                                                             ▼
                                                [Zone 4: OS Security Subsystem]
                                                (DPAPI, Credential Vault, Files)
```

```
+----------------------------------------------------------------------------------------------------+
|                                    SECURITY CONTROLS MATRIX                                        |
+------------------------------------+---------------------------------------------------------------+
| Threat / Vulnerability             | Implemented Architectural Security Control                    |
+------------------------------------+---------------------------------------------------------------+
| Cross-Site Scripting (XSS) in UI   | Node.js is completely absent from renderer; strict Content   |
|                                    | Security Policy (CSP); Tauri capability scoping blocks        |
|                                    | unauthorized native system command execution.                 |
+------------------------------------+---------------------------------------------------------------+
| Local Token Theft / Inspection     | Refresh tokens are encrypted via Windows DPAPI, binding the   |
|                                    | secret to the user's Windows login credentials.               |
+------------------------------------+---------------------------------------------------------------+
| Man-in-the-Middle (MitM) Attacks   | Enforced HTTPS / TLS 1.3; validation of server certificates;  |
|                                    | option for institutional CA certificate bundle pinning.       |
+------------------------------------+---------------------------------------------------------------+
| Malicious Update Injection         | All auto-update binaries require cryptographic Ed25519       |
|                                    | signature verification prior to staging or execution.         |
+------------------------------------+---------------------------------------------------------------+
| Direct Database Tampering          | PostgreSQL port (5432) is never exposed to desktop clients;   |
|                                    | all DB interactions occur solely through Spring Boot JPA.     |
+------------------------------------+---------------------------------------------------------------+
| Privilege Escalation via IPC       | Granular JSON capability permissions define exact command     |
|                                    | names, arguments, and window scopes permitted to invoke Rust. |
+------------------------------------+---------------------------------------------------------------+
```

---

## 14. Architectural Trade-offs & Unresolved Decisions

### 14.1 Key Architecture Trade-offs

| Decision | Selected Option | Alternative Considered | Architectural Justification |
| :--- | :--- | :--- | :--- |
| **Native Container** | **Tauri v2** | Electron v33+ | Prevents severe RAM bloat on 4GB campus workstations; leverages OS-managed WebView2 Evergreen patching. |
| **Credential Storage** | **Windows DPAPI** | Tauri Stronghold | DPAPI does not require asking users for a master passphrase every time the application opens. |
| **Server URL Storage** | **App Config JSON** | Build-time env vars | Enables a single generic installer to be distributed across multiple distinct college campuses. |
| **Initial Offline Scope**| **Strictly Read-Only** | Read/Write Offline Sync | Prevents split-brain financial reconciliation conflicts and complex distributed state collisions. |
| **Offline DB Engine** | **SQLite (rusqlite)** | Browser IndexedDB | Provides robust file-level encryption and longevity independent of browser cache clearing policies. |

### 14.2 Unresolved Decisions (Scheduled for Day 02 WP03–WP06)
1. **Decision 1 (WP03 - Credential Architecture):** Evaluate whether to use `keyring-rs` (Windows Credential Manager) or direct Win32 `CryptProtectData` (DPAPI) for the primary token persistence implementation.
2. **Decision 2 (WP04 - Backend Connectivity):** Establish the exact JSON schema and registry override location for the institutional server URL configuration.
3. **Decision 3 (WP05 - CORS & API Compatibility):** Verify whether Tauri's custom origin (`https://tauri.localhost`) requires updating Spring Boot's `CorsConfigurationSource` in `SecurityConfig.java`.
4. **Decision 4 (WP06 - Offline Architecture):** Define the exact list of the initial 10 high-priority read-only datasets for local SQLite caching.

---

## 15. Conclusion & Next Steps

This document establishes the authoritative system architecture for EduPulse ERP Windows Desktop Edition. It preserves all 259 existing modules, ensures zero frontend rewrite, retains Spring Boot and PostgreSQL as the sole backend authority, and enforces operating-system-grade security.

Proceed immediately to **Work Package 03: Credential Storage & Secure Authentication Design**.
