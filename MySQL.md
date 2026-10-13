*Q1. How does a B-tree index work in MySQL (InnoDB)? Explain the clustered vs secondary index difference and what a covering index is?*

InnoDB stores every index as a B+ tree. A B+ tree is a balanced, high-fanout tree where internal nodes hold only keys and child pointers, and all the real entries live in linked leaf pages. The clustered index is the table itself: the primary key B+ tree whose leaves hold the full rows. A secondary index is a separate B+ tree whose leaves hold the indexed columns plus the primary key value, so a lookup through it costs a second trip into the clustered index. A covering index is one that contains every column a query needs, so MySQL never makes that second trip.

# How the B+ tree works

- Data is stored in 16 KB pages (the default innodb_page_size). Each tree node is one page, and each page read is potentially one disk I/O, so the design goal is to minimize page reads.

- Internal nodes hold only keys and child pointers. This makes them small, so fanout is large (roughly 1,000+ children per node for a BIGINT key).

- Leaf nodes hold the actual entries, sorted by key, and are linked in a doubly linked list. That makes range scans and ORDER BY on the index cheap: find the start leaf, then walk the list sideways.

- Depth stays tiny. With ~1,000 fanout and ~15 rows per leaf page, a 3-level tree holds ~15M rows and a 4-level tree holds billions. A point lookup is about 3-4 page reads, and the top levels are almost always cached in the buffer pool, so it’s often 1 real disk read.

- Inserts go into the correct leaf. If the page is full, it splits into two, and the split can propagate upward. The tree stays balanced automatically because it grows from the root, not the leaves.

# Clustered index

In InnoDB the table is the primary key B+ tree. There is no separate heap file.

- Leaf pages contain the entire row, ordered by primary key.
- If you don’t define a PK, InnoDB uses the first UNIQUE NOT NULL index. If none exists, it creates a hidden 6-byte DB_ROW_ID and clusters on that.
- Because there is exactly one physical ordering, there is exactly one clustered index per table.

# Secondary index

A secondary index is its own B+ tree, sorted by the indexed columns.

- Leaf entries are (indexed columns, primary key value), not a physical row pointer.
- Lookup is a double lookup (a “bookmark lookup”):

1. Traverse the secondary index to find the matching entries and their PKs.
2. For each PK, traverse the clustered index to fetch the full row.

# Covering index

A covering index satisfies the query entirely from the index. All columns in SELECT, WHERE, JOIN, and ORDER BY are in the index, so step 2 is skipped. EXPLAIN shows Using index in the Extra column.

CREATE TABLE orders (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  customer_id BIGINT NOT NULL,
  status      VARCHAR(20) NOT NULL,
  total       DECIMAL(10,2) NOT NULL,
  created_at  DATETIME NOT NULL,
  notes       TEXT,
  KEY idx_cust_status (customer_id, status)
);

-- Not covering: needs `total`, so a clustered lookup per row
SELECT total FROM orders WHERE customer_id = 42 AND status = 'PAID';

-- Covering: customer_id, status are in the index, and `id` (the PK) is implicitly in every secondary index
SELECT id, status FROM orders WHERE customer_id = 42;

To cover the first query, extend the index:

ALTER TABLE orders ADD KEY idx_cover (customer_id, status, total);

*Q2. You have INDEX(a, b, c). Which of these queries use it, fully or partially, and why? WHERE a=1, WHERE b=2, WHERE a=1 AND c=3, WHERE a=1 AND b>5 AND c=3, ORDER BY a, b*

An index on (a, b, c) is one B+ tree sorted by a, then b within each a, then c within each (a, b). MySQL can only seek through it by following that sort order from the left, with no gaps. The first column without an equality condition is the last one that can narrow the search. Everything after it can still be filtered, but only by scanning.

*Q3. Explain transaction isolation levels and the anomalies each one prevents. What is InnoDB’s default and how does it avoid phantom reads?*

Isolation levels trade consistency for concurrency. The SQL standard defines four, ordered by which anomalies they allow: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE. InnoDB’s default is REPEATABLE READ. It avoids phantoms with two mechanisms: MVCC snapshots for plain (non-locking) reads, and next-key locks (record lock plus gap lock) for locking reads and writes.

# The anomalies

- Dirty read: you read data another transaction has written but not committed. If it rolls back, you acted on data that never existed.

- Non-repeatable read: you read the same row twice and get different values, because another transaction committed an UPDATE or DELETE in between.

- Phantom read: you run the same range query twice and get a different set of rows, because another transaction committed an INSERT (or a delete) in that range.

- Not in the standard, but asked about:

1. Lost update: two transactions read a value, both compute from it, and the second write overwrites the first.
2. Write skew: two transactions read overlapping data, then write to different rows, and together they break an invariant (for example, two on-call doctors both go off-call because each saw the other still on).

# How InnoDB implements this

MVCC (consistent non-locking reads)

- Every row has hidden columns: DB_TRX_ID (the last transaction to modify it) and DB_ROLL_PTR (pointer into the undo log). Older versions form a version chain through the undo log.

- A read view records which transactions were active when it was created. When reading, InnoDB walks the version chain until it finds a version visible to that read view.

- Read views are created at different times by level:

1. READ COMMITTED: a new read view per statement, so each statement sees the latest committed data.
2. REPEATABLE READ: one read view created at the first consistent read in the transaction, reused for the whole transaction. This is why re-reads are repeatable.

- READ UNCOMMITTED skips this and reads the newest version, committed or not.

- Readers never block writers and writers never block readers, which is the main concurrency win.

*Q4. Why is OFFSET-based pagination a problem at scale, and what are the alternatives?*

