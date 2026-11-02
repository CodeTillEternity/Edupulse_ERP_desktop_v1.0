# EduPulse ERP — Unified Web & Desktop API Compatibility
## Work Package 05C — Shared REST API Contracts, Payload Standards & Client Parity

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 02)
**Target Platform:** Windows 10 / Windows 11 Enterprise (Tauri v2 + Spring Boot 3.3.4)
**Author:** Antigravity AI Engineering Assistant (Architecture Baseline)
**Date:** 2 November 2026
**Document Status:** Approved Engineering Architecture Specification

---

## 1. Executive Summary & Core Principle

A fundamental architectural mandate of the EduPulse desktop migration is **complete API parity**. The Spring Boot backend must serve both the browser-hosted web client (`frontend-v2/` hosted on an institutional web portal) and the native Windows desktop client (`edupulse.exe` running via Tauri v2) without duplicating business logic, maintaining divergent endpoint versions, or branching DTO schemas.

This document defines the unified API contract governing network transport, serialization, error schemas, pagination, file transfers, and header semantics across all 259 planned ERP modules.

```
+----------------------------------------------------------------------------------------------------+
|                                    SHARED API CONTRACT SUMMARY                                     |
+------------------------------------+---------------------------------------------------------------+
| Protocol & Transport               | HTTPS / TLS 1.3 (JSON REST API)                               |
| Base URL Resolution                | Dynamic Injection (Desktop: config.json / Web: Origin)         |
| Authentication Header              | `Authorization: Bearer <JWT_ACCESS_TOKEN>`                     |
| Error Response Specification       | `ErrorResponse` (RFC 7807-compliant structured JSON)          |
| API Versioning Topology            | Dual Base: `/auth/*` (Auth/Session) & `/api/v1/*` (Domain ERP)|
| Pagination & Sorting               | Standard Spring Data JPA `Pageable` (`page`, `size`, `sort`)  |
| File Upload / Download             | `multipart/form-data` (10MB limit) & Octet Stream with Header  |
+------------------------------------+---------------------------------------------------------------+
```

---

## 2. API Base URL Management Across Clients

```
+-----------------------------------+-----------------------------------+-----------------------------------+
| Architectural Dimension           | Web Application Client            | Tauri Windows Desktop Client      |
+-----------------------------------+-----------------------------------+-----------------------------------+
| Resolution Mechanism              | Browser origin or `VITE_API_URL`  | Dynamic Rust Config (`config.json`)|
| Storage Medium                    | Build-time environment variable   | Local `%APPDATA%` file or GPO     |
| Client-Side Injection             | Direct Axios instantiation        | Dynamic `apiClient.defaults.baseURL|
| Relative Path Behavior (`/api`)   | Proxied by Nginx / Web Server     | FORBIDDEN (Must be absolute FQDN) |
| Protocol Scheme                   | Matches hosting website (HTTPS)   | Enforced HTTPS (Port configurable)|
+-----------------------------------+-----------------------------------+-----------------------------------+
```

### Unified Client Resolution in `frontend-v2/src/config/api.ts`:
```typescript
import axios from 'axios';
import { desktopBridge, isDesktop } from '../services/desktopBridge';

// Initial resolution with desktop awareness
let resolvedBaseUrl = import.meta.env.VITE_API_BASE_URL || (import.meta.env.DEV ? 'http://localhost:8081' : '/api');

export const apiClient = axios.create({
  baseURL: resolvedBaseUrl,
  timeout: 15000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Dynamic desktop bootstrap hook (called during React root initialization)
export const initializeApiEndpoint = async (): Promise<string> => {
  if (isDesktop()) {
    const config = await desktopBridge.getServerConfig();
    if (config.isConfigured && config.serverUrl) {
      resolvedBaseUrl = config.serverUrl.replace(/\/$/, '');
      apiClient.defaults.baseURL = resolvedBaseUrl;
    }
  }
  return resolvedBaseUrl;
};
```

---

## 3. Standardized Error Response Conventions

Both web and desktop clients consume the exact same structured error response format emitted by `backend/src/main/java/com/erp/common/exception/GlobalExceptionHandler.java`:

### 3.1 Standard Error Payload Schema
```json
{
  "statusCode": 400,
  "error": "Validation Failed",
  "message": "Input payload contains validation errors",
  "path": "/api/v1/students",
  "errors": {
    "email": "Invalid institutional email domain",
    "dateOfBirth": "Student must be at least 16 years of age"
  }
}
```

### 3.2 HTTP Status Code Semantics

```
+-------------+-------------------------+--------------------------------------------------------------------+
| Status Code | Standard Reason Phrase  | Semantic Trigger in EduPulse ERP                                   |
+-------------+-------------------------+--------------------------------------------------------------------+
| 200 OK      | Success                 | Standard successful query (GET), update (PUT), or delete.         |
| 201 Created | Created                 | New entity persisted (e.g. student admitted, fee head created).    |
| 204 No Content | No Content           | Successful action returning no body (e.g. preflight OPTIONS).      |
| 400 Bad Req | Bad Request             | DTO validation error (`MethodArgumentNotValidException`).         |
| 401 Unauth  | Unauthorized            | JWT expired or invalid (`JwtAuthenticationEntryPoint`).            |
| 403 Forbid  | Forbidden               | Role/permission check failed (`CustomAccessDeniedHandler`).        |
| 404 Not Fnd | Not Found               | Entity does not exist (`ResourceNotFoundException`).               |
| 409 Conflict| Conflict                | Duplicate key, concurrent update, or business constraint conflict. |
| 413 Payload | Payload Too Large       | Uploaded photo/document exceeds 10MB limit.                        |
| 500 Server  | Internal Server Error   | Unhandled runtime exception.                                       |
| 503 Unavail | Service Unavailable     | Server is running but database/storage volume is offline.          |
+-------------+-------------------------+--------------------------------------------------------------------+
```

---

## 4. Pagination, Sorting & Filtering Specifications

To maintain fluid UI rendering across data-heavy academic tables (such as 1,000+ student rosters, financial fee registries, and library catalogs), both Web and Desktop clients adhere to Spring Data JPA `Pageable` standards:

### 4.1 Request Query Parameters:
```http
GET /api/v1/students?page=0&size=25&sort=admissionNumber,desc&departmentId=5&status=ACTIVE
```
- `page`: Zero-indexed page integer (default: `0`).
- `size`: Items per page (default: `20`, max: `100`).
- `sort`: Comma-separated property name and direction (`asc` or `desc`).
- Domain Filters: Normalized query parameters (`departmentId`, `academicYearId`, `status`).

### 4.2 Paginated Response JSON Structure:
```json
{
  "content": [
    { "id": 101, "admissionNumber": "2026-CS-001", "fullName": "Alice Smith" }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 25,
    "sort": { "sorted": true, "empty": false }
  },
  "totalElements": 482,
  "totalPages": 20,
  "last": false,
  "first": true,
  "size": 25,
  "number": 0,
  "numberOfElements": 25,
  "empty": false
}
```

*Frontend Consumption:* TanStack Query (`useQuery`) utilizes `keepPreviousData: true` to prevent layout shift during pagination transitions in both web and desktop environments.

---

## 5. File Uploads & Binary Document Downloads

### 5.1 File Uploads (`multipart/form-data`)
- **Use Cases:** Student ID photo capture, identity document uploads, bulk student CSV imports.
- **Payload Format:** Standard `multipart/form-data` with `file` part and optional JSON metadata part.
- **Size Limit:** Capped at **10 MB** enforced via Spring Boot (`MaxUploadSizeExceededException`).
- **Client Execution:** Axios handles boundary generation automatically when passing a `FormData` object. Fully compatible with WebView2 file picker dialogs (`tauri-plugin-dialog`).

### 5.2 Binary Document Downloads (PDFs & Excel Sheets)
- **Use Cases:** Student grade transcripts, fee receipts, departmental attendance registers.
- **Response Headers:**
  - `Content-Type: application/pdf` or `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
  - `Content-Disposition: attachment; filename="FeeReceipt_REC-10492.pdf"`
- **CORS Exposure:** `CorsConfig.java:22` explicitly exposes `Content-Disposition`.
- **Desktop Handling:**
  - In Web: Triggers standard browser download shelf.
  - In Desktop: The desktop shell intercepts the download stream, allowing the user to select an exact local save path via Windows native File Save Dialog (`SaveFileDialog`), writing the file directly to the user's `Downloads` or `Documents` directory.

---

## 6. Backward Compatibility & Non-Breaking Evolution

To prevent client-server synchronization breakage when deploying desktop updates across hundreds of institutional endpoints:
1. **Additive Schema Rule:** Backend DTO changes must be strictly additive (new fields marked as optional in JSON). Existing fields are never renamed or deleted within the `/api/v1` namespace.
2. **Missing Field Tolerance:** Both clients use TypeScript interfaces and Zod schemas with fallback defaults, ensuring that if an older backend version omits a newly added field, the UI renders gracefully.
3. **Major Breaking Changes:** Any non-backwards-compatible schema changes necessitate incrementing the route prefix to `/api/v2/*`, allowing legacy desktop clients to operate undisturbed until scheduled institutional upgrade windows.

Proceed to **Work Package 05D: Authentication and Session Compatibility Architecture**.
