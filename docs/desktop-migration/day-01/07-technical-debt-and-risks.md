# Work Package 05B — Technical Debt & Desktop Migration Risk Register

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 01)  
**Scope:** Repository-wide Technical Debt, Architectural Flaws & Desktop Blockers  
**Date of Audit:** 1 November 2026  
**Auditor:** Antigravity AI Engineering Assistant (Pair Programming Baseline)

---

## 1. Executive Summary

This register consolidates all identified technical debt, legacy anomalies, architectural vulnerabilities, and desktop-specific blockers across the EduPulse codebase.

Each item is categorized by domain, assigned a severity level (**Critical**, **High**, **Medium**, **Low**), assessed for its direct impact on desktop migration, and provided with an actionable remediation strategy.

---

## 2. Technical Debt & Risk Matrix

| Risk ID | Category | Severity | Description | Desktop Migration Impact | Remediation Strategy |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RISK-01** | CI/CD | **Critical** | `ci.yml` points `working-directory` to abandoned `frontend/` instead of `frontend-v2/` | Prevents automated CI validation of frontend PRs | Update CI workflow to target `frontend-v2/` |
| **RISK-02** | Security | **High** | Auth tokens stored in browser `localStorage` (`auth-storage`) | In desktop app, tokens stored unencrypted in `%APPDATA%` SQLite files | Implement native secure store via OS Credential Manager |
| **RISK-03** | Architecture | **High** | Hardcoded API URL fallback (`/api` in production) in `src/config/api.ts` | In desktop app, relative `/api` resolves to `tauri://localhost/api` or `file:///api` (404/Network Error) | Introduce application server settings dialog allowing custom institution endpoint configuration |
| **RISK-04** | Desktop UX | **Medium** | Use of `window.location.href = ...` and `window.location.reload()` | Causes webview flashes, reloads, or breaks router state | Replace browser location navigation with React Router's `useNavigate` |
| **RISK-05** | Storage / Blob | **Medium** | `window.URL.createObjectURL(blob)` for document previews | Webview Content Security Policies (CSP) may block `blob:` protocol | Whitelist `blob:` in desktop CSP or use native OS file save dialogs |
| **RISK-06** | CI / Maven | **Medium** | `mvn verify -P postgres-it` references non-existent profile in `pom.xml` | PostgreSQL ITs are not executed in CI as intended | Define `postgres-it` profile in `pom.xml` or invoke test pattern directly |
| **RISK-07** | Network | **High** | 100% online backend dependency with no offline write queue | App becomes unusable during campus network drops or WiFi blackouts | Implement offline queue for attendance and read-only local cache |
| **RISK-08** | Repository | **Low** | Triplicate frontend directories (`frontend/`, `frontend-v2/`, root `src/`) | Potential developer confusion regarding authoritative UI source | Mark legacy directories as deprecated in documentation |
| **RISK-09** | Windows Native | **Medium** | Browser-based toast notifications (`sonner`) instead of Windows Action Center | Alerts disappear when application is minimized or behind other windows | Integrate native OS notifications via desktop runtime API |
| **RISK-10** | Unfinished Code| **Medium** | Four deferred offline sync files in `com.erp.devjson` | Incomplete feature state in working tree | Preserve files as requested; complete or isolate during sync sprint |

---

## 3. Deep-Dive Analysis of Critical Risks

### 3.1 Hardcoded API Base URL & Desktop Routing (RISK-03)
In `frontend-v2/src/config/api.ts`:
```typescript
export const BASE_URL = envBaseUrl || (import.meta.env.DEV ? 'http://localhost:8081' : '/api');
```
In a web browser hosted on Vercel, Nginx or Vercel proxies `/api/*` to the remote backend.  
In a packaged Windows desktop application (Tauri or Electron), the UI is served from a local bundle (`tauri://localhost` or `file:///`). Calling `/api` will result in an immediate failure (`net::ERR_CONNECTION_REFUSED` or `404`).

**Required Desktop Architecture:**
The desktop application must include an **Institutional Server Connection Setting**:
1. When launched for the first time, if no backend URL is configured, present an "Institution Server Setup" dialog.
2. The user or administrator inputs their campus server URL (e.g., `https://erp.campus.edu` or `http://192.168.1.100:8081`).
3. Store this configuration in local application settings.
4. Pass this configured URL to `BASE_URL`.

### 3.2 Insecure Token Storage in Desktop Runtimes (RISK-02)
Zustand's `persist` middleware writes to `localStorage`:
```typescript
persist(..., { name: 'auth-storage' })
```
On Windows:
- WebView2 (Tauri) stores `localStorage` in `%LOCALAPPDATA%\<app-id>\EBWebView\Default\Local Storage\leveldb\`.
- Electron stores `localStorage` in `%APPDATA%\<app-id>\Local Storage\leveldb\`.
Both are unencrypted plaintext on disk. Any standard user process running under the same Windows account can read these files and extract active JWT access and refresh tokens.

**Required Desktop Architecture:**
- Implement a desktop IPC bridge for token storage.
- Store JWT tokens using **Windows Data Protection API (DPAPI)** or **Windows Credential Manager**.
- Tauri: Use `tauri-plugin-stronghold` or native Rust keyring integration.
- Electron: Use `safeStorage.encryptString()` and `safeStorage.decryptString()`.

### 3.3 File System & Document Handling (RISK-05)
The existing web application uses standard `<input type="file">` and `FormData` for student photos and document attachments, and creates object URLs for PDF previews:
```typescript
const blobUrl = window.URL.createObjectURL(blob);
```
In desktop environments:
- Embedded PDFs inside `<iframe src="blob:...">` may fail if the webview disables plugins or blocks object URLs under strict CSP.
- The desktop ERP should leverage native Windows file dialogs (`IFileDialog` via Tauri/Electron APIs) for saving fee receipts, admit cards, and student records directly to the user's `Downloads` or `Documents` folder.

### 3.4 Offline & Low-Connectivity Institutional Reality (RISK-07)
Institutions frequently suffer from intermittent campus network disruptions. In the current web application, loss of internet connectivity results in immediate `AxiosError: Network Error` and complete application failure.
- Desktop Phase 1: Graceful offline detection with banner ("Offline Mode: View-Only").
- Desktop Phase 2: Local SQLite cache for student rosters, timetable schedules, and fee structures.
- Desktop Phase 3: Outbox queue for offline attendance recording that automatically syncs when connectivity is restored.
