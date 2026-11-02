# EduPulse ERP — Day 02, Work Package 09
# Deliverable 09e: Data Synchronization, Retention, and Disaster Recovery Architecture

## 1. Executive Summary & Core Architectural Invariants

EduPulse ERP Desktop Edition relies on a hybrid data architecture where PostgreSQL 17 serves as the authoritative, institutional System of Record (SoR) and an embedded, encrypted SQLCipher instance serves as a transient, read-only downstream cache.

To preserve data integrity, prevent split-brain anomalies, and guarantee swift recovery across hardware faults or network drops, the data synchronization, retention, and disaster recovery subsystems operate under four architectural invariants:

```
+-----------------------------------------------------------------------------------+
|                        DATA SYNCHRONIZATION & RECOVERY INVARIANTS                 |
|                                                                                   |
|   1. Absolute Server Authority (Server-Always-Wins):                             |
|      PostgreSQL holds the definitive state for all academic and financial data.   |
|      The desktop cache is downstream; in any divergence, server state overwrites. |
|                                                                                   |
|   2. Atomic Local Persistence:                                                    |
|      Synchronization occurs inside discrete, transactional SQLite batches.        |
|      Network failures or app crashes never leave the local cache partially torn.  |
|                                                                                   |
|   3. Ephemeral & Discardable Desktop Cache:                                       |
|      The local SQLCipher database can be wiped, corrupted, or deleted at any time |
|      without losing a single byte of institutional ERP data.                      |
|                                                                                   |
|   4. Zero Cross-Institution Residue:                                              |
|      Switching backend server endpoints cryptographically shreds the local cache, |
|      preventing data bleed between distinct institutional tenants.                |
+-----------------------------------------------------------------------------------+
```

---

## 2. Synchronization Architecture & Protocols

The synchronization pipeline transfers authorized operational datasets from the Spring Boot backend into the desktop's local SQLCipher cache.

```
+-------------------+                          +-------------------------+
|  React Desktop UI |                          | Spring Boot 3.3 Backend |
+-------------------+                          +-------------------------+
          |                                                 |
          | 1. Trigger Sync (Login / Timer / Reconnect)     |
          v                                                 |
+-------------------+                                       |
| Rust Sync Worker  |                                       |
+-------------------+                                       |
          |                                                 |
          | 2. Check Module ETag & 'since' Timestamp        |
          |------------------------------------------------>|
          |    GET /api/v1/sync/{module}?since={ts}         |
          |                                                 |
          | 3. Response: HTTP 304 (Unchanged)               |
          |    OR HTTP 200 OK (Paginated Batch + New ETag)  |
          |<------------------------------------------------|
          |                                                 |
          | 4. BEGIN IMMEDIATE TRANSACTION (SQLite)         |
          |    Upsert Entities -> Update sync_metadata      |
          |    COMMIT TRANSACTION                           |
          |                                                 |
          | 5. Emit 'sync-progress' Event                   |
          +------------------------------------------------>|
```

### 2.1 Initial Synchronization (Full Baseline Hydration)
Initial synchronization executes when the desktop client logs in for the very first time, or after the local cache has been cleared/rebuilt.

1. **Sequential Module Pipeline:** To maintain referential integrity in the local SQLite database, entities are synchronized in strict dependency order:
   - **Step 1:** System Metadata & Academic Calendar (`academic_year`, `curriculum`).
   - **Step 2:** Courses, Subjects, and Rooms (`course`, `subject`, `room`).
   - **Step 3:** Class Sections and Rosters (`section`, `students`, `guardians`).
   - **Step 4:** Weekly Timetable Slots (`timetable_slot`, `time_slot`).
   - **Step 5:** Read-Only Attendance History & Notices (`leave_request`, `notifications`).
2. **Keyset Pagination:** Initial sync endpoints do not use expensive `OFFSET` pagination (which causes high I/O on PostgreSQL tables). Instead, they enforce keyset pagination:
   ```http
   GET /api/v1/sync/students?limit=250&after_id=STU-00500 HTTP/1.1
   Host: erp.institution.edu
   Authorization: Bearer <access_token>
   ```
3. **Progress Telemetry:** The Rust sync manager computes overall progress based on `total_records` returned in the first page header and emits real-time events to the React UI:
   ```json
   {
     "module": "students",
     "current_page": 4,
     "total_pages": 12,
     "processed_records": 1000,
     "total_records": 3000,
     "percentage": 33.3
   }
   ```

### 2.2 Incremental Synchronization (Delta Sync)
Once initial hydration is complete, subsequent synchronization cycles only fetch delta modifications:

1. **Conditional Timestamps:** Rust inspects `sync_metadata.last_synced_at` and passes it in ISO-8601 UTC format:
   ```http
   GET /api/v1/sync/students?since=2026-11-02T08:30:00.000Z HTTP/1.1
   If-None-Match: "w/9f2b8a7c1e"
   ```
2. **HTTP 304 Optimization:** If no rows have changed (`updated_at > :since`), Spring Boot returns `304 Not Modified` with zero body bytes, preserving network bandwidth and battery life.
3. **Upsert Execution:** For updated rows, Rust executes atomic SQLite UPSERT statements:
   ```sql
   INSERT INTO cached_students (
       student_id, institution_id, roll_number, first_name, last_name,
       grade_level, section, status, search_tokens, data_json, cached_at, updated_at
   ) VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9, ?10, datetime('now'), ?11)
   ON CONFLICT(student_id) DO UPDATE SET
       roll_number   = excluded.roll_number,
       first_name    = excluded.first_name,
       last_name     = excluded.last_name,
       grade_level   = excluded.grade_level,
       section       = excluded.section,
       status        = excluded.status,
       search_tokens = excluded.search_tokens,
       data_json     = excluded.data_json,
       cached_at     = excluded.cached_at,
       updated_at    = excluded.updated_at;
   ```

---

### 2.3 Deletion Detection in Hard-Delete Architecture (No-Soft-Delete Gap)

#### The Architectural Challenge
As confirmed in the WP09 Database Audit (`09a`), EduPulse PostgreSQL entities **do not implement soft deletes**. There are no `deleted_at` columns, no `@SQLDelete` annotations, and no `@Where` filters. When an administrator removes a student or timetable entry, the backend executes a hard SQL `DELETE FROM table WHERE id = ?`.

As a consequence, naive incremental queries (`SELECT * FROM table WHERE updated_at > :since`) **cannot discover deleted records**, resulting in orphaned, zombie records in the local desktop cache.

#### Dual-Phase Deletion Reconciliation Strategy

```
+-----------------------------------------------------------------------------------+
|                        DUAL-PHASE DELETION RECONCILIATION                         |
|                                                                                   |
|  Phase 1: Short-Term Desktop Workaround (Zero Backend Schema Changes)             |
|  - Cryptographic Module Hash & Compact Active-ID Reconciliation Set               |
|  - Desktop executes set difference: DELETE WHERE id NOT IN (:active_ids)          |
|                                                                                   |
|  Phase 2: Long-Term Backend Enhancement (Dedicated Audit Table)                   |
|  - PostgreSQL `deletion_tombstones` table populated by JPA/DB triggers            |
|  - Delta query: SELECT entity_id FROM deletion_tombstones WHERE deleted_at > :ts  |
+-----------------------------------------------------------------------------------+
```

#### Phase 1: Cryptographic Hash & Compact Active-ID Reconciliation (Immediate)
Without modifying the authoritative database schema:
1. **Lightweight Hash Check:** Every 60 minutes or upon manual sync, Rust calls:
   ```http
   GET /api/v1/sync/{module}/reconciliation-hash
   ```
   The backend computes a SHA-256 hash across all active entity IDs and their `updated_at` values for that institution:
   ```json
   {
     "module": "students",
     "record_count": 2842,
     "reconciliation_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
   }
   ```
2. **Local Comparison:** Rust computes the same SHA-256 hash over its local `cached_students` table.
3. **Set Difference Sweep:** If hashes diverge:
   - Rust requests a compact binary array or bitset of all active IDs: `GET /api/v1/sync/students/active-id-manifest`.
   - Compares the server ID set against local SQLite IDs.
   - Deletes any local records absent from the server manifest:
     ```sql
     DELETE FROM cached_students WHERE student_id NOT IN (/* active server IDs */);
     ```

#### Phase 2: Dedicated Deletion Tombstones Table (Long-Term Spring Boot Enhancement)
To eliminate ID reconciliation overhead entirely, future backend sprints will introduce a generic tombstone table:
```sql
CREATE TABLE IF NOT EXISTS deletion_tombstones (
    id BIGSERIAL PRIMARY KEY,
    entity_name VARCHAR(64) NOT NULL,       -- e.g., 'Student', 'TimetableSlot'
    entity_id VARCHAR(64) NOT NULL,         -- Prefixed ID or UUID
    deleted_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tombstones_lookup ON deletion_tombstones (entity_name, deleted_at DESC);
```
During delta sync, Rust queries `/api/v1/sync/tombstones?since=:timestamp` and deletes the matching local rows in milliseconds.

---

### 2.4 Synchronization Triggers and State Machine

```
      +-------------+        Network Available       +-------------+
      |             | -----------------------------> |             |
      |   OFFLINE   |                                |   IDLE      |
      |             | <----------------------------- |             |
      +-------------+         Network Dropped        +-------------+
             ^                                           |   ^
             |                                 Trigger   |   | Sync Done
             |                                 Fired     |   | (or Timeout)
             |                                           v   |
             |                                       +-------------+
             +-------------------------------------- |  SYNCING    |
                           Fatal Error               +-------------+
```

Sync cycles are initiated by five discrete application triggers:
1. **User Authentication:** Baseline sync kicks off immediately after login tokens are verified.
2. **Periodic Interval:** A background Tokio task in Rust triggers an incremental sync check every 15 minutes.
3. **Network Reconnection:** The Windows network state monitor (`INetworkListManager`) signals when internet connectivity returns after an outage.
4. **Window Focus / Wake:** Resuming from sleep or focusing the desktop window checks if the cache is older than 30 minutes.
5. **Manual User Refresh:** A dedicated "Sync Now" button in the React status bar triggers an immediate foreground sync.

---

## 3. Atomic Transactional Boundaries and Interruption Handling

Synchronization across a wide-area network or campus Wi-Fi is inherently prone to packet loss, connection resets, and laptop lid closure. EduPulse enforces strict transactional isolation to guarantee database consistency:

### 3.1 SQLite Immediate Transactions
All updates for a module batch are wrapped in an immediate transaction:
```rust
let mut tx = conn.transaction_with_behavior(TransactionBehavior::Immediate)?;
for record in batch.records {
    upsert_statement.execute(params![...])?;
}
// Update sync metadata atomically within the same commit
tx.execute(
    "UPDATE sync_metadata SET last_synced_at = ?1, server_etag = ?2, sync_status = 'IDLE' WHERE module_key = ?3",
    params![batch.timestamp, batch.etag, module_key],
)?;
tx.commit()?;
```
- **Crash Immunity:** If the application process is terminated mid-sync, SQLite's Write-Ahead Log automatically rolls back uncommitted pages. The database returns cleanly to its previous consistent state on the next boot.
- **Zero Partial Hydration:** The `sync_metadata` table is updated **within the same transaction** as the entity rows. If the sync fails halfway through, the `last_synced_at` cursor remains unchanged, guaranteeing that the next attempt will resume properly.

---

## 4. Data Retention, Cache Purging, and Eviction Policies

Because desktop workstations may be shared among teachers, administrators, or lab students, rigorous retention and purging rules prevent data leakage.

### 4.1 Time-To-Live (TTL) Policy Matrix

| Cache Segment | Retention / TTL Policy | Eviction Mechanism | Purging Trigger |
| :--- | :--- | :--- | :--- |
| **Authentication Tokens** | Access Token: 15 min<br>Refresh Token: 7 days | In-memory expiry / DPAPI invalidation | Explicit user logout, token revocation, or 7-day expiry. |
| **Active Notices & Alerts** | Expires on `expires_at` date, or max 14 days | Automated SQLite pruning task | Pruned when `expires_at < date('now')`. |
| **Timetable Schedules** | Current academic term / semester | Overwritten on term transition | Replaced during semester rollover. |
| **Student Roster Cache** | Retained until tombstone deletion or account logout | Set difference or tombstone sweep | Removed when student withdraws or graduates. |
| **Attendance History** | Rolling 30 calendar days | Sliding window query pruning | Rows where `attendance_date < date('now', '-30 days')` purged daily. |
| **Student Photos** | LRU cache up to 100 MB | Evicted by least-recently-accessed | File unlinked when photo directory exceeds quota. |

### 4.2 Shared PC Protection vs. Dedicated Workstation Modes
EduPulse Desktop provides an administrative setting stored in DPAPI-protected configuration:

1. **Shared PC Mode (Default for Lab/Common PCs):**
   - On user logout, the desktop application immediately **executes a full cache purge**:
     ```sql
     DELETE FROM cached_students;
     DELETE FROM cached_timetables;
     DELETE FROM cached_announcements;
     PRAGMA incremental_vacuum;
     ```
   - Clears DPAPI refresh token and destroys cached student photo files.
   - Prevents the next Windows user from viewing institutional records.
2. **Dedicated Staff Laptop Mode:**
   - On user logout, cached operational data remains securely encrypted in SQLCipher at rest.
   - Only the DPAPI refresh token and in-memory access tokens are purged.
   - When the authorized teacher logs back in, offline cached schedules and rosters are instantly available without requiring a full re-sync.

### 4.3 Complete Cryptographic Destruction on Backend URL Switch
If an IT administrator or user changes the backend server URL in the desktop settings (e.g., migrating from cloud staging to an on-premise production server):
1. The Rust backend terminates all active database connections.
2. Deletes `%LOCALAPPDATA%\com.edupulse.erp\data\cache.key`.
3. Overwrites and unlinks `%LOCALAPPDATA%\com.edupulse.erp\data\cache.db`, `-wal`, and `-shm`.
4. Initializes a brand new SQLCipher database with a fresh 256-bit random key.
5. **Architectural Benefit:** Cryptographically guarantees that zero student records from Institution A remain accessible when pointing to Institution B.

---

## 5. Authoritative PostgreSQL Disaster Recovery & Backup Strategy

While the desktop cache is ephemeral, the authoritative PostgreSQL 17 database requires rigorous disaster recovery controls to meet institutional SLAs.

```
                               +-----------------------------+
                               | Primary PostgreSQL 17 (SoR) |
                               +-----------------------------+
                                      |               |
               Continuous WAL Shipping|               | Daily Physical Snapshot
                                      v               v
               +-----------------------+     +-----------------------+
               | Immutable WAL Archive |     | Base Backups (pg_back)|
               | (Encrypted S3 / NAS)  |     | (Encrypted Cold Store)|
               +-----------------------+     +-----------------------+
                                      \               /
                                       \             /
                               Point-In-Time Recovery (PITR)
                                             v
                               +-----------------------------+
                               | Restored Standby Instance   |
                               +-----------------------------+
```

### 5.1 Institutional Backup Architecture
1. **Continuous WAL Archiving (Point-in-Time Recovery):**
   - PostgreSQL `archive_mode` is set to `on`.
   - WAL segments (16 MB each) are securely transmitted to off-site, immutable object storage (AWS S3 with Object Lock or institutional private NAS) using tools like `pgBackRest` or `wal-g`.
   - Enables restoring the database to any specific second prior to an administrative error or ransomware incident.
2. **Daily Physical Base Backups:**
   - Full binary base backup executed nightly during off-peak hours (02:00 AM local time).
   - Retained for 30 days locally and 365 days in encrypted cold storage.
3. **Weekly Logical Exports (`pg_dump`):**
   - High-compression custom format dumps (`pg_dump -Fc`) taken weekly.
   - Serves as a schema-versioned, portable export for database upgrades or cross-cloud migrations.

### 5.2 Institutional SLA Targets (RTO & RPO)

| Metric | Target SLA | Implementation Architecture |
| :--- | :--- | :--- |
| **Recovery Point Objective (RPO)** | **< 15 minutes** | Continuous WAL archiving guarantees no more than 15 minutes of transactional loss in catastrophic datacenter failure. |
| **Recovery Time Objective (RTO)** | **< 2 hours** | Automated restore scripts using `pgBackRest` replaying WAL onto provisioned cloud/on-premise standby nodes. |
| **Desktop Self-Healing RTO** | **< 60 seconds** | Corrupt local caches rebuild automatically from the live PostgreSQL server. |

### 5.3 Automated Backup Verification & Drill Procedures
Backups that are never tested are not backups. EduPulse infrastructure mandates:
- **Weekly Automated Restore Drills:** A dedicated CI/CD worker automatically provisions an ephemeral PostgreSQL container, restores the previous night's base backup, replays WAL files, and runs schema verification tests.
- **Table Parity Verification:** Executes entity count probes across core tables (`students`, `users`, `finance_payment`) to verify data consistency post-restore.

---

## 6. Local Desktop Cache Recovery and Self-Healing Protocol

If a desktop client experiences disk block corruption, sudden laptop power loss during write operations, or unexpected SQLCipher header damage:

```
[ Desktop Launch ]
       |
       v
PRAGMA quick_check;
       |
       +---> [ Returns "ok" ] -----------> Resume Normal Desktop Operations
       |
       +---> [ Corruption Detected ] ----> 1. Force Close Connection Pool
                                           2. Rename cache.db -> cache.db.corrupt.<ts>
                                           3. Delete cache.key
                                           4. Generate New 256-bit Key via DPAPI
                                           5. Recreate Empty SQLite Schema
                                           6. Show UI Progress: "Restoring Cache..."
                                           7. Pull Full Initial Hydration from Backend
```

### 6.1 Diagnostic Triage & Quarantine
1. On application startup, Rust runs `PRAGMA quick_check;`.
2. If SQLite reports any b-tree corruption or missing pages:
   - Rust immediately severs the database connection pool.
   - Renames the file to `cache.db.corrupt.<timestamp>` to preserve forensic evidence for IT administrators.
3. Automatically wipes any orphaned `-wal` or `-shm` temporary files.

### 6.2 Self-Healing Rehydration Sequence
1. Rust initializes a clean, uncorrupted `cache.db` and applies all internal schema migrations.
2. The React frontend presents a non-blocking informational toast:
   > *"Local data cache was refreshed. Re-synchronizing records from campus server..."*
3. The background worker initiates a clean initial synchronization batch.
4. Within 30 to 60 seconds (depending on network latency), the local cache is fully restored to 100% parity with authoritative PostgreSQL state.
5. **No Data Loss:** Because EduPulse Desktop enforces zero offline writes in its initial release, not a single grade, attendance record, or payment entry is lost during a local cache failure.
