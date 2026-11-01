# Work Package 01 — Workspace & Repository Technical Audit

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 01)  
**Date of Audit:** 1 November 2026  
**Audit Branch:** `audit/day-01-reconnaissance`  
**Base Commit:** `bcbad60 fix(student): align photo bytea mapping with PostgreSQL`  
**Auditor:** Antigravity AI Engineering Assistant (Pair Programming Baseline)

---

## 1. Executive Summary & Audit Context

EduPulse is currently undergoing a strategic architectural transition from a distributed web application to a dedicated, high-performance Windows desktop Enterprise Resource Planning (ERP) suite. The target ERP roadmap encompasses **23 functional domains** and **259 modular subsystems** across three core tiers:
- **Foundation Platform:** 46 modules
- **Academic Domain:** 106 modules
- **Enterprise Operations:** 107 modules

The current repository represents the authoritative reference implementation of the web-based system. The primary objective of Day 01 / Work Package 01 is to establish a rigorous, verified technical baseline of the parent workspace, filesystem topology, version control architecture, legacy artifacts, and build systems before initiating framework selection or code migration.

---

## 2. Parent Workspace & Filesystem Topology

The physical root folder provided for the desktop workspace is:
`C:\Users\digit\Desktop\Edupulse_ERP_desktop_v1.0-main`

### 2.1 Workspace Structure Overview

```
Edupulse_ERP_desktop_v1.0-main/
├── README.md                                    # Root workspace descriptor
├── docs/                                        # Desktop transition documentation
│   └── week_00_docs/
│       ├── week00_complete.md                   # 7-day Week 00 execution roadmap
│       └── week00_day01.md                      # Day 01 audit mandate and scope
└── Institutioanl_ERP_v1.0-main/                 # Extracted reference repository package
    ├── diagrams_&_images/                       # Architectural diagrams & design assets
    ├── module_thumbs/                           # UI thumbnail collateral
    ├── reports/                                 # Domain requirements & SWOT analyses
    ├── research_papers/                         # 18 academic research & domain papers
    ├── slide_exports/                           # Project presentation slides (PNG)
    ├── EduPulse_..._Question_Bank_500.docx       # Comprehensive 500-question audit bank
    ├── EDU_PULSE_Project_Presentation.pptx      # Stakeholder presentation deck
    ├── generate_presentation.py                 # Automated slide generator
    └── Institutioanl_ERP_v1.0/                  # Authoritative Git repository & codebase
```

### 2.2 Key Workspace Observations
1. **Nested Git Directory Structure:** The active Git repository is **not** located at the root of `Edupulse_ERP_desktop_v1.0-main`, nor at the root of `Institutioanl_ERP_v1.0-main`. It is strictly contained within the subfolder:
   `C:\Users\digit\Desktop\Edupulse_ERP_desktop_v1.0-main\Institutioanl_ERP_v1.0-main\Institutioanl_ERP_v1.0\.git`
2. **Project Documentation Split:** High-level project specifications, competitor analysis reports (Fedena, OpenEduCat, Microsoft Dynamics 365 Education, Oracle Student Cloud), and research papers reside outside the version-controlled application tree under `Institutioanl_ERP_v1.0-main/reports/` and `research_papers/`.

---

## 3. Version Control & Git Repository Baseline

### 3.1 Git Remote & Branch Configuration
- **Remote Origin URL:** `https://github.com/2204Zero/Institutioanl_ERP_v1.0.git`
- **Default Branch:** `main` (tracked with `origin/main`)
- **Key Upstream Branches:**
  - `origin/develop`
  - `origin/feature/authentication`, `origin/feature/student`, `origin/feature/database`, `origin/feature/dashboard`
  - `origin/copilot/fix-erp-ci-backend-failure`, `origin/copilot/fix-github-actions-job`
- **Active Working Branch Created for Audit:** `audit/day-01-reconnaissance`

### 3.2 Latest Commit History (Baseline)
```
bcbad60 2026-10-19 fix(student): align photo bytea mapping with PostgreSQL
d073fe7 2026-10-19 feat(student): implement durable PostgreSQL student profile photo storage
e6157bb 2026-10-19 feat(notification): implement production-persistent notifications with PostgreSQL
dc83cb5 2026-10-19 fix(finance): resolve student fee lazy loading
4685bf2 2026-10-19 fix(finance): resolve fee visibility and structure loading
730e615 2026-10-17 chore(ci): remove obsolete AWS EC2 deployment workflow
c2c73ce 2026-10-17 feat(prod): prepare production release with security fixes and V7 migration
```

### 3.3 Preserved Working Tree State (Safety Audit)
Prior to this audit, seven uncommitted modifications were present in the working tree. In strict adherence to workspace safety policies, these changes are untouched and preserved:

1. `backend/src/main/java/com/erp/devjson/model/FinanceJsonModels.java` (modified - deferred sync models)
2. `backend/src/main/java/com/erp/devjson/sync/DevJsonDataHydrator.java` (modified - deferred hydrator)
3. `backend/src/main/java/com/erp/devjson/sync/DevJsonSyncEngine.java` (modified - deferred sync engine)
4. `backend/src/test/java/com/erp/auth/RoleWorkspacesAndAuthorizationTest.java` (modified - fixture setup)
5. `backend/src/test/java/com/erp/finance/StudentFeeOpenInViewTest.java` (modified - H2 test isolation)
6. `backend/src/test/java/com/erp/notification/NotificationIntegrationTest.java` (modified - user collision handling)
7. `backend/src/test/java/com/erp/devjson/FeeStructureDevJsonRoundTripTest.java` (untracked - roundtrip test)

---

## 4. Subsystem & Directory Decomposition

Within `Institutioanl_ERP_v1.0/`, the repository exhibits the following directory breakdown:

| Directory | Type / Technology | Status | Role & Purpose |
| :--- | :--- | :--- | :--- |
| `backend/` | Java 21 / Spring Boot 3.3.4 | **Active & Authoritative** | Full REST backend, Spring Security, JPA/Hibernate, Flyway migrations |
| `frontend-v2/` | React 19.2 / TS / Vite 8.3 | **Active & Authoritative** | Complete multi-role institutional ERP frontend with shadcn/ui & TanStack Query |
| `frontend/` | Empty / Legacy Stub | **Dead / Deprecated** | Contains 0-byte placeholder files and misspelled `pacakage.json`. Legacy artifact |
| `src/` (root) | React 18.3 / TS / Vite 5.2 | **Superseded Prototype** | Standalone prototype only mounting `FinanceDashboardPage`. Replaced by `frontend-v2` |
| `database/` | SQL Scripts / Seeds | **Auxiliary / Reference** | Contains manual seed scripts and clean-schema references (`migrations/` is empty) |
| `infrastructure/` | Docker / Nginx / Monitoring | **Deployment Support** | Nginx reverse-proxy configs, Prometheus/Grafana compose, and deployment scripts |
| `edupulse/` | SQL / Spring Configs | **Legacy Staging** | Contains legacy module SQL files (`V10_library`, `V11_hostel`) and compose files |
| `scratch/` | Node / Python / Dumps | **Development Scratchpad** | Acceptance scripts, CDP Chrome headless dumps, and verification harnesses |
| `docs/` | Markdown Specifications | **Authoritative Documentation** | Extensive BRD, FRD, SRS, HLD, LLD, ADRs, test cases, and branch registry |
| `.github/` | GitHub Actions | **Active CI/CD** | Contains `ci.yml` matrix workflow for frontend, backend (H2), and PostgreSQL IT |

---

## 5. Duplicate, Legacy & Disconnected Directory Analysis

A critical finding of this repository audit is significant structural fragmentation across frontend implementations and configuration files:

### 5.1 Frontend Fragmentation (Three Concurrent Trees)
1. **`frontend-v2/` (Authoritative):**
   - Configured with modern React 19.2.8, Vite 8.3.0, TypeScript 6.0, TailwindCSS 3.4.19, and TanStack Query 5.104.1.
   - Comprehensive implementation containing 14+ full role dashboards, administrative modules, fee management, student admissions, attendance, and settings.
2. **`frontend/` (Dead Stub):**
   - Contains 0-byte dummy files (`App.tsx`, `eslint.config.js`, `index.html`, `pacakage.json`, `package-lock.json`, `tsconfig.json`, `vite.config.ts`).
   - *Critical CI Risk:* `.github/workflows/ci.yml` targets this directory (`working-directory: frontend`) rather than `frontend-v2/`.
3. **`src/` (Root Repository Prototype):**
   - Root `package.json` (`educational-erp-frontend` v2.4.0) points to root `src/main.tsx` and `src/App.tsx`.
   - Mounts solely `FinanceDashboardPage` within an `AppProviders` context. This was evidently a targeted dev prototype for finance module stabilization.

### 5.2 Migration & Database Script Fragmentation
- The authoritative database migrations applied by Spring Boot at runtime are strictly located in:  
  `backend/src/main/resources/db/migration/` (Versions `V1` through `V9`).
- The directory `Institutioanl_ERP_v1.0/database/migrations/` contains only an empty `.gitkeep`.
- Additional root-level SQL files (`V1_94_lines.sql`, `V1_lf.sql`, `V1_old.sql`, `fixture.sql`, `update-pass.sql`) represent historical schema debugging sessions and are not part of automated deployment.

---

## 6. Build Systems, Tooling & Configuration Baseline

### 6.1 Backend Build Tooling
- **Build Engine:** Apache Maven 3.9+ via included wrappers (`mvnw`, `mvnw.cmd`).
- **Core Descriptor:** `backend/pom.xml`
- **Compiler Configuration:** Java 21 (`<java.version>21</java.version>`), Lombok 1.18.34, ByteBuddy 1.17.5.
- **Active Spring Profiles:**
  - `dev`: In-memory H2 database, automated Flyway migration.
  - `prod`: External Neon PostgreSQL, SSL required, connection pooling.
  - `test`: H2 with `spring.jpa.open-in-view=false`.
  - `dev-json`: File-based JSON offline synchronization engine.

### 6.2 Frontend Build Tooling
- **Package Manager:** Node.js (v20+ recommended) with npm.
- **Build Engine:** Vite 8.3.0 (`frontend-v2/vite.config.ts`).
- **Type Checker:** TypeScript ~6.0.2 via `tsc -b`.
- **Linter:** `oxlint` (`.oxlintrc.json`).

### 6.3 Containerization & Environment Files
- `backend/Dockerfile`: Multi-stage Eclipse Temurin 21 alpine build.
- `Dockerfile.frontend`: Multi-stage Node 20 / Nginx alpine build.
- `docker-compose.yml`: Local full-stack orchestration (PostgreSQL 15, backend, frontend).
- `docker-compose.aws-json.yml`: Local offline JSON store testing profile.

---

## 7. Strategic Conclusions for Desktop Migration

1. **Authoritative Codebase Isolation:** All desktop UI migration activities must target `frontend-v2/` as the single source of truth for the client interface. The root `src/` and legacy `frontend/` folders must be left untouched as reference artifacts.
2. **Backend Decoupling:** The Spring Boot backend in `backend/` is fully modularized and decoupled from frontend rendering, providing a clean REST API foundation for desktop consumption.
3. **Dedicated Documentation Repository:** All Day 01 migration deliverables are consolidated under `docs/desktop-migration/day-01/` within the version-controlled repository to ensure permanent traceability.
