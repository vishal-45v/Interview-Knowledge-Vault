# JDBC — Diagram Explanations

## 1. JDBC Architecture

The JDBC API separates your application code from the database vendor's specific protocol. Your application talks to the standard `java.sql` interfaces; the vendor-supplied driver translates those calls into the wire protocol the database understands.

```
┌──────────────────────────────────────────────┐
│              Java Application                │
│  (uses java.sql.Connection, Statement, etc.) │
└────────────────────┬─────────────────────────┘
                     │  Standard JDBC API calls
                     ▼
┌──────────────────────────────────────────────┐
│              JDBC API Layer                  │
│  (java.sql, javax.sql interfaces)            │
└────────────────────┬─────────────────────────┘
                     │  Delegates to registered driver
                     ▼
┌──────────────────────────────────────────────┐
│              JDBC Driver                     │
│  (e.g., postgresql-42.x.jar, mysql-connector)│
│  Implements: Driver, Connection, Statement   │
└────────────────────┬─────────────────────────┘
                     │  Vendor-specific wire protocol
                     │  (TCP/IP socket, TLS optional)
                     ▼
┌──────────────────────────────────────────────┐
│              Database Server                 │
│  (PostgreSQL, MySQL, Oracle, H2, etc.)       │
└──────────────────────────────────────────────┘
```

Key insight: because the application only uses standard `java.sql` interfaces, switching from MySQL to PostgreSQL requires changing the driver JAR and the connection URL — nothing in the application logic changes.

---

## 2. Connection Pool Lifecycle (HikariCP)

Connections in a pool transition through well-defined states. HikariCP tracks each connection's state internally and enforces timeouts at every stage.

```
  Application Startup
         │
         ▼
┌─────────────────┐
│   POOL CREATED  │  minimumIdle connections opened immediately
│   (idle pool)   │  maximumPoolSize is the ceiling
└────────┬────────┘
         │
         │  Thread calls dataSource.getConnection()
         ▼
┌─────────────────┐
│   IDLE          │◄──────────────────────────────────┐
│   (available)   │  waiting to be borrowed            │
└────────┬────────┘                                    │
         │  Borrowed by application thread             │
         ▼                                             │
┌─────────────────┐                                    │
│   IN-USE        │  connection.close() called         │
│   (acquired)    │──────────────────────────────────► │
└────────┬────────┘  returned to pool (not closed)     │
         │                                             │
         │  If exception / leak detected               │
         ▼                                             │
┌─────────────────┐                                    │
│   EVICTED       │  connection is truly closed        │
│   (removed)     │  new idle connection created       │
└─────────────────┘  to maintain minimumIdle           │
                                  │                    │
                                  └────────────────────┘

  Key Timeouts:
  ┌──────────────────────────┬──────────────────────────────────────┐
  │ connectionTimeout        │ max wait to borrow from pool (30s)   │
  │ idleTimeout              │ idle connection eviction time (10min) │
  │ maxLifetime              │ max age of any connection (30min)     │
  │ keepaliveTime            │ heartbeat to prevent stale conn       │
  │ leakDetectionThreshold   │ warn if held longer than this value   │
  └──────────────────────────┴──────────────────────────────────────┘
```

---

## 3. PreparedStatement Execution Flow — Parse Once, Execute Many

With a raw `Statement`, every execution is a full round trip through the SQL parser and query planner. With `PreparedStatement`, the heavy lifting happens only once.

```
  Statement (naive approach):
  ─────────────────────────────────────────────────────────
  Execution 1:  SQL String ──► Parse ──► Plan ──► Execute ──► Result
  Execution 2:  SQL String ──► Parse ──► Plan ──► Execute ──► Result
  Execution 3:  SQL String ──► Parse ──► Plan ──► Execute ──► Result
                              ↑           ↑
                         WASTED WORK   WASTED WORK (identical each time)

  PreparedStatement (optimized):
  ─────────────────────────────────────────────────────────
  prepare("SELECT * FROM users WHERE id = ?")
         │
         ▼
  SQL Template ──► Parse ──► Plan ──► [Cached Plan Stored in DB]
                                              │
  setInt(1, 101) ──► Execute with params ─────┤──► Result
  setInt(1, 202) ──► Execute with params ─────┤──► Result
  setInt(1, 303) ──► Execute with params ─────┘──► Result
         ↑
   Only parameters sent on each call (not full SQL)
   SQL injection impossible — params never interpreted as SQL
```

---

## 4. ResultSet Cursor Movement

The cursor is a server-side or client-side pointer into the result rows. The default ResultSet is `TYPE_FORWARD_ONLY`.

```
  Database returns rows: [Row1] [Row2] [Row3] [Row4] [EOF]

  Initial state:
  Cursor ──► [BEFORE_FIRST]  [Row1] [Row2] [Row3] [Row4] [EOF]

  After rs.next():
  [BEFORE_FIRST]  Cursor ──► [Row1] [Row2] [Row3] [Row4] [EOF]
  rs.getString(1) reads Row1 data

  After rs.next():
  [BEFORE_FIRST]  [Row1]  Cursor ──► [Row2] [Row3] [Row4] [EOF]

  After rs.next():
  [BEFORE_FIRST]  [Row1]  [Row2]  Cursor ──► [Row3] [Row4] [EOF]

  After rs.next() (returns false — EOF reached):
  [BEFORE_FIRST]  [Row1]  [Row2]  [Row3]  [Row4]  Cursor ──► [AFTER_LAST]

  Scrollable ResultSet (TYPE_SCROLL_INSENSITIVE) also allows:
  ┌──────────────────────────────────────────────────────┐
  │ rs.previous()      — move cursor back one row        │
  │ rs.first()         — jump to first row               │
  │ rs.last()          — jump to last row                │
  │ rs.absolute(3)     — jump to row 3                   │
  │ rs.relative(-2)    — move 2 rows backward            │
  └──────────────────────────────────────────────────────┘
  WARNING: scrollable ResultSets buffer all rows in memory
```

---

## 5. Transaction Begin / Commit / Rollback Flow

JDBC transactions are controlled by turning off auto-commit and manually managing boundaries.

```
  conn.setAutoCommit(false)
         │
         ▼
  ┌─────────────────────────────────────────────────────┐
  │               TRANSACTION BEGINS                    │
  │                                                     │
  │  stmt.executeUpdate("INSERT INTO orders ...")  ─────┤─► DB staged (not visible to others)
  │  stmt.executeUpdate("UPDATE inventory ...")    ─────┤─► DB staged
  │  stmt.executeUpdate("INSERT INTO audit ...")   ─────┤─► DB staged
  │                                                     │
  └──────────┬──────────────────────┬───────────────────┘
             │  All OK              │  Exception thrown
             ▼                      ▼
      conn.commit()           conn.rollback()
             │                      │
             ▼                      ▼
   ┌─────────────────┐    ┌──────────────────────┐
   │  ALL 3 CHANGES  │    │  ALL 3 CHANGES       │
   │  PERMANENTLY    │    │  UNDONE — DB is in   │
   │  WRITTEN TO DB  │    │  original state      │
   └─────────────────┘    └──────────────────────┘

  Savepoints allow partial rollback:
  conn.setAutoCommit(false)
  Savepoint sp1 = conn.setSavepoint("after_insert");
  // ... more statements ...
  conn.rollback(sp1);   // undo only work after sp1
  conn.commit();        // commit work up to sp1
```

---

## 6. Batch Insert Execution Model

Batch execution collapses N network round-trips into one, dramatically reducing latency for bulk operations.

```
  Without Batch (N = 5 inserts, 5 round-trips):
  ─────────────────────────────────────────────────────────
  Java App                         Database
     │──── INSERT row 1 ──────────────► │
     │◄─── ACK ──────────────────────── │
     │──── INSERT row 2 ──────────────► │
     │◄─── ACK ──────────────────────── │
     │──── INSERT row 3 ──────────────► │
     │◄─── ACK ──────────────────────── │
     │──── INSERT row 4 ──────────────► │
     │◄─── ACK ──────────────────────── │
     │──── INSERT row 5 ──────────────► │
     │◄─── ACK ──────────────────────── │
  Total: 5 × (network latency + DB overhead)

  With Batch (1 round-trip):
  ─────────────────────────────────────────────────────────
  Java App                         Database
  pstmt.addBatch()  × 5
     │                                  │
     │──── BATCH [row1,row2,row3,        │
     │           row4,row5] ──────────► │
     │                                  │  Execute all 5 sequentially
     │◄─── [1,1,1,1,1] ─────────────── │  (counts per statement)
  Total: 1 × (network latency + 5 × DB overhead)

  Best practice: commit every N rows (e.g., 1000) to avoid
  locking entire table and exhausting transaction log space.

  pstmt.addBatch() × 1000 ──► executeBatch() ──► commit()
  pstmt.addBatch() × 1000 ──► executeBatch() ──► commit()
  ...repeat...
```
