# Work Package 05A — Build, Testing & Deployment Baseline Audit

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 01)  
**Target Modules:** `.github/workflows/`, `backend/`, `frontend-v2/`, Docker & Cloud Configs  
**Date of Audit:** 1 November 2026  
**Auditor:** Antigravity AI Engineering Assistant (Pair Programming Baseline)

---

## 1. Executive Summary

This audit establishes the quality assurance, automated testing, continuous integration (CI), and deployment baselines for EduPulse. 

Key verified findings include:
- **Frontend Build Status:** Authoritative client (`frontend-v2/`) builds cleanly with zero errors in 5.14s (`tsc -b && vite build`), while the legacy `frontend/` directory is broken and causes CI failure.
- **Backend Build Status:** Backend compiles cleanly across all 371 source files and 69 test classes (`.\mvnw.cmd test-compile` in 11.89s).
- **H2 Test Execution:** Local unit and integration tests executing under the Spring Boot `test` profile succeed cleanly (`BatchControllerTest` passed 4/4 tests).
- **CI Workflow Discrepancies:** Multiple misconfigurations exist in GitHub Actions (`.github/workflows/ci.yml`), including invalid target directories and an undefined Maven profile (`postgres-it`).
- **Cloud Deployment Footprint:** In the previous web deployment model, the frontend deployed to Vercel (SPA rewrites), the backend to Render/AWS (Docker container on port 8081), and the database to Neon PostgreSQL.

---

## 2. Continuous Integration (CI/CD) Architecture

The GitHub Actions configuration is defined in `.github/workflows/ci.yml`. It specifies three parallel jobs triggered on push and pull requests to `develop` and `main`:

```
                       GitHub Actions CI (.github/workflows/ci.yml)
                                            │
               ┌────────────────────────────┼────────────────────────────┐
               ▼                            ▼                            ▼
        [Job: frontend]              [Job: backend]             [Job: backend-postgres-it]
        - Runner: ubuntu-latest      - Runner: ubuntu-latest    - Runner: ubuntu-latest
        - Node: 22                   - JDK: 21 (Temurin)        - Service: postgres:17
        - Dir: frontend/ (CRITICAL!) - Maven: clean test        - JDK: 21 (Temurin)
        - Commands: npm ci,          - Profile: Default (H2)    - Command: mvn verify
          tsc --noEmit, build                                     -P postgres-it (CRITICAL!)
```

### 2.1 Critical CI Defects Identified
1. **Frontend CI Directory Mismatch:**
   - Line 32 of `ci.yml` sets `working-directory: frontend`.
   - As established in WP01, `frontend/` contains only 0-byte dummy placeholder files (`pacakage.json` typo, 0-byte package-lock).
   - This causes `npm ci` in GitHub Actions to fail immediately on every PR.
   - *Fix Required for Desktop:* CI must target `working-directory: frontend-v2`.
2. **Undefined Maven Profile (`postgres-it`):**
   - Line 99 of `ci.yml` runs `mvn verify -P postgres-it`.
   - Inspection of `backend/pom.xml` reveals that **no `<profile>` element with id `postgres-it` exists**.
   - Consequently, the PostgreSQL integration test suite is not activated as intended in CI.

---

## 3. Test Suites & Quality Assurance Baseline

### 3.1 Backend Test Architecture
The backend maintains **69 test classes** partitioned across test tiers:

| Test Classification | Test Runner / Framework | Profile | Typical Target Scope |
| :--- | :--- | :--- | :--- |
| **Unit & MockMvc Tests** | JUnit 5 Jupiter, Mockito | `@ActiveProfiles("test")` | REST controller validation, DTO constraints, service logic |
| **H2 Integration Tests** | SpringBootTest, MockMvc | H2 In-Memory DB | Multi-domain workflows, RBAC filters, fee calculations |
| **PostgreSQL IT** | SpringBootTest, Testcontainers/Docker | `pgtest` profile (`application-pgtest.yml`) | Real PostgreSQL dialect verification, bytea photo persistence |
| **DevJson Sync Tests** | JUnit 5, File System | `dev-json` profile | Roundtrip JSON hydration & offline file synchronization |

### 3.2 Safe Test Execution Verification
- **Verified Execution:** `.\mvnw.cmd test -Dtest=BatchControllerTest`
- **Results:** Tests run: 4, Failures: 0, Errors: 0, Skipped: 0. Time elapsed: 2.115 s.
- **Total Execution Time:** 15.174 s (Clean build and pass).

---

## 4. Cloud & Container Deployment Topology (Web Baseline)

### 4.1 Frontend Cloud Deployment (Vercel)
- Configured via `frontend-v2/vercel.json`:
  ```json
  {
    "rewrites": [
      { "source": "/(.*)", "destination": "/index.html" }
    ]
  }
  ```
- All client paths rewrite to `index.html` to enable HTML5 pushState routing.

### 4.2 Backend Cloud Deployment (Render / AWS)
- Multi-stage container build via `backend/Dockerfile`.
- Runtime based on `eclipse-temurin:21-jre-jammy`.
- Non-root user `edupulse:edupulse` (UID: 1001).
- Port 8081 exposed with container liveness probe (`HEALTHCHECK --interval=30s CMD curl -f http://localhost:8081/api/health`).
- Production memory optimization: `-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0`.

### 4.3 Database Deployment (Neon Cloud PostgreSQL)
- External serverless PostgreSQL hosted on Neon.
- Enforces TLS 1.3 (`sslmode: require`).
- Connection pooling via HikariCP configured with 10 max connections and aggressive idle reclamation to suit serverless connection caps.

---

## 5. Desktop Transition Deployment Strategy

Moving to a Windows desktop application changes the deployment target:
1. **Packaging Target:** Windows 10/11 x64 executable / MSI installer.
2. **Frontend Distribution:** Packaged static bundle (`frontend-v2/dist`) embedded directly inside the desktop shell (Tauri WebView2 or Electron Chromium).
3. **API Target:** User-configurable remote server URL pointing to the institution's self-hosted or cloud-hosted EduPulse Spring Boot instance.
