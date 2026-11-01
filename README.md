# EduPulse ERP — Windows Desktop Edition

**Enterprise Resource Planning (ERP) Suite for Higher Education Institutions**

---

## 1. Project Overview

EduPulse is transitioning from a web-based architecture into a dedicated, high-performance Windows desktop application. This repository (`Edupulse_ERP_desktop_v1.0`) is the official desktop development workspace, hosting the desktop shell, native integrations, and desktop-focused architecture.

The project encompasses a comprehensive functional roadmap across **23 domains** and **259 modular subsystems**:
- **Foundation Platform:** 46 modules (Identity, RBAC, campus infrastructure, system configuration)
- **Academic Domain:** 106 modules (Programs, curriculums, courses, registrations, timetable, attendance)
- **Enterprise Operations:** 107 modules (Admissions, student finance, examinations, HR, reporting)

---

## 2. Technology Stack & Architecture

- **Desktop Framework Candidates:** Tauri v2 (Microsoft Edge WebView2) and Electron v33+ (Chromium)
- **Frontend Client:** React 19.2, TypeScript, Vite 8.3, Tailwind CSS, TanStack Query v5, Zustand, Radix UI / shadcn
- **Backend Services:** Java 21 LTS, Spring Boot 3.3.4, Spring Security (Stateless JWT + RBAC), Spring Data JPA (Hibernate 6)
- **Persistence Layer:** PostgreSQL 15+, Flyway incremental schema versioning (`V1`–`V9`)
- **Target OS:** Windows 10 / Windows 11 (x64)

---

## 3. Workspace Structure

```
Edupulse_ERP_desktop_v1.0-main/
├── .gitignore                           # Desktop root ignore (excludes reference web bundle)
├── README.md                            # Desktop repository overview & guide
├── docs/                                # Authoritative desktop migration documentation
│   ├── GIT_REPOSITORY_VERIFICATION.md   # Initial Git topology & commit audit
│   ├── week_00_docs/                    # Week 00 Foundation Sprint roadmaps
│   │   ├── week00_complete.md           # 7-day desktop transition execution plan
│   │   └── week00_day01.md              # Day 01 reconnaissance mandate
│   └── desktop-migration/
│       └── day-01/                      # 9 Day 01 technical audit reports
│           ├── 01-workspace-audit.md
│           ├── 02-frontend-architecture.md
│           ├── 03-backend-architecture.md
│           ├── 04-database-audit.md
│           ├── 05-authentication-security-audit.md
│           ├── 06-build-test-deployment-baseline.md
│           ├── 07-technical-debt-and-risks.md
│           ├── 08-desktop-migration-readiness.md
│           └── 09-day-01-executive-summary.md
└── Institutioanl_ERP_v1.0-main/         # External reference web application archive (git-ignored)
```

---

## 4. Documentation & Migration Baseline

All architectural investigations, risk assessments, and baseline reports from the Day 01 Technical Audit Sprint are preserved in `docs/desktop-migration/day-01/`. Consult `09-day-01-executive-summary.md` for a complete overview of build statuses, CI findings, and technical debt.

---

## 5. Development Roadmap (Week 00)

- **Day 01:** Repository Reconnaissance & Technical Audit Baseline (Completed)
- **Day 02:** Desktop Architecture, Security & Technology Decisions (Tauri vs. Electron)
- **Day 03:** Desktop Shell Initialization & Windows Build Verification
- **Day 04:** Frontend Integration & Desktop UI Adaptation
- **Day 05:** Backend Integration & Native Security Foundation
- **Day 06:** Domain & Functional Module Inventory Audit
- **Day 07:** Week 00 Review, Architecture Consolidation & Week 01 Backlog
