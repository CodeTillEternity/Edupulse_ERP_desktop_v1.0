# Work Package 04A — Database Architecture & Flyway Migration Audit

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 01)  
**Target Module:** `backend/src/main/resources/db/migration/` & JPA Entity Layer  
**Date of Audit:** 1 November 2026  
**Auditor:** Antigravity AI Engineering Assistant (Pair Programming Baseline)

---

## 1. Executive Summary

EduPulse persistence is engineered around a relational relational database model utilizing PostgreSQL 15+ in production (hosted on Neon Cloud) and H2 in-memory for unit and rapid integration testing. Schema evolution is managed strictly through Flyway database migrations (`V1` through `V9`). 

The schema comprises **54 relational tables** covering the entire institutional ERP operational domain. The database enforces strict relational integrity, composite foreign key constraints, unique business keys (e.g. roll numbers, enrollment numbers, fee heads), and index-backed idempotency constraints on financial transactions.

---

## 2. Flyway Migration Version Ledger

The authoritative migration chain is stored in `backend/src/main/resources/db/migration/`:

| Version | Migration Script | Scope & Description | Key Tables / Columns Affected |
| :--- | :--- | :--- | :--- |
| **V1** | `V1__baseline_erp_schema.sql` | Initial complete ERP baseline schema | 51 tables: Academic, Admission, Attendance, Finance, Students, Faculty, Campuses |
| **V2** | `V2__persistent_user_management.sql` | Enterprise user authentication & RBAC | Creates `users` table with username, hashed password, email, role, active status |
| **V3** | `V3__enforce_faculty_fk.sql` | Relational integrity for teaching staff | Enforces `department_id` and foreign key constraints on `faculty` table |
| **V4** | `V4__link_users_to_domain_entities.sql` | User-to-entity polymorphic linking | Adds `user_id BIGINT` foreign keys to `students` and `faculty` linking to `users` |
| **V5** | `V5__payment_idempotency.sql` | Financial integrity & idempotency | Adds unique constraint `transaction_reference` on `finance_payment` to prevent duplicate charges |
| **V6** | `V6__link_guardians_to_users.sql` | Guardian portal account linkage | Adds `user_id BIGINT` foreign key to `guardians` table linking to `users` |
| **V7** | `V7__add_missing_schema_columns.sql` | Production hardening & workflow alignment | Adds missing workflow state and audit columns across admission and finance entities |
| **V8** | `V8__create_notifications_table.sql` | In-app notification durability | Creates `notifications` table (`id`, `user_id`, `title`, `message`, `type`, `read`, `created_at`) |
| **V9** | `V9__create_student_photos_table.sql`| Durable PostgreSQL binary photo storage | Creates `student_photos` table with `BYTEA` photo payload, MIME type, size, and student FK |

---

## 3. Database Schema & Domain Table Inventory

The 54 persistent tables are organized into 8 functional ERP domains:

### 3.1 Academic & Curriculum Domain (16 tables)
- `institution`: Root multi-tenant educational institution records.
- `campus`: Physical campus facilities and sites.
- `department`: Academic divisions (e.g., Computer Science, Electrical Engineering).
- `program`: Degrees and majors (e.g., B.Tech, M.Sc) linked to departments.
- `academic_year`: Fiscal and calendar academic cycles.
- `semester`: Term periods with start and end dates.
- `batch`: Student cohort groupings by admission year.
- `course`: Master course catalog with credit weights.
- `course_prerequisite`: Directed acyclic course prerequisite requirements.
- `course_offering`: Term-specific course activations.
- `section`: Classroom student divisions within an offering.
- `section_faculty`: Faculty assignment and teaching roles per section.
- `section_student`: Student enrollment roster per section.
- `room`: Physical classroom capacity and building locations.
- `time_slot`: Master timetable periods (day of week, start time, end time).
- `timetable_slot`: Dynamic schedule mapping (room, section, faculty, slot).

### 3.2 Student Information Subsystem (SIS) (8 tables)
- `students`: Core demographic record (roll number, email, blood group, status).
- `student_academic_records`: GPA, CGPA, and semester academic standings.
- `student_enrollments`: Program, batch, semester, and enrollment number binding.
- `student_history`: Comprehensive state change audit trail.
- `student_status_history`: Longitudinal status transitions (ACTIVE, SUSPENDED, ALUMNI).
- `student_documents`: File metadata archive for uploaded compliance documents.
- `student_photos`: Durable binary storage (`BYTEA`) for official student profile images.
- `student_categories`: Caste/demographic categorization for quota and compliance.

### 3.3 Attendance Domain (4 tables)
- `student_attendance_record`: Session-level attendance marks (PRESENT, ABSENT, LATE, EXCUSED).
- `class_schedule`: Daily scheduled class occurrences.
- `attendance_alert`: Automated triggers when attendance drops below threshold.
- `attendance_correction`: Formal faculty workflow for retrospective attendance adjustments.

### 3.4 Finance & Revenue Domain (8 tables)
- `fee_head`: Master ledger heads (Tuition, Laboratory, Library, Hostel).
- `fee_structure`: Program and academic year fee templates.
- `fee_structure_component`: Line-item breakdown per fee structure.
- `student_fee`: Student individual fee assessment ledger.
- `student_fee_component`: Itemized breakdown of outstanding vs paid dues.
- `finance_payment`: Monetary transactions with transaction reference and payment mode.
- `finance_payment_allocation`: Allocation of payments against individual fee components.
- `finance_receipt`: Formal institutional receipts generated upon payment verification.

### 3.5 Admissions & Intake Domain (6 tables)
- `applicant`: External prospective applicant demographic record.
- `admission_application`: Submitted admission dossiers with status lifecycle.
- `applicant_document`: Uploaded candidate identification and mark sheets.
- `admission_inquiry`: Pre-application prospective queries and follow-ups.
- `add_drop_request`: Student course add/drop requests during registration windows.
- `credit_rule`: Departmental credit limit rules for course selection.

### 3.6 Faculty & Staff Domain (2 tables)
- `faculty`: Academic staff dossier, departmental assignment, and official email.
- `leave_request`: Faculty and staff leave tracking.

### 3.7 Identity & System Security (2 tables)
- `users`: Core authentication identity, role assignment, and password hash.
- `guardians`: Parent and emergency contact information linked to student and user.

### 3.8 Notifications (1 table)
- `notifications`: User-directed operational notifications and system alerts.

---

## 4. Integrity Constraints, Keys & Indexing Strategy

### 4.1 Primary & Foreign Keys
- All entities utilize synthetic 64-bit surrogate primary keys (`BIGSERIAL` in PostgreSQL / `BIGINT GENERATED BY DEFAULT AS IDENTITY`).
- Cascade rules are carefully parameterized: soft operational records prevent accidental cascade deletion, while dependent children (`student_fee_component`, `section_faculty`) maintain strict referential integrity.

### 4.2 Unique Business Constraints
- `students(roll_number)`: Strictly unique institutional roll number.
- `students(email)`: Unique institutional student email.
- `student_enrollments(enrollment_number)`: Unique registrar enrollment code.
- `users(username)` & `users(email)`: Unique login identities.
- `finance_payment(transaction_reference)`: Enforced via `V5` migration for transaction deduplication.
- `student_attendance_record(student_id, schedule_id, attendance_date)`: Unique compound constraint preventing duplicate roll calls for the same student in a single lecture.

---

## 5. Environment Configuration Comparison

| Architectural Aspect | Development (`dev`) | Production (`prod`) | Testing (`test`) |
| :--- | :--- | :--- | :--- |
| **Engine** | H2 (In-Memory) | PostgreSQL 15+ (Neon Cloud) | H2 (In-Memory) |
| **Connection URL** | `jdbc:h2:mem:devdb` | `${SPRING_DATASOURCE_URL}` | `jdbc:h2:mem:test_db` |
| **SSL Enforcement** | None | `sslmode: require` | None |
| **Hibernate DDL Auto**| `update` | `validate` (Strict) | `update` |
| **Flyway Migrations** | `enabled: true` | `enabled: true` | `enabled: false` |
| **Connection Pool** | Default H2 pool | HikariCP (Max: 10, Min: 2) | Default H2 pool |
| **Open-In-View (OSIV)**| `false` | `false` | `false` |

---

## 6. Desktop Deployment Implications & Database Strategy

When transitioning EduPulse into a desktop application, the persistence strategy must support institutional operational requirements:

1. **Standard Client-Server Model (Primary Desktop Mode):**
   - The desktop client connects over TLS to the institution's existing central PostgreSQL backend.
   - Zero local database installation required on desktop workstations.
   - Real-time multi-user consistency across registrar, finance, and attendance terminals.
2. **Offline Read-Only Cache (Future Phase):**
   - For intermittent campus connectivity, a local SQLite or embedded DuckDB/PGLite cache can store student rosters and cached schedules.
   - The verified schema structure easily maps to local SQLite tables for read-only offline lookups.
