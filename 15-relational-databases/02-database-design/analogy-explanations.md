# Chapter 02 — Database Design: Analogy Explanations

Making normalization, constraints, and schema patterns concrete through
everyday analogies.

---

## Analogy 1: Normalization — The Messy Notebook vs The Filing Cabinet

**The Story:**
Imagine a student who writes all their class notes in one giant notebook. Every
time they write a new entry, they also write the teacher's full name, the room
number, and the department name. If the teacher's room changes, the student must
flip through all 500 pages and correct every occurrence.

A well-organized student instead maintains:
- A class roster (class_id → teacher, room, department)
- Their actual notes (note_id, class_id, content)

If the room changes, they update ONE line in the roster. All notes automatically
reference the correct room because they point to the roster entry.

**Connection:**
This is exactly what normalization achieves. The messy notebook has update anomalies
(you must update many rows when one fact changes), insert anomalies (you can't record
a teacher's room without creating a note), and delete anomalies (deleting the last
note loses the teacher's room information).

```sql
-- Messy notebook: update anomaly
UPDATE class_notes SET teacher_room = 'B204'
WHERE teacher_name = 'Prof. Smith';  -- Must update EVERY row

-- Normalized: update one row
UPDATE classrooms SET room = 'B204' WHERE teacher_id = 42;
-- All notes that JOIN to teacher_id = 42 automatically see the new room
```

---

## Analogy 2: Foreign Keys — The ID Badge at a Secure Building

**The Story:**
A company building has a security desk. Every visitor is given a badge with an
employee ID number. The badge works ONLY if that employee ID exists in the
employee database. If the employee is fired and removed from the system, their
ID badge immediately stops working — you cannot badge in using a deleted employee's
number.

This is exactly what a foreign key constraint does: it prevents a child row from
referencing a parent row that does not exist, and controls what happens to the
child rows if the parent is later deleted.

**Connection:**
```sql
-- Orders cannot reference a non-existent customer
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id)
        ON DELETE RESTRICT    -- Cannot delete customer if orders exist
        ON UPDATE CASCADE     -- If customer ID changes, update all order references
);

-- Attempting to create an order for customer_id = 9999 (doesn't exist):
INSERT INTO orders (customer_id) VALUES (9999);
-- ERROR: insert or update on table "orders" violates foreign key constraint
```

ON DELETE CASCADE is like saying "if an employee is deleted from the system,
their badge (child records) are immediately shredded." ON DELETE RESTRICT is like
saying "you cannot remove an employee from the system if they still have active
badges outstanding — revoke the badges first."

---

## Analogy 3: The 2NF Partial Dependency — Course Catalog on a Transcript

**The Story:**
Imagine a university transcript with one big table: (StudentID, CourseID,
CourseName, CourseCredits, Grade). The primary key is (StudentID, CourseID) —
you need both to identify a unique grade.

But here's the problem: CourseName and CourseCredits are properties of the Course,
not of the Student-Course combination. Every time the same course appears on
another student's transcript, CourseName and CourseCredits are duplicated again.

If a course is renamed, someone must update every single transcript entry for that
course across all students. A 500-student enrollment has 500 rows to update for
one course rename.

**Connection:**
This is a partial dependency — CourseName depends only on CourseID, not the full
composite key (StudentID, CourseID).

```sql
-- Violates 2NF: course_name and credits depend only on course_id, not (student_id, course_id)
CREATE TABLE transcript (
    student_id   INT,
    course_id    INT,
    course_name  TEXT,   -- partial dependency
    credits      INT,    -- partial dependency
    grade        TEXT,
    PRIMARY KEY (student_id, course_id)
);

-- 2NF fix: extract the partial dependency to its own table
CREATE TABLE courses (id INT PRIMARY KEY, name TEXT, credits INT);
CREATE TABLE transcript_2nf (
    student_id INT,
    course_id  INT REFERENCES courses,
    grade      TEXT,
    PRIMARY KEY (student_id, course_id)
);
```

---

## Analogy 4: The 3NF Transitive Dependency — The Zip Code Problem

**The Story:**
An employee directory stores: (EmployeeID, Name, ZipCode, City, State). The
primary key is EmployeeID. At first glance this seems fine — everything depends
on EmployeeID.

But City and State don't actually depend on EmployeeID directly. They depend on
ZipCode. The chain is: EmployeeID → ZipCode → City → State. City and State are
determined transitively through ZipCode, not directly by EmployeeID.

If the city for a zip code is ever reclassified (e.g., a city merges with a
neighboring city), you must update every employee row with that zip code. Thousands
of updates for one geographic change.

**Connection:**
```sql
-- Violates 3NF: city and state depend on zip_code, not employee_id
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    name        TEXT,
    zip_code    TEXT,
    city        TEXT,   -- transitive: zip_code → city
    state       TEXT    -- transitive: zip_code → state
);

-- 3NF fix
CREATE TABLE zip_codes   (zip_code TEXT PRIMARY KEY, city TEXT, state TEXT);
CREATE TABLE employees_3nf (
    employee_id INT PRIMARY KEY,
    name        TEXT,
    zip_code    TEXT REFERENCES zip_codes
);
```

---

## Analogy 5: Denormalization — The Pre-Computed Scoreboard

**The Story:**
A sports league has hundreds of thousands of games played over a decade. Every
time the standings page loads, should the application sum up all wins, losses,
and points from every historical game? Or should it maintain a `team_standings`
table that gets updated whenever a game result is recorded?

The normalized approach is "correct" in a database theory sense but impractical
for a page that loads every 30 seconds for 100,000 concurrent users. The
denormalized standings table is pre-computed, always slightly stale (within
seconds), but reads in microseconds instead of seconds.

**Connection:**
```sql
-- Normalized: correct but expensive for every read
SELECT team_id, SUM(CASE WHEN home_score > away_score THEN 1 ELSE 0 END) AS wins
FROM games WHERE season = 2024 GROUP BY team_id ORDER BY wins DESC;
-- Full table scan of millions of game rows every request

-- Denormalized: fast read, updated by trigger or background job
CREATE TABLE team_standings (
    team_id     INT PRIMARY KEY REFERENCES teams,
    season      INT,
    wins        INT NOT NULL DEFAULT 0,
    losses      INT NOT NULL DEFAULT 0,
    points      INT NOT NULL DEFAULT 0,
    last_updated TIMESTAMPTZ DEFAULT NOW()
);
-- Each game result update triggers an increment to standings: microsecond write
-- Each standings read: single row lookup by team_id
```

The cost: if a game result is corrected, the standings must be recomputed or
carefully adjusted. Write complexity increases; read complexity drops to near zero.

---

## Analogy 6: Surrogate vs Natural Keys — The Library Card vs The Book Title

**The Story:**
A library can identify a book in two ways:

1. **Natural Key:** The ISBN (e.g., 978-0-06-112008-4). Meaningful, globally
   unique (in theory), readable. But ISBNs can be duplicated by accident, can
   be reassigned in rare cases, and are 13 characters long — copying them into
   every borrowing record wastes space.

2. **Surrogate Key:** An internal accession number (e.g., #00042). Meaningless
   outside the library system, generated sequentially, 5 digits. Every borrowing
   record says "book #00042" — short, fast to join, never changes even if the
   ISBN is corrected in the catalog.

**Connection:**
```sql
-- Natural key approach: ISBN as PK
CREATE TABLE books (isbn TEXT PRIMARY KEY, title TEXT, author TEXT);
CREATE TABLE loans (
    loan_id     BIGINT PRIMARY KEY,
    book_isbn   TEXT REFERENCES books(isbn),  -- 13-char FK in every loan row
    borrower_id INT
);

-- Surrogate key approach: internal ID as PK, ISBN as unique constraint
CREATE TABLE books (
    id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    isbn   TEXT UNIQUE NOT NULL,  -- still enforced, just not the PK
    title  TEXT,
    author TEXT
);
CREATE TABLE loans (
    loan_id   BIGINT PRIMARY KEY,
    book_id   BIGINT REFERENCES books(id),  -- 8-byte INT, fast join
    borrower_id INT
);
-- If ISBN is corrected: UPDATE books SET isbn = '...' WHERE id = 42;
-- All loans still point to id=42 — zero cascade needed
```

---

## Analogy 7: Star Schema — The Hub and Spoke Airport

**The Story:**
An airline hub-and-spoke network routes all flights through a central hub.
From Chicago O'Hare, you can fly to Dallas (one connection to American's hub),
to Tokyo (one connection to United's hub), to London. You never need more than
one connection through the hub.

The analytics star schema works the same way. The fact table is the hub (every
flight = a sale transaction). The dimensions are the spokes (date, product,
customer, store). Any analytical query can be answered with at most one hop from
facts to any dimension.

A snowflake schema would be like a hub-and-spoke network where some spoke cities
also have their own smaller regional hubs — you sometimes need two connections
to reach your final destination.

**Connection:**
```sql
-- Star: one join from fact to any dimension
SELECT d.month, p.category, SUM(f.revenue)
FROM fact_sales f
JOIN dim_date    d ON f.date_key = d.key
JOIN dim_product p ON f.product_key = p.key
GROUP BY d.month, p.category;
-- Two joins regardless of how many levels of product hierarchy exist

-- Snowflake: extra joins to travel up the hierarchy
SELECT d.month, cat.name, SUM(f.revenue)
FROM fact_sales f
JOIN dim_date    d   ON f.date_key = d.key
JOIN dim_product p   ON f.product_key = p.key
JOIN dim_category cat ON p.category_key = cat.key  -- extra join
GROUP BY d.month, cat.name;
```

---

## Analogy 8: Soft Delete — The Museum's "On Loan" Label

**The Story:**
A museum never throws away artwork. When a piece is sent on loan to another
institution, they put a small "On Loan" label on the display stand but leave
the stand in place. All gallery maps still show the stand exists. New artwork
cannot be placed in that spot because the stand is still "there."

This is exactly the soft delete pattern. The row still physically occupies space
in all indexes. Unique constraint violations can occur with "deleted" rows. Every
map (query) must explicitly check for the "On Loan" label (`WHERE deleted_at IS NULL`).

**Connection:**
```sql
-- The "On Loan" label approach
ALTER TABLE products ADD COLUMN deleted_at TIMESTAMPTZ;

-- All queries must carry the filter — one forgotten filter leaks deleted data
SELECT * FROM products WHERE deleted_at IS NULL AND category = 'electronics';

-- Unique constraint broken by soft deletes:
ALTER TABLE products ADD CONSTRAINT unique_sku UNIQUE (sku);
-- Problem: deleted product with SKU 'ABC123' blocks creating a new product with same SKU
-- Fix: partial unique index
CREATE UNIQUE INDEX products_active_sku ON products (sku) WHERE deleted_at IS NULL;

-- Index bloat: all deleted rows still live in every index on the products table
-- In a table with 90% deleted rows, indexes are 10x larger than they need to be
```

The cleaner alternative for many use cases is an archive table:
```sql
-- Physical delete + archive = clean live table, historical data preserved
INSERT INTO products_archive SELECT *, NOW() AS archived_at FROM products WHERE id = 42;
DELETE FROM products WHERE id = 42;
```
