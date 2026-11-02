# EduPulse ERP — Day 02, Work Package 09
# Deliverable 09d: Local SQLCipher Cache Database Architecture

## 1. Executive Summary and Five Core Architectural Invariants

EduPulse ERP Desktop Edition integrates an encrypted, embedded SQLite database powered by **SQLCipher** (via Rust `rusqlite` compiled with the `bundled-sqlcipher` feature). The database file resides strictly within the user's isolated Windows AppData profile:
```text
%LOCALAPPDATA%\com.edupulse.erp\data\cache.db
```

This local database operates under five immutable architectural invariants:

```
+-----------------------------------------------------------------------------------+
|                        FIVE LOCAL CACHE ARCHITECTURAL INVARIANTS                  |
|                                                                                   |
|   1. PostgreSQL is the Sole Authoritative System of Record:                       |
|      All official academic, enrollment, and financial state lives in PostgreSQL.  |
|                                                                                   |
|   2. SQLCipher is Strictly a Downstream Read-Only Projection:                     |
|      The local database is an ephemeral replica populated via one-way sync.      |
|                                                                                   |
|   3. Zero Offline Writes in Initial Release:                                      |
|      No offline attendance submission, grade entry, or fee collection.            |
|                                                                                   |
|   4. Local Cache Never Becomes a Second Source of Truth:                          |
|      In any conflict or hash divergence, local data is overwritten by the server. |
|                                                                                   |
|   5. Sensitive Records Never Exposed via Unrestricted IPC:                        |
|      Rust commands enforce role-based authorization before returning cached data. |
+-----------------------------------------------------------------------------------+
```

---

## 2. Selective Caching and Data Minimization Matrix

Of the 54 authoritative JPA entities and database tables in EduPulse PostgreSQL, only operational entities required for offline reference and low-latency navigation are cached. Highly sensitive records, financial ledgers, and raw authentication secrets are strictly excluded from the local desktop client.

| Functional Domain | Authoritative PostgreSQL Tables | Cached Locally? | Cache Model & Minimization Policy | Security Justification |
| :--- | :--- | :--- | :--- | :--- |
| **Authentication & IAM** | `users` (54) | **NEVER** | Zero user records, passwords, or password hashes cached. | Prevents offline dictionary attacks against argon2/bcrypt hashes on stolen laptops. |
| **Active Sessions** | N/A (In-memory Spring Security) | **EPHEMERAL** | Memory-only access token (15-min TTL). Refresh token encrypted in DPAPI vault. | Minimal attack surface; zero tokens in SQLite files. |
| **Student Demographics** | `students`, `guardians` | **YES (Sanitized)** | Normalized projection: Name, roll number, section, blood group, emergency phone. Bank accounts and national IDs **redacted**. | Allows emergency roster lookups while preventing mass identity theft. |
| **Student Photos** | `student_photos` | **YES (LRU Cache)** | BLOB/JPEG data stored in `%LOCALAPPDATA%\com.edupulse.erp\cache\photos\` with encrypted paths. | Rapid badge verification and roster viewing during network drops. |
| **Academics & Curricula** | `course`, `subject`, `curriculum`, `academic_year` | **YES** | Full offline reference: Course titles, credit hours, subject syllabi, academic terms. | Static catalog data; low sensitivity, high offline utility. |
| **Timetable & Scheduling** | `timetable_slot`, `time_slot`, `room`, `section` | **YES** | Master timetable view by room, class section, and instructor ID. | Teachers and students must view their schedules without active internet. |
| **Attendance History** | `student_attendance_record`, `leave_request` | **READ-ONLY (30 Days)** | Rolling 30-day historical attendance record per student. Read-only projection. | Attendance entry sheet is locked offline; viewing past records permitted. |
| **Fee Structures & Dues** | `fee_head`, `fee_structure`, `student_fee` | **READ-ONLY (Summary)** | Active fee breakdown and outstanding total balance per student. | Enables front-office inquiries while preventing offline balance tampering. |
| **Financial Ledger** | `finance_payment`, `finance_payment_allocation` | **NEVER** | Completely excluded from local cache. | Strict PCI-DSS and financial integrity requirement. Cash collection requires live backend. |
| **Administrative Notices** | `notifications` | **YES** | Active institutional announcements and emergency campus alerts. | Public broadcast information; ensures staff and students receive notices. |

---

## 3. Local Key Management Architecture

SQLCipher encrypts database pages at rest using 256-bit AES in CBC mode (or XTS mode depending on engine configuration) with HMAC-SHA512 per-page integrity verification. The key lifecycle is tied directly to the logged-in Windows user.

```
                    +------------------------------------+
                    | Windows OS (Cryptographic Provider)|
                    +------------------------------------+
                                      |
                      BCryptGenRandom(32 bytes)
                                      v
                        [ Raw 256-bit Master Key ]
                                      |
                    +-----------------+-----------------+
                    |                                   |
         dpapi::protect()                    zeroize::Zeroize
         (CurrentUser Scope)                 (Clears raw key from memory)
                    |
                    v
    %APPDATA%\...\data\cache.key
    (DPAPI Encrypted Blob)
```

### 3.1 Encryption Key Generation
1. During the initial application launch or first successful user login, the Rust background runtime checks for the existence of `%LOCALAPPDATA%\com.edupulse.erp\data\cache.key`.
2. If absent, Rust invokes the Windows Cryptography Next Generation (CNG) API (`BCryptGenRandom`) or `ring::rand::SystemRandom` to generate a cryptographically secure 256-bit (32-byte) pseudo-random master key.

### 3.2 Key Protection via Windows DPAPI
1. The 32-byte raw key is passed to the Windows Data Protection API:
   ```rust
   // CryptProtectData wrapper
   let entropy = b"EduPulse_Desktop_SQLCipher_Salt_v1";
   let encrypted_blob = dpapi::protect_with_entropy(&raw_key, entropy, CryptProtectScope::CurrentUser)?;
   std::fs::write(&key_path, encrypted_blob)?;
   ```
2. The raw key buffer in Rust memory is immediately cleared using the `zeroize::Zeroize` trait to eliminate plaintext residual keys in memory dumps.
3. The resulting DPAPI blob is saved to disk as `cache.key`. This file cannot be decrypted by any other Windows user profile, service account, or remote attacker without physical/NTLM session compromise of the logged-in user.

### 3.3 Database Key Application
When opening a connection in the Rust `r2d2` or `deadpool-sqlite` connection pool:
1. Rust reads `cache.key` and unprotects it via `CryptUnprotectData`.
2. Passes the unencrypted key to SQLCipher using a parameterized PRAGMA:
   ```sql
   PRAGMA key = "x'HEX_ENCODED_32_BYTE_KEY'";
   ```
3. Immediately zeroizes the hex string in memory.
4. Executes a validation probe:
   ```sql
   SELECT count(*) FROM sqlite_master;
   ```
   If the key is invalid or the header is corrupted, this query returns `SQLITE_NOTADB` (code 26), immediately terminating pool initialization.

---

## 4. SQLite and SQLCipher Engine Configuration

To guarantee ACID durability, high concurrency, minimal memory footprint, and immunity to unencrypted temp file leakage, EduPulse enforces the following SQLite pragmas on every newly established connection:

```sql
-- 1. Cryptographic and Page Parameters (must precede any database read/write)
PRAGMA cipher_page_size = 4096;
PRAGMA kdf_iter = 256000;
PRAGMA cipher_hmac_algorithm = HMAC_SHA512;
PRAGMA cipher_kdf_algorithm = PBKDF2_HMAC_SHA512;

-- 2. Concurrency and Journaling
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA busy_timeout = 5000;

-- 3. In-Memory Security and Leak Prevention
PRAGMA temp_store = MEMORY;

-- 4. Memory and Query Optimization
PRAGMA cache_size = -8000;        -- 8 MB in-memory page cache (negative = KiB)
PRAGMA mmap_size = 67108864;      -- 64 MB memory-mapped I/O limit
PRAGMA foreign_keys = ON;
PRAGMA auto_vacuum = INCREMENTAL;
```

### Architectural Rationale for Pragmas:
- **`journal_mode = WAL`:** Write-Ahead Logging allows simultaneous concurrent reads while the background synchronization task writes delta updates. SQLCipher encrypts WAL frames with the same 256-bit AES key.
- **`temp_store = MEMORY`:** Prohibits SQLite from writing unencrypted temporary files (such as large query sorts or intermediate index builds) to Windows `%TEMP%` or `%LOCALAPPDATA%\Temp`.
- **`synchronous = NORMAL`:** In WAL mode, `NORMAL` guarantees full ACID safety against application crashes without incurring the severe I/O penalties of `FULL` synchronization on every commit.
- **`auto_vacuum = INCREMENTAL`:** Allows the cache manager to reclaim free database pages via `PRAGMA incremental_vacuum(N);` after batch sync deletions without locking the database for an expensive full vacuum.

---

## 5. Local Cached Schema Definition

The local cache schema uses hybrid relational-document tables. Frequently queried search filters and foreign keys are modeled as indexed relational columns, while the complete entity representation is stored as a compressed JSON payload (`data_json`).

```sql
-- Table 1: Synchronization Tracking & Module State
CREATE TABLE IF NOT EXISTS sync_metadata (
    module_key TEXT PRIMARY KEY,             -- e.g., 'STUDENTS', 'TIMETABLE', 'NOTICES'
    last_synced_at TEXT NOT NULL,            -- ISO-8601 UTC timestamp
    entity_count INTEGER NOT NULL DEFAULT 0,
    server_etag TEXT,                        -- HTTP ETag from backend
    schema_version INTEGER NOT NULL,         -- Module schema revision
    sync_status TEXT NOT NULL                -- 'IDLE', 'SYNCING', 'STALE', 'ERROR'
);

-- Table 2: Cached Students Roster (Sanitized)
CREATE TABLE IF NOT EXISTS cached_students (
    student_id TEXT PRIMARY KEY,             -- Prefixed ID: 'STU-10024'
    institution_id TEXT NOT NULL,            -- Multi-campus isolation filter
    roll_number TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    grade_level TEXT NOT NULL,
    section TEXT NOT NULL,
    status TEXT NOT NULL,                    -- 'ACTIVE', 'INACTIVE', 'SUSPENDED'
    search_tokens TEXT NOT NULL,             -- Normalized 'roll_number first_name last_name'
    data_json TEXT NOT NULL,                 -- Sanitized StudentDTO JSON
    cached_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_students_search ON cached_students (search_tokens);
CREATE INDEX IF NOT EXISTS idx_students_cohort ON cached_students (institution_id, grade_level, section);
CREATE INDEX IF NOT EXISTS idx_students_roll ON cached_students (institution_id, roll_number);

-- Table 3: Cached Academic Timetables
CREATE TABLE IF NOT EXISTS cached_timetables (
    entry_id TEXT PRIMARY KEY,               -- Backend slot UUID / ID
    institution_id TEXT NOT NULL,
    day_of_week INTEGER NOT NULL,            -- 1 (Monday) to 7 (Sunday)
    period_number INTEGER NOT NULL,
    start_time TEXT NOT NULL,                -- '08:30:00'
    end_time TEXT NOT NULL,                  -- '09:15:00'
    grade_level TEXT NOT NULL,
    section TEXT NOT NULL,
    teacher_id TEXT NOT NULL,
    teacher_name TEXT NOT NULL,
    subject_code TEXT NOT NULL,
    subject_name TEXT NOT NULL,
    room_number TEXT NOT NULL,
    data_json TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_timetable_lookup ON cached_timetables (institution_id, grade_level, section, day_of_week);
CREATE INDEX IF NOT EXISTS idx_timetable_teacher ON cached_timetables (institution_id, teacher_id, day_of_week);

-- Table 4: Cached Announcements & Circulars
CREATE TABLE IF NOT EXISTS cached_announcements (
    notice_id TEXT PRIMARY KEY,
    institution_id TEXT NOT NULL,
    title TEXT NOT NULL,
    category TEXT NOT NULL,                  -- 'GENERAL', 'EXAM', 'EMERGENCY', 'HOLIDAY'
    target_role TEXT,                        -- 'ALL', 'FACULTY', 'STUDENTS'
    publish_date TEXT NOT NULL,
    expires_at TEXT,
    body_text TEXT NOT NULL,
    data_json TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_notices_published ON cached_announcements (institution_id, publish_date DESC);

-- Table 5: Cache Invalidation & Deletion Tombstones
CREATE TABLE IF NOT EXISTS cached_tombstones (
    entity_type TEXT NOT NULL,               -- 'STUDENT', 'TIMETABLE', 'NOTICE'
    entity_id TEXT NOT NULL,
    deleted_at TEXT NOT NULL,
    PRIMARY KEY (entity_type, entity_id)
);
```

---

## 6. Cache Size Management, Quotas, and Eviction Policies

To prevent EduPulse Desktop Edition from exhausting host disk space on lower-specification administrative PCs:

### 6.1 Storage Quotas
- **Target Working Size:** 50 MB to 100 MB for average institutions (5,000 students).
- **Soft Quota Warning:** **150 MB**. Sync worker raises a background notification to trigger LRU eviction.
- **Hard Quota Ceiling:** **250 MB**. If reached, sync worker refuses delta downloads until auto-pruning executes.

### 6.2 Pruning and Eviction Hierarchy
When storage exceeds the soft threshold:
1. **Tier 1 (Instant Eviction):** Delete expired announcements (`expires_at < datetime('now')`).
2. **Tier 2 (Rolling Eviction):** Prune cached student attendance records older than 30 days.
3. **Tier 3 (Thumbnail/Photo Eviction):** Evict least-recently-accessed student photo BLOBs from `%LOCALAPPDATA%\com.edupulse.erp\cache\photos\`.
4. **Tier 4 (Incremental Vacuum):** Run `PRAGMA incremental_vacuum(500);` to return freed pages to the operating system.

---

## 7. Database Integrity Verification and Self-Healing Recovery

Because desktop workstations may experience abrupt power loss, operating system crashes, or disk hardware faults, the desktop application executes a rigid integrity protocol.

```
                   [ Application Startup ]
                              |
                     PRAGMA quick_check;
                              |
               +--------------+--------------+
               |                             |
            "ok"                        Not "ok" (Corruption)
               |                             |
      [ Normal Startup ]            1. Close SQLite Connection
                                    2. Rename cache.db -> cache.db.corrupt.<ts>
                                    3. Re-initialize empty cache.db
                                    4. Apply migrations
                                    5. Trigger Full Initial Sync from Server
```

### 7.1 Startup Verification
On every desktop startup, before the Tauri frontend is rendered or IPC commands are serviced:
```sql
PRAGMA quick_check;
```
`PRAGMA quick_check` scans page headers, b-tree cell offsets, and key ordering in milliseconds without reading every individual row payload.

### 7.2 Corruption Handling & Self-Healing Protocol
If `PRAGMA quick_check` returns anything other than `"ok"`:
1. **Quarantine:** The active connection pool is forcefully torn down. The corrupted file is renamed to:
   ```text
   %LOCALAPPDATA%\com.edupulse.erp\data\cache.db.corrupt.<timestamp>
   ```
2. **Re-Initialization:** A fresh, zero-byte `cache.db` is created and initialized with the latest schema migrations.
3. **Authoritative Resynchronization:** The client flags `sync_metadata` as uninitialized. When the user logs in and establishes network connectivity with Spring Boot, a clean initial synchronization repopulates the local cache.
4. **Zero Impact on Institutional Records:** Because PostgreSQL is the authoritative system of record and the local cache holds no offline writes, corrupt cache recovery is completely transparent and loss-free.

---

## 8. Local Database Schema Migrations and Upgrade Compatibility

Desktop schema versioning cannot rely on server Flyway migrations because the desktop database runs in SQLite, has a distinct denormalized structure, and may be updated out-of-sync with backend deployments.

### 8.1 SQLite Migration Framework
Desktop schema evolution is managed via Rust using `rusqlite_migration`:
```rust
use rusqlite_migration::{Migrations, M};

pub fn get_desktop_migrations() -> Migrations<'static> {
    Migrations::new(vec![
        M::up("CREATE TABLE sync_metadata (...);"),
        M::up("CREATE TABLE cached_students (...);"),
        M::up("CREATE TABLE cached_timetables (...);"),
        M::up("CREATE TABLE cached_announcements (...);"),
        M::up("ALTER TABLE cached_students ADD COLUMN blood_group TEXT;"),
    ])
}
```

### 8.2 Safe Ephemeral Upgrade Principle
If a desktop software update installs a new version whose local schema migration fails (for example, due to an unrecoverable SQLite syntax limitation or constraint incompatibility):
- The runtime deletes the local `cache.db`.
- Recreates the schema clean from migration `M[0]` to `M[N]`.
- Resynchronizes from Spring Boot.
- This guarantees that desktop software updates never crash or lock users out due to local SQLite migration failures.
