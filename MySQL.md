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

*Q4. InnoDB vs MyISAM and the storage engine architecture. Why is InnoDB the default (transactions, row-level locking, crash recovery, foreign keys)? What do the buffer pool, redo log, undo log and doublewrite buffer each do, and what happens on a crash and restart?*

In MySQL, a storage engine determines how tables store data, manage indexes, handle locking, and recover from failures. The two well-known engines are InnoDB and MyISAM, but **InnoDB is the default choice for modern transactional applications** because it supports ACID transactions, row-level locking, crash recovery, and foreign-key constraints.

### 1. InnoDB vs MyISAM

| Feature | InnoDB | MyISAM |
|---|---|---|
| Transactions | Supported | Not supported |
| Locking | Row-level locking for typical row operations, plus other locks | Table-level locking |
| Crash recovery | Redo/undo-based recovery | No equivalent transactional recovery |
| Foreign keys | Supported | Not enforced |
| MVCC | Supported | Not supported |
| Concurrent writes | Generally better | Writes can block other operations on the table |
| Full-text indexes | Supported | Supported |
| Typical use | Transactional applications | Legacy or specialized read-oriented workloads |

InnoDB is the better choice for systems such as restaurant ERP applications, where creating an order might involve updating an order header, inserting order items, and deducting inventory.

These operations should either all succeed or all fail. InnoDB transactions provide that guarantee when the application defines the correct transaction boundaries.

MyISAM lacks transactions and foreign-key enforcement, so maintaining consistency across related writes requires significantly more application-level handling.

Also, row-level locking does not mean every InnoDB operation locks only individual rows. Index-range locks, metadata locks, and other locking mechanisms can affect concurrency.

### 2. InnoDB storage architecture

Four important components are the buffer pool, redo log, undo log, and doublewrite buffer.

<text color="secondary" size="sm">Simplified architecture</text>

```text
            SQL Query
                |
                v
       InnoDB Storage Engine
                |
        +-------+--------+
        |                |
        v                v
   Buffer Pool        Undo Log
   (cached pages)     (old versions)
        |
        v
   Data / Index Pages
        |
        +-------> Redo Log
        |          (recovery)
        |
        +-------> Doublewrite Buffer
                   (page protection)
```

### 3. Buffer pool — caching data and index pages

The buffer pool is a memory area where InnoDB caches database pages, including table data and index pages.

When a query needs a page, InnoDB checks whether it is already in memory. If it is not, the engine reads the page from disk into the buffer pool.

When a transaction modifies a page, the change is initially made to the in-memory page. That page becomes a **dirty page** until the modified version is written to the data files.

A background process flushes dirty pages to disk as needed.

The buffer pool improves performance by reducing disk I/O. Its size is an important tuning parameter for database servers, especially those with a large working set.

However, a dirty page does not need to be written immediately for every transaction commit. InnoDB relies on its recovery mechanisms to make this safe.

### 4. Redo log — recovering committed changes

The redo log records information needed to reproduce changes to database pages after a crash.

Suppose an order's inventory quantity changes from 10 to 7. InnoDB modifies the in-memory page and records the corresponding redo information.

The key mechanism is **write-ahead logging (WAL)**: the relevant redo log records must be persisted before the corresponding dirty data page is allowed to be written to its final data-file location.

With the usual durable configuration, InnoDB flushes the necessary redo log records to durable storage at commit. This allows it to recover committed changes even if the modified data pages have not yet been flushed.

The redo log is therefore essential for crash recovery and efficient writes. It avoids requiring every changed data page to be written to its final location synchronously before a transaction can commit.

The exact durability guarantees depend on configuration, particularly `innodb_flush_log_at_trx_commit`, and on whether the operating system and storage hardware honor flush requests.

### 5. Undo log — rollback and MVCC

The undo log stores information that allows InnoDB to reverse changes and reconstruct earlier versions of rows.

It serves two important purposes:

- **Transaction rollback:** If a transaction fails before committing, InnoDB can undo its modifications.
- **MVCC:** A transaction may need to read an older committed version of a row while another transaction has modified it.

For example, imagine a transaction changes a product price from ₹1,000 to ₹900 but has not committed. Another transaction running under an appropriate isolation level may still see the previously committed price of ₹1,000.

InnoDB can reconstruct that earlier version using undo information.

Undo records are not simply a permanent history of every row change. Once old versions are no longer needed for rollback or consistent reads, purge processing can reclaim the associated undo information.

**Remember:** Redo supports recovery by replaying changes; undo supports rollback and reconstruction of older row versions.

### 6. Doublewrite buffer — protecting against partial page writes

A database page is commonly larger than the smallest unit the storage device can guarantee to write atomically. A crash during a page write can therefore leave a partially written or torn page.

InnoDB's doublewrite mechanism protects against this.

In simplified terms:

1. InnoDB prepares a dirty page for flushing.
2. It writes the page to the doublewrite area and makes that copy durable as required.
3. It writes the page to its final location in the table's data file.
4. If a crash leaves the final page torn, recovery can use the valid doublewrite copy to restore the page.

The doublewrite mechanism is different from the redo log. Redo reconstructs database changes; the doublewrite buffer protects against corruption caused by incomplete page writes.

The exact implementation varies by MySQL version and configuration. Modern InnoDB can use a dedicated doublewrite file or other supported arrangements, so it is not always accurate to think of the buffer as just a small region inside the redo log.

### 7. What happens when MySQL crashes and restarts?

Consider a transaction that updates inventory and commits, but the server crashes before the dirty data page reaches its final location.

A simplified recovery sequence is:

<list marker="number" gap={2}>
  <list-item>**Crash occurs.** Some modified pages may remain only in memory, and some disk pages may be incomplete.</list-item>
  <list-item>**Redo recovery.** InnoDB examines the redo log and reapplies required changes so that database pages reflect logged changes, including committed changes that had not reached their final data pages.</list-item>
  <list-item>**Page repair if needed.** If a page was torn during a write, the doublewrite mechanism may provide a valid copy for recovery.</list-item>
  <list-item>**Transaction cleanup.** Incomplete transactions are rolled back using undo information. Their uncommitted changes must not become visible as committed business data.</list-item>
  <list-item>**Normal operation resumes.** InnoDB continues serving queries, while background work such as flushing dirty pages and purging obsolete undo versions proceeds.</list-item>
</list>

The actual internal recovery process is more nuanced than this sequence, but the key idea is that InnoDB uses redo to recover changes and undo to remove incomplete transactions.

For example, if an order and its inventory deduction were committed in the same transaction, recovery preserves that committed transaction. If the transaction had not committed, recovery rolls back its changes rather than leaving an order without the corresponding inventory update.

### 8. How I would troubleshoot and tune InnoDB

In production, I would monitor:

- **Buffer pool hit rate and reads:** To understand whether the working set fits in memory.
- **Dirty pages and flushing:** To detect write pressure or insufficient flushing capacity.
- **Redo log pressure:** To identify excessive write activity or inadequate redo capacity.
- **Lock waits and deadlocks:** To detect contention and problematic transaction design.
- **Undo history and long-running transactions:** To identify transactions preventing old versions from being purged.
- **Disk latency and durability settings:** To understand commit latency and actual crash-durability guarantees.

I would avoid increasing memory or changing durability settings blindly. For example, relaxing redo-log flushing may improve throughput but can increase the amount of recently committed data at risk during an operating-system or server crash, depending on the configuration and failure mode.

### 9. Practical conclusion

For a transactional backend, I would choose InnoDB because it provides the mechanisms needed to maintain consistency under concurrent access and recover from failures.

The four components have distinct responsibilities:

- **Buffer pool:** Keeps frequently accessed data and index pages in memory.
- **Redo log:** Reconstructs logged changes after a crash.
- **Undo log:** Rolls back incomplete transactions and supports MVCC.
- **Doublewrite buffer:** Protects against torn database-page writes.

**The key principle is that InnoDB separates fast in-memory modification from durable recovery. The buffer pool improves performance, while redo, undo, and doublewrite mechanisms help preserve database correctness when transactions fail or the server crashes.**

*Q5. Index structure and design. How does a clustered index (primary key) differ from a secondary index in InnoDB, and why does a large or random primary key (e.g. UUID v4) hurt insert performance and secondary index size? Explain composite index ordering, the leftmost-prefix rule, covering indexes and prefix indexes.*

In MySQL, indexes help the database locate rows without scanning an entire table. In InnoDB, index design is especially important because the primary key determines the physical organization of the table's clustered index, and every secondary index also stores the primary key.

I focus on choosing an appropriate primary key, designing composite indexes around actual query patterns, and using covering indexes to reduce unnecessary table lookups.

### 1. Clustered index vs secondary index

**Clustered index:** In InnoDB, the table's rows are organized in the leaf pages of the clustered index, which is normally the primary key. The leaf pages contain the row data along with the primary-key ordering.

**Secondary index:** A secondary index stores its indexed columns and the corresponding primary-key value. When a query finds a matching secondary-index entry but needs additional columns, InnoDB uses the primary key to look up the row in the clustered index.

For example:

```sql
CREATE TABLE Products (
    Id BIGINT PRIMARY KEY,
    Name VARCHAR(150),
    CategoryId INT,
    Price DECIMAL(10, 2),
    INDEX idx_category (CategoryId)
);
```

The clustered index is organized by `Id`. The secondary index on `CategoryId` contains entries conceptually like:

```text
CategoryId → Primary Key
10         → 101
10         → 108
20         → 105
```

If I execute:

```sql
SELECT Name, Price
FROM Products
WHERE CategoryId = 10;
```

MySQL can find matching entries using `idx_category`, then perform clustered-index lookups to retrieve `Name` and `Price`.

This extra lookup is often called a **bookmark lookup** or secondary-to-clustered-index lookup.

### 2. Why do large or random primary keys hurt performance?

InnoDB stores the primary-key value in every secondary-index entry. Therefore, primary-key design affects not just the clustered index but also the size of every secondary index.

Consider two designs:

- A `BIGINT` primary key, typically 8 bytes.
- A UUID primary key, typically 16 bytes when stored in its compact binary form.

With a UUID, each secondary-index entry needs to carry a larger key. Across millions of rows and many secondary indexes, that increases storage requirements, reduces the number of entries that fit in each index page, and can increase memory and disk I/O.

The effects are especially significant when a table has several secondary indexes.

**Randomness introduces a separate problem.** UUID v4 values are randomly distributed, so inserts can target many different locations in the clustered B-tree. This can cause more page splits, less sequential insertion, and poorer page locality than a steadily increasing key.

By comparison, an increasing integer key usually directs new inserts toward the right-hand side of the clustered index. This often improves insertion locality, although it can also create a hot page under very high concurrent write loads.

A practical alternative is a time-ordered identifier, such as a correctly generated UUID v7, or a compact numeric primary key. Time-ordered identifiers improve insertion locality, but their effectiveness depends on encoding, byte ordering, generation patterns, and the workload.

I do not automatically reject UUIDs: they are useful for distributed ID generation and avoiding predictable sequential identifiers in exposed APIs. I weigh those benefits against index size and write performance.

### 3. Composite indexes and column ordering

A composite index contains multiple columns in a defined order.

For example:

```sql
CREATE INDEX idx_category_price
ON Products (CategoryId, Price);
```

This index is ordered first by `CategoryId` and then by `Price` within each category.

It is useful for queries such as:

```sql
-- Both columns
SELECT *
FROM Products
WHERE CategoryId = 10
  AND Price > 500;

-- Leftmost column only
SELECT *
FROM Products
WHERE CategoryId = 10;
```

However, it generally cannot efficiently perform a direct lookup on `Price` alone because `CategoryId` is the leading column.

This is the **leftmost-prefix rule**: a composite B-tree index can efficiently support lookups using its leading column or a leading sequence of columns.

For an index on `(A, B, C)`, common useful prefixes are:

- `(A)`
- `(A, B)`
- `(A, B, C)`

A query filtering only on `(B)` or `(B, C)` generally cannot use the index as an equally efficient direct lookup. MySQL may sometimes use an index scan or the Index Skip Scan optimization, depending on the query, statistics, and server version, but I do not design indexes around assuming those alternatives will apply.

### 4. How do I choose the order of composite-index columns?

I start with the actual query patterns, including filtering, sorting, grouping, and join conditions.

For example:

```sql
SELECT Id, Name, Price
FROM Products
WHERE CategoryId = 10
  AND Status = 'Active'
ORDER BY Price;
```

A potential index is:

```sql
CREATE INDEX idx_category_status_price
ON Products (CategoryId, Status, Price);
```

The equality conditions on `CategoryId` and `Status` establish a narrow range, and the index ordering by `Price` can help satisfy the requested sort.

My design considerations include:

- Put columns used in common equality conditions before range conditions when that best matches the query.
- Consider `ORDER BY` and `GROUP BY` requirements to reduce sorting.
- Consider selectivity, but do not blindly put the highest-cardinality column first; usable prefixes and query patterns matter more.
- Account for range conditions, which can limit how effectively later columns narrow the search.
- Avoid creating overlapping indexes without a demonstrated benefit, because each index increases storage and write overhead.

Index column order is a workload-specific decision, not a universal rule based only on cardinality.

### 5. Covering indexes

A **covering index** contains all the columns needed by a particular query, allowing MySQL to return the result directly from the index without looking up the full row in the clustered index.

For example:

```sql
CREATE INDEX idx_category_price_name
ON Products (CategoryId, Price, Name);
```

This query may be covered by that index:

```sql
SELECT CategoryId, Price, Name
FROM Products
WHERE CategoryId = 10
ORDER BY Price;
```

Because the index contains the filter, ordering, and selected columns, InnoDB may be able to answer the query using only the secondary index.

Covering indexes can reduce random I/O and improve read performance. However, adding many columns to an index increases its size and makes inserts, updates, and deletes more expensive.

I create covering indexes for important, frequent queries rather than trying to include every column in every index.

I verify the actual execution plan using `EXPLAIN` and, when available and appropriate, `EXPLAIN ANALYZE` to determine whether the index is used and whether the query still performs expensive row lookups or sorting.

### 6. Prefix indexes

A prefix index indexes only the first portion of a string column. It can reduce index size for long text values.

For example:

```sql
CREATE INDEX idx_product_name_prefix
ON Products (Name(20));
```

This indexes the first 20 characters of `Name`, rather than the complete value.

Prefix indexes are useful when values are long and their initial characters provide sufficient selectivity. However, they have limitations:

- Many values sharing the same prefix can reduce selectivity.
- The index cannot fully distinguish strings that differ only after the indexed prefix.
- It generally cannot cover a query that needs the complete string value.
- Unique prefix indexes enforce uniqueness only on the indexed prefixes, which can reject distinct full strings that share the same prefix.

For identifiers, email addresses, and other values requiring exact uniqueness, I carefully consider whether a prefix index is appropriate. For large text-search requirements, a full-text index or another specialized search strategy may be a better fit.

### 7. How I optimize indexes in a real application

For a production MySQL application, my workflow is:

1. Identify slow or frequent queries using query logs and performance monitoring.
2. Inspect the execution plan with `EXPLAIN`.
3. Check whether the query performs a full table scan, an expensive sort, or too many clustered-index lookups.
4. Design an index that matches the actual filter, join, and sorting pattern.
5. Re-test with representative data volumes and realistic query distributions.
6. Measure both read improvements and the additional write, storage, and maintenance costs.

For example, in a restaurant ERP, I might use a compact primary key for orders and an index on `(RestaurantId, CreatedAt)` for retrieving a restaurant's orders over a time range. If a frequent query also filters by status, I would evaluate an index such as `(RestaurantId, Status, CreatedAt)` against the real query workload.

**The key principle is that indexes are data structures with real storage and maintenance costs. A good primary key keeps clustered and secondary indexes efficient, while well-ordered composite and covering indexes accelerate important queries without creating unnecessary write overhead.**

*Q6. Reading EXPLAIN and optimizing slow queries. What do type (ALL, index, range, ref, const), key, rows, filtered and Extra (Using filesort, Using temporary, Using index) tell you? Walk through finding a slow query with the slow query log and fixing it, including EXPLAIN ANALYZE.*

When I optimize a slow SQL query, I start by measuring its actual performance rather than immediately adding indexes. I use the slow query log to identify expensive queries, `EXPLAIN` to understand the execution plan, and `EXPLAIN ANALYZE` to compare estimated costs with actual execution behavior.

My goal is to reduce unnecessary rows examined, disk I/O, sorting, temporary-table work, and repeated lookups without making inserts and updates unnecessarily expensive.

### 1. Understanding the `type` column

In MySQL's traditional `EXPLAIN` output, `type` describes the access method used to find rows. Generally, `const` and `ref` are more selective than `range`, `index`, or `ALL`, but the best access method depends on the query and table size.

| Type | Meaning | What I look for |
|---|---|---|
| `ALL` | Full table scan | Potential problem for large tables |
| `index` | Full index scan | May be useful for a covering query, but can scan many entries |
| `range` | Reads an index range | Common for ranges, inequalities, and some `IN` conditions |
| `ref` | Uses a non-unique index lookup | Common for equality joins and non-unique indexed columns |
| `const` | Reads at most one matching row using a constant unique lookup | Very efficient for primary-key or unique-key equality conditions |

For example:

```sql
SELECT *
FROM Orders
WHERE Id = 1001;
```

If `Id` is a primary key, MySQL can often use `const` access.

However:

```sql
SELECT *
FROM Orders
WHERE RestaurantId = 10;
```

If `RestaurantId` has a suitable non-unique index, the plan may use `ref`.

A `range` access might appear for:

```sql
SELECT *
FROM Orders
WHERE CreatedAt >= '2026-10-01'
  AND CreatedAt < '2026-11-01';
```

If `ALL` appears for a query on millions of rows, I investigate whether an appropriate index exists and whether the query can use it. A full scan is not automatically bad: scanning a small table or a large fraction of a table can be cheaper than performing many index lookups.

### 2. Understanding `key`, `rows`, and `filtered`

These columns help explain which index MySQL selected and how much data it expects to process.

- **`key`:** The index MySQL selected. If it is `NULL`, no index is being used for the access method. That can be perfectly reasonable for a small table or a query where scanning is cheaper.
- **`rows`:** The optimizer's estimated number of rows it will examine from the chosen access path.
- **`filtered`:** The estimated percentage of those rows that will remain after applying additional conditions not already accounted for in the access method.

For example, suppose the plan estimates:

- `rows = 10,000`
- `filtered = 10`

The optimizer estimates that approximately 10% of those rows will pass the remaining filter conditions, or about 1,000 rows.

This is an estimate, not a guaranteed count. If actual data distribution differs significantly from the optimizer's statistics, the estimate can be inaccurate.

When I see unexpectedly high row estimates, I check the indexes, predicates, and data distribution. I may refresh table statistics with `ANALYZE TABLE` and evaluate whether a better composite index would improve the plan.

### 3. Interpreting the `Extra` column

`Extra` provides additional information about how MySQL executes the query.

**`Using filesort`**

MySQL needs an additional sorting operation to satisfy an `ORDER BY` or related ordering requirement.

Despite the name, filesort does not necessarily mean sorting to disk. It can happen in memory, although large sorts may spill to disk.

For example:

```sql
SELECT Id, CreatedAt
FROM Orders
WHERE RestaurantId = 10
ORDER BY CreatedAt DESC;
```

An index on `(RestaurantId, CreatedAt)` may allow MySQL to retrieve the matching rows in the required order without an additional sort, depending on the complete query and index definition.

**`Using temporary`**

MySQL uses an internal temporary table to help process the query, often for operations involving grouping, distinct values, or certain ordering patterns.

Temporary tables are not automatically a problem, but large intermediate results can increase memory usage and disk I/O. I investigate whether better indexes or a simpler query can reduce that work.

**`Using index`**

This usually indicates a covering-index access: MySQL can obtain the required columns from the index without fetching the full rows from the table.

For example, an index on `(RestaurantId, Status)` may cover:

```sql
SELECT RestaurantId, Status
FROM Orders
WHERE RestaurantId = 10;
```

The exact interpretation depends on the plan, so I examine the complete output rather than judging performance from one `Extra` value.

Other useful indicators include `Using index condition`, which indicates index condition pushdown, and `Using where`, which indicates that a condition is evaluated to filter rows.

### 4. Finding slow queries with the slow query log

My first step is to identify a real performance problem.

MySQL's slow query log records queries that exceed a configured execution-time threshold and can optionally record queries that examine many rows.

For example, in a controlled configuration, I might use:

```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
```

This sets a one-second threshold for subsequent queries, subject to session settings and server configuration. In production, I would configure the appropriate persistent settings for the deployed MySQL version and verify the log destination and permissions.

I then inspect the log or use a query-analysis tool such as `pt-query-digest` to group similar queries and rank them by total execution time, frequency, and average latency.

I don't optimize only the slowest individual query. A query taking 100 ms but executing 20,000 times may consume more total database time than one query taking 2 seconds and executing once.

I also check whether the bottleneck is genuinely SQL execution, rather than connection-pool exhaustion, network latency, lock contention, or application-level processing.

### 5. Example: optimizing a slow order query

Suppose my restaurant ERP has a large `Orders` table, and this query is slow:

```sql
SELECT Id, CreatedAt, Status, Total
FROM Orders
WHERE RestaurantId = 10
  AND Status = 'Completed'
ORDER BY CreatedAt DESC
LIMIT 20;
```

Assume the table contains millions of orders.

**Step 1: Inspect the execution plan.**

```sql
EXPLAIN
SELECT Id, CreatedAt, Status, Total
FROM Orders
WHERE RestaurantId = 10
  AND Status = 'Completed'
ORDER BY CreatedAt DESC
LIMIT 20;
```

Suppose the plan shows a full table scan, a large estimated row count, and `Using filesort`.

That suggests MySQL may be reading many rows, filtering them, and sorting the matching results before returning the latest 20.

**Step 2: Design an index around the query.**

A candidate index is:

```sql
CREATE INDEX idx_orders_restaurant_status_created
ON Orders (RestaurantId, Status, CreatedAt);
```

The two equality conditions lead the index, followed by the timestamp used for ordering. MySQL can potentially navigate to the relevant restaurant and status range, then read recent orders in index order.

Because `CreatedAt` is indexed, the engine may be able to avoid a separate sort. MySQL can traverse the index backward when that satisfies the requested ordering, so a descending index definition is not always necessary.

**Step 3: Check whether the index covers the query.**

The query also selects `Id`, `Total`, and the other columns. InnoDB secondary indexes implicitly contain the primary-key value, but `Total` is not in this index. Retrieving it may require clustered-index lookups.

I could consider extending the index to include `Total`:

```sql
CREATE INDEX idx_orders_restaurant_status_created_total
ON Orders (RestaurantId, Status, CreatedAt, Total);
```

However, I would not create both indexes automatically. The second index is wider, costs more to maintain, and may duplicate much of the first index's functionality. I would compare the real workload and decide whether the covering benefit justifies the additional cost.

**Step 4: Compare the plans and runtime.**

After creating the candidate index, I rerun `EXPLAIN` and verify whether MySQL selects the index, estimates fewer rows, and avoids unnecessary sorting.

I also measure execution time and inspect the query under representative data volumes. An index is not a successful optimization merely because it appears in the plan; it must improve the workload.

### 6. Why use `EXPLAIN ANALYZE`?

`EXPLAIN` primarily reports the optimizer's estimated execution plan. `EXPLAIN ANALYZE` actually executes the statement and reports measured execution information, including actual rows and timing for plan iterators.

For example:

```sql
EXPLAIN ANALYZE
SELECT Id, CreatedAt, Status, Total
FROM Orders
WHERE RestaurantId = 10
  AND Status = 'Completed'
ORDER BY CreatedAt DESC
LIMIT 20;
```

I compare:

- **Estimated rows vs actual rows:** Large differences may indicate inaccurate statistics or skewed data.
- **Actual execution time:** Helps identify expensive parts of the plan.
- **Rows processed at each iterator:** Reveals where excessive work occurs.
- **Loops:** A nested-loop operation executed many times may be expensive even when each individual lookup is fast.
- **Sort and filtering operations:** Helps determine whether the query performs unnecessary work.

For example, if an iterator estimates 100 rows but actually processes 100,000, I investigate why the optimizer underestimated the data volume. If a nested-loop lookup executes thousands of times, I check whether a better join index or a different query strategy would reduce repeated work.

**Important:** `EXPLAIN ANALYZE` executes the statement. I use it carefully, particularly with expensive queries and any statement that modifies data. For `SELECT` queries, it still consumes real database resources, so I avoid running heavy analysis indiscriminately on a busy production server.

### 7. What if the index does not fix the query?

An index is only one possible solution. I also investigate:

- **Non-sargable predicates:** Applying a function to an indexed column can prevent efficient index access in some cases. I may rewrite the condition as a range.
- **Implicit type conversions:** Comparing incompatible types can interfere with index usage.
- **Poor join order or missing join indexes:** Can multiply the number of rows processed.
- **Stale statistics:** Can lead to poor optimizer estimates.
- **Overly broad queries:** Returning thousands of unnecessary columns or rows increases I/O and network costs.
- **Lock contention:** A query may be slow because it waits for another transaction, not because its execution plan is inefficient.
- **Deep pagination:** Large `OFFSET` values can force MySQL to read and discard many rows. Keyset pagination can be more efficient for appropriate queries.

I also check the write workload before keeping a new index. Every additional index consumes storage and must be maintained when indexed data changes.

### 8. My optimization workflow

For a production backend, I follow this sequence:

1. Identify expensive or frequent queries using the slow query log and monitoring.
2. Reproduce the problem with representative data and parameters.
3. Run `EXPLAIN` to understand the access path, index selection, estimates, and extra operations.
4. Use `EXPLAIN ANALYZE` where safe to compare estimates with actual execution.
5. Fix the root cause through index design, query rewriting, improved pagination, or transaction changes.
6. Re-measure latency, rows processed, and resource usage.
7. Monitor the impact on inserts, updates, storage, and other queries after deployment.

**The key principle is to optimize the amount of work the database performs, not just to make an index appear in the execution plan. The slow query log tells me where to investigate, `EXPLAIN` explains the optimizer's plan, and `EXPLAIN ANALYZE` shows what actually happened.**

*Q7. Why indexes don’t get used. What makes the optimizer skip an index (functions on columns, implicit type conversion, leading wildcard LIKE, OR conditions, low selectivity, mismatched collations)? How do index statistics and ANALYZE TABLE affect plan choice, and when do you use index hints?*

An index is not guaranteed to be used just because it exists. The database optimizer estimates the cost of different execution plans and chooses the cheapest one. If it estimates that a table scan is cheaper than using an index and fetching rows, it may ignore the index.

### 1. Common reasons an index is skipped

**1. Functions or expressions on indexed columns**

Applying a function to an indexed column can prevent a normal B-tree index from being used efficiently.

```sql
-- Index exists on email
SELECT * FROM users
WHERE LOWER(email) = 'dev@example.com';
```

The database may need to evaluate `LOWER(email)` for many rows instead of directly searching the index.

**Solutions:**
- Store normalized values when appropriate.
- Create a functional index if the database supports it.
- In MySQL, consider a generated column with an index.

```sql
CREATE INDEX idx_users_email_lower
ON users ((LOWER(email)));
```

Check database-specific syntax and version support before using functional indexes.

**2. Implicit type conversion**

Comparing different data types can cause conversions that interfere with index usage.

For example, if `user_id` is an indexed `VARCHAR` column:

```sql
SELECT * FROM users WHERE user_id = 123;
```

Comparing a string column with a numeric value may cause MySQL to convert values in a way that prevents efficient index lookup.

**Solution:** Use matching data types in queries and schema definitions.

```sql
SELECT * FROM users WHERE user_id = '123';
```

**3. Leading wildcard in `LIKE`**

A standard B-tree index can efficiently support prefix matching but generally cannot efficiently seek into a string when the pattern starts with a wildcard.

```sql
-- Can often use a B-tree index
SELECT * FROM users WHERE name LIKE 'Dev%';

-- Generally cannot use a normal B-tree index for a seek
SELECT * FROM users WHERE name LIKE '%Dev%';
```

For substring or linguistic search, consider full-text indexing or a specialized search engine such as Elasticsearch, depending on the requirements.

**4. `OR` conditions**

An `OR` condition can make index usage less attractive, especially when one condition is not indexed.

```sql
SELECT * FROM users
WHERE email = 'dev@example.com'
   OR phone = '9999999999';
```

If both columns have suitable indexes, MySQL may use Index Merge. Otherwise, it may choose a table scan.

**Solutions:**
- Index the relevant columns when justified by the workload.
- Rewrite the query as `UNION` or `UNION ALL` only when the semantics are preserved and the execution plan improves.
- Compare plans rather than assuming a rewrite is faster.

**5. Low selectivity**

An index is less useful when a condition matches a large proportion of the table.

```sql
SELECT * FROM employees WHERE gender = 'M';
```

If most employees match the condition, scanning the table may be cheaper than reading index entries and fetching many rows.

An index on a low-cardinality column is not automatically useless. It can still help when a value is rare, the index covers the query, or the table is sufficiently large.

**6. Mismatched collations or character sets**

Comparisons involving different collations or character sets can introduce conversions and affect whether an index can be used efficiently.

**Solution:** Keep compared string columns compatible in character set and collation, and avoid unnecessary conversions in predicates.

### 2. Why index statistics matter

The optimizer relies on statistics to estimate how many rows a predicate will match and how expensive each plan will be.

For example, suppose a table has one million orders:

- `status = 'completed'` matches 900,000 rows.
- `status = 'cancelled'` matches 500 rows.

An index on `status` may be unattractive for the first query but very useful for the second. Statistics help the optimizer estimate this difference.

However, statistics can become stale as data changes. If the optimizer incorrectly estimates selectivity, it may choose a poor execution plan.

In MySQL, refresh table and index statistics with:

```sql
ANALYZE TABLE orders;
```

Then inspect the plan:

```sql
EXPLAIN
SELECT *
FROM orders
WHERE status = 'cancelled';
```

For deeper investigation, use `EXPLAIN ANALYZE` where supported. It executes the query and reports actual execution information alongside estimates, so use it carefully on expensive or production queries.

Statistics are not a guarantee of a better plan. If the problem persists, investigate data skew, correlated predicates, composite indexes, and whether the query retrieves too many rows for an index to be beneficial.

### 3. When should you use index hints?

Index hints influence the optimizer's choice. In MySQL, examples include:

```sql
SELECT *
FROM orders FORCE INDEX (idx_orders_status)
WHERE status = 'cancelled';
```

Common hints include:

- `USE INDEX`: restricts the indexes considered for the relevant operation.
- `FORCE INDEX`: strongly favors the specified index and makes a table scan appear more expensive.
- `IGNORE INDEX`: excludes specified indexes from consideration.

I use hints only when I have evidence that the optimizer repeatedly chooses a poor plan, and I have tested the alternative with representative data.

Before adding a hint, I check:

1. Whether the query predicate matches the index definition.
2. Whether implicit conversions or functions prevent index usage.
3. Whether statistics are current.
4. Whether a better composite or covering index is needed.
5. Whether the alternative plan actually improves execution time and resource consumption.

Hints can become harmful as data distributions, table sizes, and query patterns change. They should not replace proper schema design or accurate statistics.

### 4. Practical debugging workflow

When an expected index is not used, I follow this sequence:

1. Run `EXPLAIN` to inspect the selected access path, chosen key, estimated rows, and filtering.
2. Check the `WHERE`, `JOIN`, and `ORDER BY` expressions for functions, implicit conversions, leading wildcards, or incompatible collations.
3. Verify that the index's column order matches the query's predicates and sorting requirements.
4. Run `ANALYZE TABLE` if statistics may be stale.
5. Compare estimated and actual execution behavior using `EXPLAIN ANALYZE`, where available.
6. Test a schema or query improvement before considering a hint.

**The key principle is that indexes reduce the cost of finding or retrieving rows; they do not automatically make every query faster.** The optimizer chooses a plan based on estimated cost, and good debugging means identifying why that estimate or the available index design does not match the real workload.

*Q8. Locking and deadlocks. Explain record, gap and next-key locks, intention locks and how SELECT ... FOR UPDATE and FOR SHARE behave. How do you read SHOW ENGINE INNODB STATUS to diagnose a deadlock, and how do you reduce them (consistent lock ordering, short transactions, proper indexes)?*

In MySQL InnoDB, locks protect concurrent transactions from modifying the same data inconsistently. InnoDB uses row-level locking, but it can also lock index ranges and acquire internal table-level intention locks. Deadlocks occur when transactions wait for each other in a cycle, so neither can proceed.

### 1. Types of InnoDB locks

**Record locks**

A record lock locks an index record. For example:

```sql
SELECT *
FROM accounts
WHERE id = 10
FOR UPDATE;
```

If `id` is a unique indexed column and the query finds an existing row using an exact match, InnoDB can lock that record without locking the surrounding gap under the usual isolation-level conditions.

Record locks protect the matched index record, not necessarily an abstract row independent of its indexes.

**Gap locks**

A gap lock prevents other transactions from inserting index records into a particular gap between existing index values. It does not lock the existing records themselves.

For example, if an index contains keys `10` and `20`, a transaction locking the gap between them can prevent another transaction from inserting `15`.

Gap locks are primarily used to prevent phantom inserts under `REPEATABLE READ`. They are inhibitive: compatible gap locks can coexist, even though they prevent inserts into the protected gap.

**Next-key locks**

A next-key lock combines a record lock with a gap lock preceding that index record. It protects both the existing index record and the gap before it.

For example, a range query might lock index records and their preceding gaps to prevent another transaction from inserting a row that would change the range result.

Next-key locking is important under InnoDB's default `REPEATABLE READ` isolation level. The exact locks depend on the query plan, index structure, predicate, isolation level, and whether the search is a unique-key lookup or a range scan.

**Intention locks**

Intention locks are table-level indicators that a transaction intends to acquire row locks.

- `IS` — intention shared; the transaction intends to acquire shared row locks.
- `IX` — intention exclusive; the transaction intends to acquire exclusive row locks.
- `S` and `X` — table-level shared and exclusive locks.

These locks allow InnoDB to coordinate row locking with operations that request table-level locks. An intention lock does not mean that every row in the table is locked.

### 2. `SELECT ... FOR UPDATE` vs `SELECT ... FOR SHARE`

Both statements are locking reads and must be executed inside an explicit transaction if the locks need to remain held across subsequent application operations.

**`FOR UPDATE` — exclusive locking read**

```sql
START TRANSACTION;

SELECT balance
FROM accounts
WHERE id = 10
FOR UPDATE;

UPDATE accounts
SET balance = balance - 100
WHERE id = 10;

COMMIT;
```

This acquires exclusive locks on the records or index ranges required by the query. Other transactions generally cannot modify those locked records until the transaction finishes.

I use it when a transaction must read data and then safely modify it, such as transferring money or reserving inventory.

**`FOR SHARE` — shared locking read**

```sql
START TRANSACTION;

SELECT balance
FROM accounts
WHERE id = 10
FOR SHARE;

COMMIT;
```

This acquires shared locks on the relevant records. Multiple transactions can generally hold shared locks on the same record, but a conflicting exclusive lock cannot be acquired until those shared locks are released.

I use it when a transaction needs to ensure that a record cannot be concurrently modified while it performs a related operation.

**Important differences:**

- Both are locking reads, unlike an ordinary consistent `SELECT`.
- Locks are generally retained until `COMMIT` or `ROLLBACK`, not merely until the query finishes.
- If autocommit is enabled and the statement runs as its own transaction, the locks are released when that transaction completes.
- A range query can lock more than the rows ultimately returned because of gap or next-key locking.
- Locking behavior also depends on the isolation level and the indexes used.
- `FOR UPDATE` and `FOR SHARE` do not eliminate deadlocks; they can participate in them.

### 3. What is a deadlock?

A deadlock occurs when transactions form a circular wait.

Consider two transactions updating two accounts in opposite orders:

```sql
-- Transaction A
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

```sql
-- Transaction B
START TRANSACTION;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
COMMIT;
```

A possible sequence is:

1. Transaction A locks account `1`.
2. Transaction B locks account `2`.
3. Transaction A requests account `2` and waits for B.
4. Transaction B requests account `1` and waits for A.

Neither can continue. InnoDB detects the cycle and normally rolls back one transaction as the deadlock victim, allowing the other to proceed.

A deadlock is different from a simple lock wait: a lock wait may eventually resolve, while a deadlock requires breaking the cycle.

### 4. How I diagnose a deadlock

First, I inspect the latest deadlock report:

```sql
SHOW ENGINE INNODB STATUS;
```

I look for the `LATEST DETECTED DEADLOCK` section.

**What I examine:**

1. **Transaction details:** Identify the transactions involved, their states, and how long they have been active.
2. **Queries being executed:** Find the SQL statements that were running when the deadlock occurred.
3. **Lock details:** Look for `HOLDS THE LOCK(S)` and `WAITING FOR THIS LOCK TO BE GRANTED`. These indicate which locks each transaction owns and which lock it needs.
4. **Index and record information:** Inspect the index name, table, lock mode, and record information to determine whether the conflict involves a record, gap, or next-key lock.
5. **Victim selection:** Check which transaction InnoDB chose to roll back.

For example, if one transaction holds an exclusive lock on `PRIMARY` key `1` and waits for key `2`, while another holds key `2` and waits for key `1`, I have a clear indication of inconsistent lock ordering.

If the report shows locks on a secondary index or a broad range, I investigate the query plan and index structure. An apparently simple `UPDATE` or `DELETE` can lock many index records when its predicate lacks an appropriate index.

For recurring incidents, I also use:

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks';
```

And I can enable deadlock logging for additional evidence:

```sql
SET GLOBAL innodb_print_all_deadlocks = ON;
```

This is useful for capturing multiple incidents in the MySQL error log rather than relying only on the latest deadlock retained in the status output. I would coordinate the setting with the database operational policy.

### 5. How I reduce deadlocks

**1. Use consistent lock ordering**

If transactions need multiple accounts or inventory records, acquire locks in the same order.

For example, always lock the smaller account ID first and the larger ID second, regardless of transfer direction. This reduces circular waits.

**2. Keep transactions short**

Do not hold database locks while making HTTP requests, sending emails, calling payment gateways, or waiting for user input.

Perform external work outside the critical transaction when the business workflow permits it. For workflows spanning systems, use an outbox or saga pattern rather than holding a database transaction open across network calls.

**3. Add appropriate indexes**

Consider:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 100
  AND warehouse_id = 5;
```

A suitable composite index on `(product_id, warehouse_id)` can help InnoDB locate the relevant records efficiently. Without an appropriate index, it may scan and lock many more index records than intended.

An index does not guarantee fewer locks in every case; verify the execution plan and actual lock behavior.

**4. Avoid unnecessarily broad locking reads**

Use selective predicates and appropriate indexes. Do not use `FOR UPDATE` on an entire range if the transaction only needs to protect a specific record.

Choose the isolation level based on correctness requirements rather than lowering it indiscriminately to avoid locking.

**5. Update rows atomically where possible**

For example, instead of reading inventory into application memory and later updating it, use a conditional update:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 100
  AND quantity > 0;
```

Then verify the affected-row count. This can avoid a separate read-lock-write sequence for a straightforward inventory decrement. More complex business rules may still require a transaction.

**6. Implement retry handling**

Deadlocks can still occur even with good design. Applications should recognize MySQL error `1213` (SQLSTATE `40001` for a deadlock victim) and retry the entire transaction when safe.

Use a bounded retry count with a small randomized backoff. Do not retry only the failed statement because the transaction may already have been rolled back. Ensure external side effects are not duplicated during retries.

Also distinguish deadlocks from lock wait timeouts, which require their own error-handling considerations.

### 6. Practical production workflow

When deadlocks appear in production, I:

1. Capture the deadlock report and identify the conflicting statements and indexes.
2. Reproduce the concurrency pattern with two or more transactions.
3. Inspect `EXPLAIN` plans and verify which indexes are used.
4. Standardize lock acquisition order and shorten transaction duration.
5. Add or adjust indexes when scans or broad range locks are unnecessary.
6. Add safe, bounded transaction retries.
7. Monitor deadlock frequency and transaction latency after the change.

**The key principle:** Deadlocks are a normal possibility in concurrent transactional systems, not proof that InnoDB is broken. The goal is to reduce unnecessary contention, make lock acquisition predictable, and ensure the application recovers safely when a deadlock occurs.

*Q9. Joins and query patterns. How do nested loop and hash joins work, and why must join columns be indexed and share the same type and collation? Compare JOIN vs subquery vs EXISTS/IN, handle GROUP BY and ORDER BY ... LIMIT efficiently, and replace deep OFFSET pagination with keyset pagination.*

When optimizing SQL queries, I focus on the execution plan, join strategy, indexes, data distribution, and the number of rows processed. A query that looks simple in SQL can still be expensive if it scans large tables, sorts many rows, or processes unnecessary intermediate results.

### 1. Nested loop join vs. hash join

**Nested loop join**

A nested loop join takes rows from one input and searches for matching rows in the other input.

Conceptually:

```sql
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id;
```

The database iterates over rows from one side and looks up matching rows on the other side. With an index on `customers.id`, it can perform an indexed lookup for each outer row.

If there are \(N\) outer rows and each lookup is efficient, the cost can be roughly proportional to \(N\) index lookups. Without a useful index, repeatedly scanning the inner table can become expensive.

Nested loop joins work well when:
- One input is small or highly selective.
- The inner input has a suitable index.
- The query returns relatively few matching rows.

**Hash join**

A hash join builds an in-memory hash table from one input using the join key, then probes that hash table with rows from the other input.

Conceptually:

1. Read the build-side rows.
2. Hash their join keys into a hash table.
3. Read the probe-side rows and find matching hash entries.
4. Return matching rows.

Its expected cost is approximately \(O(N+M)\) for inputs of sizes \(N\) and \(M\), assuming reasonable hash distribution and sufficient memory.

Hash joins are useful for large equality joins when indexed lookups would be expensive. If the hash table exceeds available memory, the database may spill work to disk, increasing execution time.

**Important database-specific distinction:** PostgreSQL supports hash joins and nested loop joins. MySQL 8.0.18 and later introduced hash joins, but the optimizer's supported strategies and behavior differ by version and query shape. Do not assume every database can use every join algorithm.

### 2. Why join columns, types, and collations matter

Consider:

```sql
SELECT o.id, c.name
FROM orders o
JOIN customers c
  ON o.customer_id = c.id;
```

For efficient execution, I want `orders.customer_id` to reference the same logical identifier and compatible data type as `customers.id`.

**Indexes**

- An index on `customers.id` can make nested loop lookups efficient.
- An index on `orders.customer_id` can help when filtering orders, traversing relationships in the opposite direction, or satisfying other query patterns.
- Hash joins may not require indexes on the join columns to perform the join efficiently, because they build and probe a hash table. Indexes can still benefit filters and other parts of the query.

Therefore, the rule is not “every join column must always have an index.” The rule is to index according to access patterns and the chosen execution plan.

**Matching data types**

Suppose one join column is an integer and the other is a string containing numeric identifiers. Implicit conversions can introduce extra work and sometimes prevent efficient index use or change comparison semantics.

**Matching collations**

For string joins, incompatible collations or character sets can require conversions, complicate comparisons, and interfere with index usage. Matching compatible types, character sets, and collations avoids unnecessary conversion and ambiguity.

I also verify that the joined columns represent the same kind of value. Matching types alone does not guarantee that the data has the same meaning.

### 3. `JOIN` vs. subquery vs. `EXISTS` vs. `IN`

These constructs express different relationships. The optimizer may transform them into similar execution plans, so I choose based on semantics first and inspect performance second.

**`JOIN` — combine related rows**

```sql
SELECT c.id, c.name, o.id AS order_id
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

This returns matching customer-order pairs. A customer with five orders can appear five times.

Use a join when you need columns from both tables.

**Subquery — derive or filter using another query**

```sql
SELECT *
FROM customers
WHERE id IN (
    SELECT customer_id
    FROM orders
    WHERE status = 'completed'
);
```

Subqueries can express filtering, scalar calculations, derived tables, and other relationships. A subquery is not automatically slower than a join; the optimizer may rewrite it into a semijoin or another equivalent strategy.

**`EXISTS` — test whether a matching row exists**

```sql
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
      AND o.status = 'completed'
);
```

This expresses an existence test. It avoids multiplying customer rows when a customer has multiple matching orders, and the database may stop searching after finding a match.

An index such as `(customer_id, status)` can be useful for this correlated lookup, depending on the other query patterns.

**`IN` — test membership in a set**

```sql
SELECT *
FROM customers
WHERE id IN (10, 20, 30);
```

`IN` is concise for membership checks and can also be used with a subquery. For non-null values, `IN` and `EXISTS` can often express equivalent filtering logic.

**Watch out for `NOT IN` and `NULL`:**

```sql
SELECT *
FROM customers
WHERE id NOT IN (
    SELECT customer_id
    FROM orders
);
```

If the subquery returns a `NULL`, `NOT IN` can evaluate to unknown rather than true for otherwise nonmatching rows. This can produce surprising results.

When the intended meaning is “no matching row exists,” `NOT EXISTS` is often safer:

```sql
SELECT c.*
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

### 4. Efficient `GROUP BY`

`GROUP BY` aggregates rows into groups, but it can be expensive when the database must process a large input or sort it.

For example:

```sql
SELECT customer_id, COUNT(*) AS order_count
FROM orders
WHERE created_at >= '2026-01-01'
GROUP BY customer_id;
```

I optimize this by:

- Filtering rows as early as possible.
- Indexing selective filter columns where appropriate.
- Considering a composite index such as `(created_at, customer_id)` for the date-filtered access pattern.
- Checking whether the database uses an index-based grouping strategy, a sort, or a temporary structure.

The best index depends on the query. A range predicate on `created_at` means that an index beginning with `customer_id` may be better for some grouping workloads but less effective for the date filter. There is no universal index order.

For frequent large aggregations, a summary table or materialized view may be appropriate, provided its refresh and consistency requirements are acceptable.

### 5. Efficient `ORDER BY ... LIMIT`

Consider a page of recent orders:

```sql
SELECT id, customer_id, created_at
FROM orders
WHERE status = 'completed'
ORDER BY created_at DESC
LIMIT 20;
```

Without a suitable index, the database may scan many rows and sort the matches before returning 20.

A candidate composite index is:

```sql
CREATE INDEX idx_orders_status_created
ON orders (status, created_at);
```

This can support filtering by `status` and reading matching records in date order. MySQL can scan a B-tree index backward for a descending order when appropriate; explicit descending index definitions may be useful for certain workloads and versions.

If multiple orders can have the same timestamp, make the ordering deterministic:

```sql
SELECT id, customer_id, created_at
FROM orders
WHERE status = 'completed'
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

A candidate index is `(status, created_at, id)`, subject to the engine's index ordering rules and the rest of the workload.

The objective is to let the database find the first 20 qualifying rows directly rather than sort a large intermediate result.

### 6. Why deep `OFFSET` pagination becomes slow

Consider:

```sql
SELECT id, created_at
FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 100000;
```

The database generally has to walk past approximately 100,000 qualifying rows before returning the next 20. An index can avoid a full sort, but it does not eliminate the cost of skipping a large number of entries.

This is why deep `OFFSET` pagination becomes progressively more expensive.

**Keyset pagination** uses the last row from the previous page as a cursor.

For example, if the last row on the previous page has `created_at = '2026-10-10 12:00:00'` and `id = 500`, the next page is:

```sql
SELECT id, created_at
FROM orders
WHERE (created_at, id) < ('2026-10-10 12:00:00', 500)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

This example uses MySQL row-constructor comparison syntax. The ordering columns should be non-null for this simple pattern, and the tuple comparison must match the ordering semantics.

A corresponding index is:

```sql
CREATE INDEX idx_orders_created_id
ON orders (created_at, id);
```

For a query filtered by status, a candidate would be `(status, created_at, id)`.

**Why keyset pagination is better:**

- It seeks from the cursor instead of skipping all earlier rows.
- Performance is more stable for deep pages.
- Inserts before the cursor are less likely to shift the next page compared with offset-based pagination.
- It works well for feeds, order histories, and large datasets.

Keyset pagination does not automatically provide a consistent snapshot across all pages. Concurrent updates or deletions can still affect which records appear. If a fully consistent multi-page export is required, consider a suitable transaction snapshot or an explicit snapshot boundary.

### 7. Practical optimization workflow

When a join or query pattern is slow, I:

1. Run `EXPLAIN` and, where supported, `EXPLAIN ANALYZE`.
2. Check actual and estimated row counts, join order, join algorithm, scans, and sorting or temporary operations.
3. Verify join-column types, collations, and index definitions.
4. Reduce rows early with selective predicates and choose `EXISTS` when I only need to test existence.
5. Design composite indexes around the real combination of filtering, joining, and ordering.
6. Replace deep `OFFSET` pagination with keyset pagination when the product requirements allow it.
7. Benchmark with representative data and concurrency instead of assuming a rewrite or index is faster.

**The key principle:** Efficient SQL minimizes unnecessary rows processed, avoids expensive intermediate work, and gives the optimizer useful access paths. Query syntax matters, but the execution plan and actual workload determine performance.

*Q10. Schema design, data types and money. How do you choose between INT/BIGINT, VARCHAR/TEXT, DATETIME/TIMESTAMP (time zone and the 2038 limit), ENUM and JSON? Why use DECIMAL for money, what does utf8mb4 with utf8mb4_0900_ai_ci change, and how do generated columns and JSON indexes work?*

When designing a database schema, I choose data types based on the valid data range, precision, storage requirements, query patterns, and business rules. Good schema design prevents data corruption, reduces unnecessary storage, and makes indexing and querying more efficient.

### 1. `INT` vs. `BIGINT`

Both are integer types, but they support different ranges.

| Type | Storage | Signed range |
|---|---:|---|
| `INT` | 4 bytes | −2.1 billion to 2.1 billion approximately |
| `BIGINT` | 8 bytes | −9.22 quintillion to 9.22 quintillion approximately |

I use `INT` for bounded values such as small counters or quantities when the range is sufficient. I use `BIGINT` for high-volume identifiers, event IDs, and counters that may grow substantially.

```sql
CREATE TABLE orders (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT UNSIGNED NOT NULL,
    quantity INT UNSIGNED NOT NULL
);
```

Using `BIGINT` everywhere is not automatically better. Wider keys consume more space in indexes and can increase memory and I/O costs, especially in large tables.

I also ensure that foreign-key columns use compatible types and signedness. For example, an `INT UNSIGNED` identifier should not casually reference a `BIGINT` signed identifier.

### 2. `VARCHAR` vs. `TEXT`

Both store variable-length strings, but they serve different schema-design purposes.

**`VARCHAR(n)`**

Use it for values with a known maximum length, such as email addresses, usernames, product codes, and names.

```sql
email VARCHAR(254) NOT NULL
```

**`TEXT`**

Use it for longer content, such as descriptions, articles, comments, and other potentially large text fields.

```sql
description TEXT
```

In MySQL, `TEXT` has a maximum capacity of 65,535 bytes, whereas `VARCHAR` has a declared maximum length subject to row-size, character-set, and other limits.

The distinction is not simply that `VARCHAR` is stored inline and `TEXT` is always stored off-page; InnoDB's actual storage behavior depends on row format and record size.

For indexing, a `VARCHAR` can generally be indexed using a normal index within the applicable key-length limits. `TEXT` indexes commonly require a prefix length in MySQL:

```sql
CREATE INDEX idx_description_prefix
ON products (description(100));
```

A prefix index can help with some searches, but it does not index the complete text for all purposes. For full-text search, consider a `FULLTEXT` index where appropriate.

### 3. `DATETIME` vs. `TIMESTAMP`

Both store date and time information, but MySQL handles time zones and ranges differently.

**`DATETIME`**

- Stores a date and time without time-zone conversion.
- Supports dates from year 1000 through year 9999 in MySQL.
- Useful for local business times, such as a restaurant's scheduled opening time or an appointment recorded as a wall-clock time.

**`TIMESTAMP`**

- Represents an instant that MySQL converts between the session time zone and UTC during storage and retrieval.
- Has a much narrower range in MySQL; the traditional 4-byte form ends in January 2038.
- Useful for event timestamps and audit records when the application's time-zone handling is explicit.

For example:

```sql
CREATE TABLE audit_logs (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**The 2038 issue:** Traditional 32-bit Unix timestamps overflow in January 2038. MySQL's `TIMESTAMP` range is tied to this limitation, although exact behavior depends on the MySQL version and supported range. `DATETIME` supports a much wider range.

My usual approach is to store event instants consistently in UTC, with a documented application and database time-zone policy. For future events that depend on local civil time, such as a restaurant opening at 9 AM in New York, I also preserve the relevant IANA time-zone identifier, because UTC offsets can change due to daylight-saving rules.

### 4. `ENUM` vs. `JSON`

**`ENUM`**

Use `ENUM` for a small, stable set of predefined values.

```sql
status ENUM('pending', 'confirmed', 'cancelled')
NOT NULL DEFAULT 'pending'
```

It constrains values to the defined set and can be compact. However, changing the allowed values is a schema change, and its ordering and comparison semantics can surprise developers.

For frequently changing business states, a lookup table or a validated string column may be more maintainable.

**`JSON`**

Use `JSON` for structured, semi-variable attributes that do not fit naturally into fixed relational columns.

```sql
CREATE TABLE products (
    id BIGINT UNSIGNED PRIMARY KEY,
    name VARCHAR(150) NOT NULL,
    attributes JSON
);
```

Example value:

```json
{
  "color": "blue",
  "weight": 1.5,
  "features": ["waterproof", "lightweight"]
}
```

MySQL validates JSON documents stored in a native `JSON` column, but it does not automatically create indexes on every property. JSON is useful for flexible attributes, not a reason to put every field into one document.

If a property is frequently filtered, joined, constrained, or sorted, a normal relational column is often a better choice.

### 5. Why use `DECIMAL` for money?

I use `DECIMAL` because money requires exact decimal arithmetic. `FLOAT` and `DOUBLE` use binary floating-point representations, which cannot exactly represent many decimal fractions.

For example, a floating-point calculation can produce a result like `0.30000000000000004` instead of the expected decimal `0.30`.

```sql
CREATE TABLE invoices (
    id BIGINT UNSIGNED PRIMARY KEY,
    total DECIMAL(12, 2) NOT NULL
);
```

`DECIMAL(12,2)` supports 12 total decimal digits, including two fractional digits.

For a restaurant ERP, I might use:

- `DECIMAL(12,2)` for invoice totals and monetary amounts with two fractional digits.
- `DECIMAL(5,2)` for percentage rates where the allowed range fits.
- A suitable higher-precision `DECIMAL` for intermediate tax, discount, or unit-price calculations when needed.

I define precision based on the maximum business value and the calculation requirements rather than choosing arbitrary sizes.

**Important money-handling rules:**

1. Do not use floating-point types for authoritative monetary calculations.
2. Define the currency explicitly, such as `INR`, `USD`, or `GBP`. A numeric amount alone does not identify a currency.
3. Apply rounding according to the business and tax rules, at the correct stage of the calculation.
4. Use parameterized queries and decimal types throughout the application rather than converting money to binary floating point.
5. Consider storing integer minor units, such as paise or cents, when the currency and business rules make that model appropriate.

Not all currencies have two fractional digits, and some calculations require more precision than the final amount displayed to a customer.

### 6. What does `utf8mb4` with `utf8mb4_0900_ai_ci` change?

`utf8mb4` is MySQL's UTF-8 character set that supports Unicode code points up to four bytes, including emoji and many supplementary characters. MySQL's older `utf8` alias historically referred to the limited `utf8mb3` character set.

The collation `utf8mb4_0900_ai_ci` defines how strings are compared and sorted:

- `utf8mb4` — character encoding.
- `0900` — based on Unicode Collation Algorithm version 9.0.
- `ai` — accent-insensitive.
- `ci` — case-insensitive.

For example, comparisons under this collation generally treat `Cafe` and `café` as equivalent, and `DEV` and `dev` as equivalent.

This affects more than display order. Collation can change equality comparisons, sorting, grouping, and uniqueness enforcement.

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED PRIMARY KEY,
    username VARCHAR(100)
      CHARACTER SET utf8mb4
      COLLATE utf8mb4_0900_ai_ci
      NOT NULL
);
```

If `username` has a unique index, values considered equal under its collation cannot both be stored.

I choose a collation deliberately based on the required comparison semantics. A case-insensitive, accent-insensitive collation may be appropriate for search-oriented names, but it may be unsuitable for case-sensitive identifiers or specialized linguistic requirements.

Also, `utf8mb4_0900_ai_ci` is a MySQL 8.0-era collation and is not available in every MySQL-compatible database or older version. Verify compatibility before migrations.

### 7. Generated columns and JSON indexes

A generated column derives its value from an expression rather than requiring the application to maintain that value independently.

MySQL supports `VIRTUAL` generated columns, which are computed when needed, and `STORED` generated columns, whose results are materialized in the table.

Example:

```sql
CREATE TABLE orders (
    id BIGINT UNSIGNED PRIMARY KEY,
    details JSON,
    customer_id BIGINT UNSIGNED
      GENERATED ALWAYS AS (
        CAST(details->>'$.customerId' AS UNSIGNED)
      ) STORED,
    INDEX idx_customer_id (customer_id)
);
```

Here, `customer_id` is derived from the JSON document and indexed. Queries that filter by the generated column can use the index:

```sql
SELECT *
FROM orders
WHERE customer_id = 42;
```

This is useful when a JSON property is frequently queried but the application still benefits from keeping the original flexible document.

**Functional indexes** can also index expressions directly in supported MySQL versions:

```sql
CREATE INDEX idx_lower_email
ON users ((LOWER(email)));
```

This can help queries using the same expression, subject to MySQL's expression and optimizer rules.

For JSON specifically, a common alternative is to create an indexed generated column for the property you query frequently. MySQL can also support multi-valued indexes for certain JSON arrays, but they have specific limitations and are not a replacement for normal relational modeling.

**Important:** Indexing a JSON-derived value requires careful data typing. Validate the expected JSON shape, handle missing or invalid values appropriately, and choose the correct generated-column type. Otherwise, implicit conversions or unexpected values can undermine correctness and performance.

### 8. How I make schema decisions in practice

My process is:

1. Define the domain and valid range for each field.
2. Choose the narrowest safe type that supports foreseeable growth.
3. Use exact numeric types for financial calculations.
4. Define UTC, local-time, and time-zone policies explicitly.
5. Use relational columns for stable, frequently queried data and JSON for genuinely flexible attributes.
6. Choose character sets and collations based on comparison and uniqueness requirements.
7. Design indexes around real filters, joins, sorting, and uniqueness constraints.
8. Test migrations and query plans using realistic data volumes.

**The key principle:** A database schema should enforce the domain's correctness rules while supporting the application's access patterns. Choosing data types is not merely a storage optimization; it directly affects precision, query performance, compatibility, and long-term maintainability.

*Q11. Replication, backup and high availability. How do binary logs, row vs statement vs mixed formats, and asynchronous vs semi-synchronous replication work? What causes replication lag, how do you read from replicas safely, and what is the difference between logical (mysqldump) and physical backups, and how does point-in-time recovery work?*

In MySQL, replication, backups, and high availability solve related but different problems. Replication maintains copies of data across servers, backups allow recovery from data loss or corruption, and high availability reduces downtime when a server fails. Replication alone is not a backup because accidental deletes or corrupted data can also propagate to replicas.

### 1. How binary logs work

The MySQL binary log, or binlog, records changes made by the server that are relevant to replication and point-in-time recovery.

When a transaction commits, MySQL records the transaction's changes in the binary log, subject to the server's configuration. Replicas retrieve these events and apply them to their own data.

A simplified flow is:

1. A client executes an `INSERT`, `UPDATE`, or `DELETE`.
2. The source server modifies its data and records the transaction in the binlog.
3. A replica's I/O receiver fetches binlog events and writes them to its relay log.
4. The replica's applier processes the relay-log events and applies the changes.
5. Replication status exposes the receiver and applier positions or GTID progress.

Binary logs are also essential for point-in-time recovery because they preserve changes made after a full backup.

### 2. Statement-based vs. row-based vs. mixed replication

MySQL supports three main binary logging formats.

**Statement-based replication (`STATEMENT`)**

The binlog records the SQL statements that changed data.

```sql
UPDATE accounts
SET balance = balance - 100
WHERE id = 10;
```

The replica executes the statement.

Advantages:
- Often produces smaller logs for operations affecting many rows.
- Records the logical statement rather than every changed row.

Risks:
- Some statements are nondeterministic or depend on server-specific state.
- Differences in execution context can produce inconsistent results.

**Row-based replication (`ROW`)**

The binlog records row-level changes rather than relying on the replica to rerun the original DML statement.

Advantages:
- Generally safer for replication consistency.
- Handles many nondeterministic statements more predictably.

Trade-off:
- Can generate large binlogs when many rows change.

Row-based logging is the common default in modern MySQL deployments.

**Mixed replication (`MIXED`)**

MySQL normally uses statement-based logging and switches to row-based logging for statements that require it to preserve replication safety.

The choice depends on the workload, correctness requirements, and operational trade-offs. I generally favor row-based logging when predictable replication behavior is more important than minimizing binlog volume.

### 3. Asynchronous vs. semi-synchronous replication

**Asynchronous replication**

The source commits a transaction without waiting for a replica to confirm receipt of the transaction's binlog events.

Advantages:
- Low write latency.
- Source availability is not tightly coupled to replica responsiveness.

Risk:
- If the source fails before changes reach a replica, a newly promoted replica may be missing recently committed transactions.

**Semi-synchronous replication**

The source waits for at least one replica to acknowledge receipt of the transaction's binlog events before completing the commit, according to the configured acknowledgment policy.

This reduces the window in which a committed transaction exists only on the source.

However, acknowledgment generally means that the replica has received the relevant events, not necessarily that it has applied them and made them visible to queries.

Semi-synchronous replication can increase commit latency. If acknowledgments time out, MySQL can fall back to asynchronous behavior, depending on the configuration. Therefore, semi-synchronous replication does not automatically guarantee zero data loss.

**Important distinction:** Replication is not the same as failover. A production HA design also needs health checks, promotion logic, routing, fencing of the old primary, and a strategy for avoiding split-brain writes.

### 4. What causes replication lag?

Replication lag occurs when a replica falls behind the source. Common causes include:

- Heavy write traffic on the source.
- Slow storage, CPU constraints, or insufficient resources on the replica.
- Large transactions that take a long time to apply.
- A workload that cannot be applied efficiently in parallel.
- Long-running queries or locking conflicts on the replica.
- Network interruptions or limited bandwidth.
- DDL operations or bursts of writes that overwhelm the applier.
- Replica configuration differences and operational bottlenecks.

I diagnose lag by examining receiver and applier status, relay-log progress, transaction history, and server resource utilization.

For example:

```sql
SHOW REPLICA STATUS\G
```

Important fields vary by MySQL version, but commonly include:

- `Replica_IO_Running` and `Replica_SQL_Running` — whether the receiver and applier are running.
- `Seconds_Behind_Source` — a useful but imperfect lag indicator.
- `Last_IO_Error` and `Last_SQL_Error` — errors preventing progress.
- Retrieved and executed GTID sets or corresponding log positions — to assess replication progress.

`Seconds_Behind_Source` can be `NULL`, can be misleading during interruptions, and does not guarantee that a specific recent transaction has been applied. I treat it as one signal rather than proof of freshness.

### 5. How to read from replicas safely

Read replicas can scale read-heavy workloads, but replication is often eventually consistent. A read immediately following a write may not see that write if it goes to a lagging replica.

For example, an application may update an order's status on the primary and then immediately fetch the order from a replica. The response could show the old status.

I use several strategies:

**Read-after-write consistency**

Route the subsequent read to the primary for a defined period or until the relevant change is known to have reached the replica.

**GTID-based consistency**

With GTID-based replication, the application or routing layer can use transaction identifiers to establish whether a replica has applied the required transaction. MySQL provides `WAIT_FOR_EXECUTED_GTID_SET()` for waiting on a GTID set, subject to timeouts and operational constraints.

**Use replicas for suitable workloads**

Replicas are appropriate for reporting, analytics, catalog browsing, and other workloads that can tolerate bounded staleness. Critical decisions such as checking inventory before confirming a purchase should use a consistency strategy that cannot incorrectly accept stale data.

**Handle lag and failures explicitly**

A read-routing layer should detect unhealthy or excessively lagging replicas and route queries elsewhere when necessary. I also avoid assuming that a successful replication connection means the replica is fully caught up.

### 6. Logical vs. physical backups

**Logical backup: `mysqldump`**

A logical backup exports schema definitions and SQL statements that recreate data.

```bash
mysqldump \
  --single-transaction \
  --routines \
  --events \
  --triggers \
  --databases restaurant_erp \
  > restaurant_erp.sql
```

For InnoDB tables, `--single-transaction` provides a consistent transactional snapshot for the dump, assuming the workload and schema operations are compatible with that approach.

Advantages:
- Portable across supported environments and often useful across versions.
- Easy to inspect or restore selected objects.
- Useful for migrations and selective recovery.

Disadvantages:
- Large databases can take longer to export and restore.
- SQL parsing and reinserting rows can make recovery slow.
- Nontransactional tables require additional consistency precautions.

**Physical backup**

A physical backup copies the database's data files and related structures, often using a tool such as Percona XtraBackup or a MySQL Enterprise Backup solution.

Advantages:
- Usually faster for large databases.
- Can restore data without recreating every row through SQL statements.
- Better suited to large-scale recovery and replica provisioning.

Disadvantages:
- More sensitive to server versions, storage layout, and backup-tool compatibility.
- Requires correct handling of consistency, redo logs, and recovery preparation.
- Less convenient for restoring only a few individual rows.

I choose based on database size, recovery time objectives, recovery point objectives, portability needs, and operational tooling. A robust production strategy often combines physical backups for fast recovery with logical exports for portability or selective restoration.

### 7. How point-in-time recovery (PITR) works

Point-in-time recovery restores a backup and then replays binlog events until the desired recovery point.

Suppose a developer accidentally deletes customer records at 14:30, and the last valid state was 14:29.

The recovery process is:

1. Restore a known-good backup taken before the deletion.
2. Identify the corresponding binlog position or GTID boundary.
3. Replay binlog events from that point up to just before the destructive transaction.
4. Validate the recovered data.
5. Restore service or selectively merge the recovered records.

A simplified example using a logical backup and timestamp-based binlog replay is:

```bash
mysql -u root -p restaurant_erp < restaurant_erp.sql
```

Then replay the relevant binlog events with `mysqlbinlog`, stopping at the chosen point:

```bash
mysqlbinlog \
  --start-datetime="2026-10-15 00:00:00" \
  --stop-datetime="2026-10-15 14:29:59" \
  /var/lib/mysql/binlog.000123 \
  | mysql -u root -p
```

This is illustrative, not a complete production recovery command. A real recovery must account for the backup's exact starting position, all subsequent binlog files, time-zone assumptions, transaction boundaries, and the risk of replaying statements outside the intended database or schema. A precise binlog position or GTID boundary is generally safer than relying solely on timestamps.

Also, a backup restored into an existing database can conflict with existing objects or data. I normally recover into an isolated instance first, validate it, and then decide whether to replace the production database or selectively recover data.

### 8. Designing an effective HA and backup strategy

I define two important recovery targets:

- **RPO (Recovery Point Objective):** How much data loss is acceptable, measured in time.
- **RTO (Recovery Time Objective):** How quickly service must be restored.

For example, an RPO of five minutes means the business aims to lose no more than approximately five minutes of committed data during a disaster. It does not mean the architecture automatically achieves that target; it must be validated through testing.

My operational checklist includes:

1. Configure binlog retention long enough for recovery and replica catch-up.
2. Use a suitable replication topology and monitor replica health.
3. Automate failover carefully, with fencing to prevent two writable primaries.
4. Schedule full backups and, where appropriate, incremental backups.
5. Preserve the binlogs required for PITR, including off-host or off-site copies.
6. Encrypt backups and restrict access to them.
7. Monitor backup completion, binlog availability, replication lag, and storage capacity.
8. Perform regular restore drills and measure actual recovery time.

**The key principle:** Replication improves availability and can reduce read load, but backups protect recoverability. A reliable MySQL system needs both, plus tested failover and point-in-time recovery procedures.

*Q12. Online schema changes, partitioning and scaling. How do you add a column or index to a huge table without downtime (ALGORITHM=INPLACE/INSTANT, pt-online-schema-change, gh-ost)? When does partitioning help, and when does it not? What do connection limits, max_connections, thread pooling and ProxySQL/connection pooling solve?*

For large MySQL tables, schema changes and scaling must be planned around table size, lock duration, replication lag, application compatibility, and available resources. A DDL statement that takes seconds on a small development database can cause significant disruption on a table containing hundreds of millions of rows.

### 1. Adding columns and indexes without downtime

MySQL supports several DDL algorithms, but the correct choice depends on the operation, MySQL version, storage engine, and column definition.

**`ALGORITHM=INSTANT`**

This is generally the least disruptive option when supported. MySQL changes metadata without rebuilding the entire table for eligible operations.

For example, on a supported MySQL 8.0 version and for an eligible column addition:

```sql
ALTER TABLE orders
ADD COLUMN source VARCHAR(30) NULL,
ALGORITHM=INSTANT;
```

The exact eligibility rules depend on the MySQL version and table definition. Some operations, such as certain column reordering or other structural changes, cannot use this algorithm.

Instant DDL still needs brief metadata locks. It can also be delayed by long-running transactions that hold metadata locks, so I inspect transaction activity before running it.

**`ALGORITHM=INPLACE`**

This allows certain operations to modify a table without copying the entire table into a new one. However, *in-place does not mean lock-free*.

For example, creating an index on InnoDB commonly uses an in-place algorithm:

```sql
ALTER TABLE orders
ADD INDEX idx_orders_status (status),
ALGORITHM=INPLACE,
LOCK=NONE;
```

Where supported, `LOCK=NONE` permits concurrent reads and writes during the main operation. The DDL can still require brief exclusive metadata locks at the beginning or end.

If the requested algorithm or lock level is unsupported, MySQL may reject the statement rather than perform the requested operation. That is useful in production because it prevents an unexpected fallback to a more disruptive method.

**`ALGORITHM=COPY`**

This typically rebuilds the table by copying data into a new structure. It can consume substantial disk space, I/O, and time, and may restrict concurrent writes. I avoid it on large production tables unless the operation and maintenance plan justify the impact.

### 2. Using `pt-online-schema-change` and `gh-ost`

When native online DDL is insufficient, online schema-change tools can reduce application downtime by copying data into a new table while allowing the original table to remain in use.

**`pt-online-schema-change`**

The Percona Toolkit tool typically:

1. Creates a shadow table with the new schema.
2. Copies rows in chunks.
3. Uses triggers on the original table to propagate concurrent changes to the shadow table.
4. Synchronizes the tables and swaps them when ready.

Advantages:
- Works with many MySQL environments.
- Can perform schema changes that would otherwise require a long table copy.

Risks:
- Triggers can add overhead to writes.
- Foreign keys require special handling.
- The final table swap requires metadata locking.
- Long transactions can block the swap, and replication lag can increase during copying.

**`gh-ost`**

GitHub's online schema migration tool uses binary-log events to propagate ongoing changes to a ghost table rather than relying on triggers for that purpose.

Its general process is to create a ghost table, copy existing rows, apply captured changes, and perform a controlled cutover.

Advantages:
- Avoids adding migration triggers to the source table.
- Offers controls for throttling and postponing cutover.

Limitations:
- Requires suitable binlog configuration and permissions.
- Compatibility depends on the table, foreign keys, replication topology, and tool version.
- The cutover still requires metadata locking.

Neither tool guarantees zero downtime. Both can create substantial I/O, consume disk space, and increase replication lag.

### 3. My production migration strategy

For a large table, I follow this sequence:

1. **Check compatibility:** Confirm the MySQL version, storage engine, table definition, foreign keys, and available DDL algorithm.
2. **Inspect metadata locks:** Check for long-running transactions and sessions that could block DDL.
3. **Estimate impact:** Evaluate table size, disk headroom, write volume, replica capacity, and expected index-build time.
4. **Prefer native online DDL:** Use `INSTANT` or an appropriate in-place operation where supported.
5. **Use a migration tool if needed:** Select `pt-online-schema-change` or `gh-ost` after checking its compatibility with the environment.
6. **Throttle and monitor:** Track CPU, I/O, disk space, application latency, errors, and replica lag.
7. **Plan cutover and recovery:** Verify the new structure, keep the migration reversible where possible, and ensure application code remains compatible.

I also use an **expand-and-contract deployment** for application changes:

- Expand: Add the new column or table without breaking the old application.
- Deploy: Make the application compatible with both old and new structures.
- Backfill: Populate existing rows in manageable batches.
- Switch: Move reads and writes to the new structure.
- Contract: Remove obsolete schema only after all old application versions are gone.

This prevents a schema deployment from requiring every application instance to change simultaneously.

### 4. When does partitioning help?

Partitioning divides a logical table into physical partitions managed by the database. In MySQL, partitioning is commonly based on `RANGE`, `LIST`, `HASH`, or `KEY`.

**Range partitioning example**

A large orders table might be partitioned by month using `created_at`.

```sql
CREATE TABLE orders (
    id BIGINT UNSIGNED NOT NULL,
    created_at DATETIME NOT NULL,
    customer_id BIGINT UNSIGNED NOT NULL,
    total DECIMAL(12,2) NOT NULL,
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE COLUMNS (created_at) (
    PARTITION p202610 VALUES LESS THAN ('2026-11-01'),
    PARTITION p202611 VALUES LESS THAN ('2026-12-01'),
    PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
```

This illustrates the partitioning structure rather than a complete production partition-management plan. In MySQL, every unique key on a partitioned table must include the columns used in the partitioning expression. That is why the example's primary key includes `created_at`.

**When partitioning helps**

- Queries frequently filter on the partitioning key, allowing partition pruning.
- Data is naturally organized by date or another stable range.
- Old data needs to be removed efficiently, such as dropping an expired time partition instead of deleting millions of rows individually.
- Maintenance and archival operations benefit from managing subsets of a table.

**When partitioning does not help**

- Queries do not filter on the partitioning key and must scan most partitions.
- The bottleneck is an inefficient join, missing index, or excessive connection count.
- The table is not large enough for partitioning to justify the operational complexity.
- The workload needs frequent global uniqueness checks that conflict with MySQL's partitioning restrictions.

Partitioning is not sharding. All partitions normally remain within one MySQL server, so partitioning does not automatically distribute writes across multiple machines or increase total server capacity.

I partition based on demonstrated query and retention patterns, not merely because a table has many rows.

### 5. `max_connections` and connection limits

MySQL limits the number of simultaneous client connections using `max_connections`.

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
```

Increasing `max_connections` can allow more clients to connect, but each active connection consumes resources. Under high concurrency, more connections can increase memory usage, context switching, lock contention, and latency.

The important question is not simply how many connections the server can accept, but how many it can process efficiently.

For example, if an application runs many API instances and each maintains a large connection pool, the aggregate connection count can overwhelm MySQL even when each application instance appears reasonably configured.

I size pools across the entire deployment, leaving capacity for background jobs, administrative sessions, and failover scenarios.

### 6. Connection pooling and thread pooling

These solve related but different problems.

**Connection pooling**

An application keeps a bounded collection of reusable database connections instead of opening a new TCP connection for every query.

Benefits:
- Reduces connection establishment and authentication overhead.
- Limits the number of concurrent database sessions.
- Reuses established connections across requests.

However, a pool that is too large can overwhelm the database. A pool that is too small can cause requests to wait for a connection. I measure pool wait time, query latency, connection utilization, and database capacity when tuning it.

For example, in a .NET application, a database connection should normally be opened for the operation and promptly disposed. With a properly configured ADO.NET provider, disposal typically returns the connection to the pool rather than closing the physical connection.

**Thread pooling**

Traditional MySQL connection handling commonly associates client connections with server threads. A thread pool can reduce the overhead of managing large numbers of active connections by reusing a bounded number of worker threads.

Thread pooling is intended to help workloads with high concurrency and many connections. It does not eliminate the cost of queries, row locks, I/O, or excessive concurrent work. Availability and implementation depend on the MySQL edition or compatible server.

### 7. What does ProxySQL solve?

ProxySQL is a database proxy that sits between applications and MySQL servers. It can provide:

- Connection multiplexing, allowing many client connections to share fewer backend connections when safe.
- Query routing, including routing selected reads to replicas.
- Connection management and pooling.
- Query rules, traffic controls, and monitoring.
- Backend health checks and support for topology-aware routing.

For example, a system may have many application instances but only need a bounded number of active MySQL backend connections. ProxySQL can help manage that relationship.

However, multiplexing is not always possible. Session state, open transactions, temporary tables, prepared statements, and other session-specific behavior can pin a client to a backend connection.

Read/write splitting also requires care: routing a read to a replica immediately after a write can return stale data. A proxy does not automatically guarantee read-after-write consistency or eliminate the need for failover orchestration.

### 8. How I approach scaling a MySQL application

My order of operations is:

1. Optimize slow queries and indexes.
2. Bound application connection pools and measure database saturation.
3. Use caching or read replicas where the consistency model permits.
4. Apply online schema changes and controlled backfills for large tables.
5. Partition only when pruning or data-retention benefits justify it.
6. Introduce ProxySQL or another suitable proxy when connection management and routing need a central control layer.
7. Consider sharding only when a single database server remains the bottleneck and the added operational complexity is justified.

**The key principle:** Online DDL reduces schema-change disruption, partitioning improves specific access and retention patterns, and connection pooling or proxies control database concurrency. None of these substitutes for understanding the actual bottleneck, and every production change needs monitoring, compatibility checks, and a recovery plan.

*Q13. Window functions and CTEs. How do ROW_NUMBER, RANK, DENSE_RANK, LAG/LEAD and running totals with SUM() OVER (PARTITION BY ... ORDER BY ...) work? Solve “top N per group” and “latest row per customer” with window functions vs correlated subqueries. When are recursive CTEs useful (hierarchies, date series), and when does a CTE get materialized vs merged?*

Window functions perform calculations across related rows without collapsing those rows into a single result, unlike ordinary aggregate queries using `GROUP BY`. CTEs, or Common Table Expressions, let me define a named query that can be referenced by the main statement, making complex SQL easier to structure and maintain.

### 1. `ROW_NUMBER()` vs. `RANK()` vs. `DENSE_RANK()`

All three functions assign rankings within a result set, but they handle ties differently.

Suppose we have these employee salaries:

| Employee | Salary |
|---|---:|
| A | 100,000 |
| B | 90,000 |
| C | 90,000 |
| D | 80,000 |

Query:

```sql
SELECT
    employee_name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK() OVER (ORDER BY salary DESC) AS salary_rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;
```

The results are:

| Employee | `ROW_NUMBER` | `RANK` | `DENSE_RANK` |
|---|---:|---:|---:|
| A | 1 | 1 | 1 |
| B | 2 | 2 | 2 |
| C | 3 | 2 | 2 |
| D | 4 | 4 | 3 |

- **`ROW_NUMBER()`** assigns a unique sequential number to each row, even when values tie. Add a unique tie-breaker to the `ORDER BY` if the selection must be deterministic.
- **`RANK()`** gives tied rows the same rank and leaves gaps after ties.
- **`DENSE_RANK()`** gives tied rows the same rank but does not leave gaps.

I choose the function according to the requirement. For exactly three rows per group, I use `ROW_NUMBER()`. For the top three distinct salary levels, I use `DENSE_RANK()`.

### 2. `LAG()` and `LEAD()`

These functions access values from a preceding or following row within the ordered window.

- `LAG(column)` reads a value from a previous row.
- `LEAD(column)` reads a value from a subsequent row.

For example, to compare each order with the previous order for the same customer:

```sql
SELECT
    customer_id,
    order_date,
    total,
    LAG(total) OVER (
        PARTITION BY customer_id
        ORDER BY order_date, id
    ) AS previous_order_total,
    LEAD(total) OVER (
        PARTITION BY customer_id
        ORDER BY order_date, id
    ) AS next_order_total
FROM orders;
```

The `id` tie-breaker makes the ordering deterministic when multiple orders have the same timestamp.

For the first row in a partition, `LAG()` returns `NULL` by default because there is no preceding row. You can supply a default value:

```sql
LAG(total, 1, 0) OVER (
    PARTITION BY customer_id
    ORDER BY order_date, id
)
```

A practical use case is calculating the change from the previous order:

```sql
total - LAG(total) OVER (
    PARTITION BY customer_id
    ORDER BY order_date, id
) AS change_from_previous
```

This is useful for trend analysis, sales comparisons, and detecting changes in customer behavior.

### 3. Running totals with `SUM() OVER`

A running total calculates an accumulated sum while preserving each individual row.

```sql
SELECT
    customer_id,
    order_date,
    total,
    SUM(total) OVER (
        PARTITION BY customer_id
        ORDER BY order_date, id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM orders;
```

Here:

- `PARTITION BY customer_id` starts a separate running total for each customer.
- `ORDER BY order_date, id` defines the accumulation order.
- `UNBOUNDED PRECEDING` starts at the first row in the partition.
- `CURRENT ROW` ends the frame at the current row.

I explicitly specify a `ROWS` frame for row-by-row accumulation. Without an explicit frame, the default can include peer rows with the same ordering value, depending on the database and window definition.

For example, if two orders share a timestamp, a default `RANGE` frame may assign both rows the same cumulative total. A `ROWS` frame with deterministic ordering avoids that ambiguity.

### 4. Solving top N per group

Suppose I need the three highest-value orders for each customer.

```sql
WITH ranked_orders AS (
    SELECT
        id,
        customer_id,
        total,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY total DESC, id DESC
        ) AS rn
    FROM orders
)
SELECT id, customer_id, total
FROM ranked_orders
WHERE rn <= 3;
```

The CTE calculates the row number for every order within its customer's partition. The outer query filters to the first three.

If the requirement is the top three distinct order values, including all ties, I can use `DENSE_RANK()` instead:

```sql
WITH ranked_orders AS (
    SELECT
        id,
        customer_id,
        total,
        DENSE_RANK() OVER (
            PARTITION BY customer_id
            ORDER BY total DESC
        ) AS rnk
    FROM orders
)
SELECT id, customer_id, total
FROM ranked_orders
WHERE rnk <= 3;
```

The distinction matters: `ROW_NUMBER() <= 3` returns at most three rows per customer, while `DENSE_RANK() <= 3` can return more than three rows when values tie.

### 5. Finding the latest row per customer

A common requirement is retrieving each customer's most recent order.

```sql
WITH ranked_orders AS (
    SELECT
        id,
        customer_id,
        order_date,
        total,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY order_date DESC, id DESC
        ) AS rn
    FROM orders
)
SELECT id, customer_id, order_date, total
FROM ranked_orders
WHERE rn = 1;
```

This returns one latest order per customer. The `id DESC` tie-breaker determines which row wins if timestamps match.

If customers without orders must also appear, use a `LEFT JOIN` from `customers` to this ranked result and restrict the match to `rn = 1` in the join condition.

### 6. Window functions vs. correlated subqueries

The same “latest order per customer” problem can be solved with a correlated subquery:

```sql
SELECT o.*
FROM orders o
WHERE o.id = (
    SELECT o2.id
    FROM orders o2
    WHERE o2.customer_id = o.customer_id
    ORDER BY o2.order_date DESC, o2.id DESC
    LIMIT 1
);
```

This is illustrative MySQL syntax; it assumes `id` uniquely identifies each order and that the ordering and null-handling rules match the business requirement.

**Window function advantages:**
- Clearly expresses ranking within groups.
- Easily extends to top N, running totals, and previous/next-row comparisons.
- Often easier to maintain when the query needs multiple window calculations.

**Correlated subquery advantages:**
- Can be concise for a single lookup per outer row.
- May perform well with a suitable index and an optimizer that efficiently handles the lookup.

A useful index candidate for the latest-order query is:

```sql
CREATE INDEX idx_orders_customer_date
ON orders (customer_id, order_date, id);
```

The best index definition and scan direction depend on the database version and workload. A window function is not automatically faster than a correlated subquery; inspect `EXPLAIN` and benchmark against representative data.

### 7. What is a CTE?

A CTE is a named query defined using `WITH`. It improves readability by breaking a complex statement into logical stages.

```sql
WITH customer_totals AS (
    SELECT customer_id, SUM(total) AS lifetime_value
    FROM orders
    GROUP BY customer_id
)
SELECT customer_id, lifetime_value
FROM customer_totals
WHERE lifetime_value > 10000;
```

Here, `customer_totals` represents the aggregation that the outer query consumes.

CTEs can make a query easier to reason about, but they do not automatically make it faster. Their performance depends on how the optimizer handles them.

### 8. When are CTEs merged or materialized?

**Merging**

The optimizer incorporates a CTE or derived table into the surrounding query, much like expanding a view. This can allow predicates to be pushed down and avoid storing an intermediate result.

**Materialization**

The database computes the CTE into an intermediate result that the outer query reads. This can be useful when the intermediate result is expensive to compute or reused, but it can consume memory or temporary disk space.

In MySQL 8.0, nonrecursive CTEs and derived tables may be merged or materialized depending on the query structure and optimizer decisions. Constructs such as aggregation, window functions, `DISTINCT`, and `LIMIT` can prevent merging in relevant contexts.

A CTE referenced multiple times may be materialized once and reused, depending on the plan. Materialization is not inherently bad, and merging is not inherently better: the best strategy depends on row counts, predicate selectivity, and reuse.

I inspect the execution plan instead of assuming that a CTE creates a temporary table or that it is always inlined.

### 9. Recursive CTEs: hierarchies and date series

A recursive CTE refers to its own result. It typically has two parts:

1. An **anchor query** that provides the starting rows.
2. A **recursive query** that derives subsequent rows from the previous iteration.

**Example: organizational hierarchy**

Suppose each employee has a `manager_id`.

```sql
WITH RECURSIVE employee_tree AS (
    SELECT id, manager_id, name, 0 AS depth
    FROM employees
    WHERE id = 1

    UNION ALL

    SELECT e.id, e.manager_id, e.name, t.depth + 1
    FROM employees e
    JOIN employee_tree t
      ON e.manager_id = t.id
)
SELECT *
FROM employee_tree;
```

This starts at employee `1` and recursively finds their direct and indirect reports.

An index on `employees.manager_id` helps locate the next level efficiently. In production, hierarchical data also needs protection against cycles and excessive recursion depth.

**Example: generating a date series**

A recursive CTE can generate dates without a dedicated calendar table:

```sql
WITH RECURSIVE dates AS (
    SELECT DATE('2026-10-01') AS day

    UNION ALL

    SELECT day + INTERVAL 1 DAY
    FROM dates
    WHERE day < DATE('2026-10-31')
)
SELECT day
FROM dates;
```

This produces dates from October 1 through October 31, inclusive.

Recursive CTEs are useful for modest hierarchies, date ranges, and graph-like traversal. For repeated reporting over large date ranges, a permanent calendar table may be simpler and more efficient. MySQL also imposes a configurable recursion-depth limit, so large sequences require appropriate planning.

### 10. Practical optimization checklist

When using window functions and CTEs, I:

1. Define partition and ordering semantics precisely, including tie-breaking and `NULL` handling.
2. Choose `ROW_NUMBER`, `RANK`, or `DENSE_RANK` according to whether the requirement is rows or distinct rank levels.
3. Specify an explicit window frame for running totals when row-by-row behavior is required.
4. Filter input rows as early as correctness allows.
5. Index the partition, join, and ordering columns according to the actual query plan.
6. Compare window functions with correlated subqueries when performance matters.
7. Inspect whether CTEs are merged or materialized using `EXPLAIN`.
8. Bound recursive queries and prevent cycles in hierarchical data.

**The key principle:** Window functions solve row-relative calculations without collapsing the result set, while CTEs structure complex SQL into readable stages. Correct ordering, appropriate indexes, and the optimizer's execution plan determine whether those queries are also efficient.

*Q14. Stored procedures, functions, triggers and events. What are the real trade-offs of putting logic in the database (performance, versioning, testing, debugging, portability)? Where do triggers cause hidden problems (cascading writes, replication, bulk loads), and how do DELIMITER, DECLARE ... HANDLER and SIGNAL work for error handling?*

I treat stored procedures, functions, triggers, and events as database-side programming tools. They can improve performance and consistency in the right situations, but putting too much business logic in the database makes testing, debugging, versioning, and portability harder.

## 1. Stored Procedures vs Functions vs Triggers vs Events

| Feature | Purpose | Example |
|---|---|---|
| **Stored procedure** | Executes a sequence of SQL operations; can accept input/output parameters | Process a payment or transfer funds |
| **Function** | Computes and returns a value, often usable inside SQL expressions | Calculate tax or normalize a value |
| **Trigger** | Runs automatically when a specified table event occurs | Record an audit entry after an update |
| **Event** | Runs a scheduled database task | Delete expired sessions nightly |

For example, a stored procedure can perform several related database operations within a transaction. A function is better suited to a computation that returns a value. A trigger is appropriate when a database-level side effect must happen automatically, while an event is useful for database-local scheduled maintenance.

In MySQL, the Event Scheduler must be enabled for scheduled events to run.

## 2. What are the trade-offs of putting logic in the database?

### Advantages

- **Reduced network round trips:** Multiple SQL operations can execute in one procedure call rather than requiring several application-to-database requests.
- **Transaction consistency:** Related changes can be grouped into one database transaction.
- **Centralized enforcement:** Rules implemented at the database level can protect data even when multiple applications write to the same tables.
- **Reduced data transfer:** Computation and filtering can happen close to the data instead of returning large datasets to the application.
- **Controlled access:** Applications can sometimes receive permission to execute approved procedures without receiving unrestricted access to underlying tables.

### Disadvantages

- **Versioning and deployment:** Database routines need migrations, version control, review, and coordinated deployment with application code.
- **Testing:** Database-specific logic requires a real database for reliable integration testing. Mocking SQL behavior does not adequately test triggers, transaction semantics, or stored procedures.
- **Debugging:** Application debuggers and tracing tools often provide a better development experience than debugging database routines.
- **Portability:** MySQL procedures, PostgreSQL PL/pgSQL functions, and SQL Server T-SQL procedures differ in syntax and behavior.
- **Operational complexity:** A slow procedure or unexpected trigger can affect many application requests and be difficult to trace back to its source.
- **Separation of concerns:** Business rules become harder to understand when split between application services, stored procedures, and triggers.

**My approach:** I keep most business orchestration in the application service layer and use database constraints, transactions, indexes, and carefully selected stored procedures for data-intensive or database-specific operations. I use triggers sparingly because their automatic execution can hide side effects.

## 3. Where do triggers cause hidden problems?

Triggers execute implicitly. An application may update one row but cause several additional writes without knowing it.

### A. Cascading writes

Suppose an `orders` trigger updates `inventory`, which activates another trigger that writes to an audit table. One apparently simple update now has multiple side effects.

This creates problems such as:

- Unexpected latency and lock contention.
- Complex transaction behavior.
- Difficult debugging when a write fails.
- Recursive or cyclic trigger logic, where supported or indirectly created.

I document trigger side effects and avoid circular dependencies between tables and triggers.

### B. Replication

Trigger behavior depends on the database and replication configuration.

In MySQL, with statement-based replication, the statement is replayed on the replica and triggers may execute there. With row-based replication, the primary's resulting row changes are generally replicated, and replica triggers are not independently fired for those replicated row events.

This matters because assuming that every trigger runs identically on both primary and replica can cause incorrect audit records or duplicate side effects. I verify the replication format and test the actual deployment topology.

### C. Bulk imports and migrations

Bulk loads can cause triggers to run for large numbers of rows, increasing execution time, lock duration, and log volume. They can also reject otherwise valid imports or generate unexpected audit records.

Before a bulk operation, I check which triggers will execute, estimate the workload, and use a controlled import strategy. I do not assume that bulk operations automatically bypass triggers.

### D. External side effects

I avoid sending emails, calling APIs, or publishing messages directly from a trigger. A database transaction might roll back after the external action has already succeeded, leaving the external system inconsistent.

For reliable event publishing, I prefer the **transactional outbox pattern**: write the business change and an outbox record in the same transaction, then publish the event from an application worker.

## 4. What does `DELIMITER` do in MySQL?

The MySQL command-line client normally treats `;` as the end of a statement. But a procedure or trigger contains multiple SQL statements, each of which also ends with `;`.

`DELIMITER` changes the client-side statement terminator temporarily so the entire routine can be submitted as one statement. It is a client command, not part of the stored procedure's SQL body.

Example:

```sql
DELIMITER //

CREATE PROCEDURE GetOrderCount(
    IN p_customer_id INT,
    OUT p_order_count INT
)
BEGIN
    SELECT COUNT(*)
    INTO p_order_count
    FROM orders
    WHERE customer_id = p_customer_id;
END //

DELIMITER ;
```

Here, `//` terminates the complete `CREATE PROCEDURE` statement. The semicolons inside `BEGIN ... END` remain part of the procedure body.

**Interview nuance:** `DELIMITER` is understood by clients such as the MySQL command-line client. It is not a SQL statement that should be sent to the MySQL server through a normal application database driver. Migration tools and drivers typically handle routine definitions differently.

## 5. How does `DECLARE ... HANDLER` work?

In MySQL stored programs, a handler specifies what to do when a condition occurs, such as an SQL exception, a warning, or `NOT FOUND`.

The basic syntax is:

```sql
DECLARE handler_action HANDLER
FOR condition_value
statement;
```

For example, an exit handler can roll back a transaction and propagate the error:

```sql
DELIMITER //

CREATE PROCEDURE CreateOrder(
    IN p_customer_id INT
)
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;

    INSERT INTO orders (customer_id, status)
    VALUES (p_customer_id, 'pending');

    COMMIT;
END //

DELIMITER ;
```

The flow is:

1. Start a transaction.
2. Insert the order.
3. If an SQL exception occurs, execute the `EXIT` handler.
4. Roll back the transaction.
5. `RESIGNAL` propagates the original error to the caller.

An `EXIT` handler leaves the enclosing compound block after its handler executes. A `CONTINUE` handler handles the condition and continues execution after the statement that raised it.

Other common conditions include:

- `SQLEXCEPTION`: SQL exceptions.
- `SQLWARNING`: SQL warnings.
- `NOT FOUND`: No row was found, commonly encountered in cursor processing or `SELECT ... INTO`.

**Important:** Do not catch an error and silently continue unless that behavior is intentional. Also, transaction ownership matters: a procedure that starts and rolls back a transaction can interfere with an application-managed transaction. Define clearly whether the application or procedure controls the transaction boundary.

## 6. How does `SIGNAL` work?

`SIGNAL` lets a stored program raise an error deliberately, rather than relying only on errors produced by SQL statements.

For example:

```sql
DELIMITER //

CREATE PROCEDURE TransferFunds(
    IN p_amount DECIMAL(12, 2)
)
BEGIN
    IF p_amount <= 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Transfer amount must be positive';
    END IF;

    -- Transfer logic goes here.
END //

DELIMITER ;
```

`SQLSTATE '45000'` is commonly used for a user-defined exception. The application receives an error instead of continuing with an invalid amount.

I use `SIGNAL` to enforce database-side preconditions when appropriate. However, I also validate user input in the application for better feedback; database validation remains an additional integrity safeguard.

## 7. How would I handle errors in production?

My approach is to:

1. **Validate inputs** before performing writes.
2. **Use transactions** for logically related database changes.
3. **Declare appropriate handlers** for errors that need cleanup or special treatment.
4. **Roll back and propagate unexpected failures** rather than swallowing them.
5. **Log context in the application**, including the procedure name, request or correlation ID, and relevant business identifiers—without logging sensitive data.
6. **Test failure paths**, including constraint violations, duplicate keys, deadlocks, and partial-operation failures.
7. **Version routines through migrations** and deploy compatible application and database changes.

I also distinguish between expected business failures, such as an insufficient balance, and unexpected infrastructure or SQL failures. Expected failures can use a deliberate error contract; unexpected failures should retain their diagnostic information.

## 8. What is my practical decision rule?

- **Constraints and foreign keys:** Enforce fundamental data integrity.
- **Stored procedures:** Use when multi-step database operations or data-heavy processing benefit from execution close to the data.
- **Functions:** Use for focused calculations or transformations, subject to the database's restrictions on functions.
- **Triggers:** Use selectively for unavoidable database-level automation, such as certain audit requirements.
- **Events:** Use for database-local scheduled maintenance, not as a replacement for a robust distributed job scheduler.

My guiding principle is that database-side code should be intentional, observable, tested, and version-controlled. A stored procedure is not automatically faster than application code, and a trigger is not automatically safer just because it executes close to the data. The right choice depends on transaction boundaries, workload, operational tooling, and who owns the business rule.

*Q15. Foreign keys, constraints and referential integrity. What do ON DELETE CASCADE, SET NULL and RESTRICT do, and what are their locking and performance side effects? How do CHECK constraints (enforced from 8.0.16), UNIQUE with NULLs, and generated columns protect data, and why do many large systems skip foreign keys and how do they compensate?*

I use database constraints to enforce data integrity as close to the data as possible. Application validation improves user experience, but it cannot guarantee consistency when multiple services, background jobs, scripts, or concurrent requests modify the same database.

## 1. What are foreign keys and referential integrity?

A **foreign key** ensures that a value in a child table references an existing row in a parent table, or is `NULL` if the relationship permits it.

For example, an order should not reference a customer who does not exist.

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    amount DECIMAL(12, 2) NOT NULL,

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id)
        REFERENCES customers(id)
);
```

Now, inserting an order with a nonexistent `customer_id` fails. Deleting a referenced customer also fails under the default `NO ACTION` behavior in MySQL, which is effectively `RESTRICT`.

<LearningViz type_id="SQL_PRIMARY_FOREIGN_KEY_CONSTRAINTS" locale_override="en-US" />

Foreign keys protect against orphaned records, but they do not replace authorization, business validation, or transaction design.

## 2. `ON DELETE CASCADE`, `SET NULL`, and `RESTRICT`

These options determine what happens when an application deletes a parent row that is referenced by child rows.

| Action | Behavior | Typical use |
|---|---|---|
| `CASCADE` | Automatically deletes referencing child rows | Order items when an order is permanently deleted |
| `SET NULL` | Sets the child foreign-key column to `NULL` | Optional association to a deleted user |
| `RESTRICT` | Rejects the parent deletion while references exist | Prevent deleting a customer with existing orders |
| `NO ACTION` | In MySQL, effectively the same as `RESTRICT` | Default-style protection against invalid references |

### Example

```sql
CREATE TABLE order_items (
    id BIGINT PRIMARY KEY,
    order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,

    CONSTRAINT fk_item_order
        FOREIGN KEY (order_id)
        REFERENCES orders(id)
        ON DELETE CASCADE
);
```

Deleting an order automatically deletes its associated order items.

For `SET NULL`, the child column must allow nulls:

```sql
FOREIGN KEY (user_id)
REFERENCES users(id)
ON DELETE SET NULL
```

If `user_id` is declared `NOT NULL`, this design is incompatible with the intended action.

### What about `ON UPDATE`?

The same concepts apply to changes to referenced parent keys:

- `ON UPDATE CASCADE` propagates the new key to child rows.
- `ON UPDATE SET NULL` clears child references, where permitted.
- `ON UPDATE RESTRICT` prevents the referenced key from changing while references exist.

In practice, primary keys are usually immutable, so `ON UPDATE CASCADE` is less commonly needed.

## 3. What are the locking and performance side effects?

Foreign keys are not free. In InnoDB, enforcing them can introduce additional index lookups and locks.

### A. Foreign-key checks on inserts and updates

When inserting a child row, InnoDB checks that the referenced parent exists. It takes appropriate locks on the referenced record to protect the integrity check.

Under concurrency, these checks can interact with other transactions and contribute to lock waits or deadlocks.

### B. Cascading deletes

A parent deletion with `ON DELETE CASCADE` can trigger many child-row deletions.

For example, deleting one customer might cascade through thousands of orders and associated items if the schema has that relationship chain.

Potential consequences include:

- More rows locked and more undo/redo logging.
- Longer-running transactions.
- Increased replication lag from large write transactions.
- Deadlocks when concurrent operations acquire locks in different orders.
- Larger rollback costs if the transaction fails.

A cascade is part of the same transactional operation in InnoDB; it is not an asynchronous background cleanup job.

### C. Index requirements

InnoDB requires suitable indexes for foreign-key enforcement. The referencing child columns need an index whose leading columns match the foreign-key columns; InnoDB can create an index automatically when needed.

I still design and name indexes deliberately rather than relying blindly on automatic index creation. An existing composite index may already satisfy the requirement.

### D. Practical decision

I use cascading deletes for tightly owned dependent records, such as line items belonging to an order that is being permanently removed. I generally avoid cascading deletion across important business entities such as customers, invoices, and payments.

For large datasets, I assess the number of affected rows and lock duration before executing destructive operations.

## 4. How do `CHECK` constraints work?

A `CHECK` constraint validates a condition for each inserted or updated row.

For example:

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    price DECIMAL(12, 2) NOT NULL,
    stock INT NOT NULL,

    CONSTRAINT chk_price_nonnegative
        CHECK (price >= 0),

    CONSTRAINT chk_stock_nonnegative
        CHECK (stock >= 0)
);
```

This prevents negative prices and stock quantities even if an application bug attempts to write them.

**MySQL version detail:** Before MySQL 8.0.16, `CHECK` constraints were parsed but not enforced. From MySQL 8.0.16 onward, they are enforced, subject to the constraint definition and database settings.

Important nuances:

- A `CHECK` expression that evaluates to `FALSE` rejects the row.
- An expression that evaluates to `UNKNOWN` because of `NULL` generally passes the check.
- Use `NOT NULL` as well when a value must be present.
- `CHECK` constraints are appropriate for row-level conditions, not for enforcing arbitrary conditions involving other rows or tables.

For example, `CHECK (price >= 0)` does not replace a transaction that verifies sufficient inventory and then decrements stock safely.

## 5. How does `UNIQUE` behave with `NULL` values?

In MySQL, a unique index generally allows multiple `NULL` values because `NULL` represents an unknown or absent value rather than an ordinary value that must equal another one.

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);
```

This permits multiple users with `email = NULL`, but it rejects duplicate non-`NULL` email values.

If every user must have an email, use:

```sql
email VARCHAR(255) NOT NULL UNIQUE
```

For a composite unique index, MySQL also permits multiple rows whose indexed values include `NULL`, subject to the index's uniqueness rules for non-`NULL` values.

This distinction matters for optional identifiers, external IDs, and email addresses.

**Interview point:** A unique constraint prevents duplicate indexed values, but it does not necessarily enforce the existence of a value. Combine `UNIQUE` and `NOT NULL` when both properties are required.

## 6. How do generated columns protect data?

A generated column derives its value from an expression rather than relying on every application writer to calculate it correctly.

MySQL supports both `VIRTUAL` and `STORED` generated columns:

- **VIRTUAL:** The value is computed when read, rather than stored as an ordinary column value.
- **STORED:** The computed value is stored and maintained when the underlying row changes.

Example:

```sql
CREATE TABLE order_lines (
    id BIGINT PRIMARY KEY,
    quantity INT NOT NULL,
    unit_price DECIMAL(12, 2) NOT NULL,

    total DECIMAL(14, 2)
        GENERATED ALWAYS AS (quantity * unit_price) STORED
);
```

The application cannot accidentally save an independently calculated `total` that disagrees with `quantity * unit_price`.

Generated columns can also support indexing expressions, including selected JSON-derived values:

```sql
CREATE TABLE events (
    id BIGINT PRIMARY KEY,
    payload JSON,

    event_type VARCHAR(50)
        GENERATED ALWAYS AS (
            JSON_UNQUOTE(JSON_EXTRACT(payload, '$.type'))
        ) STORED,

    INDEX idx_event_type (event_type)
);
```

This makes filtering by the extracted event type more practical than repeatedly parsing the JSON in every query.

Generated columns are not a universal replacement for constraints. For example, a generated total does not prove that the price or quantity is valid; use `CHECK` constraints for those conditions.

## 7. Why do large systems sometimes skip foreign keys?

Some large systems intentionally do not use database-enforced foreign keys, particularly when data is distributed across services or tables are partitioned and sharded.

### Common reasons

1. **Independent service ownership:** A service may own its database while another service stores only the external entity's ID. A database foreign key cannot enforce a relationship across two independent databases.
2. **Sharding:** Parent and child records may live on different shards. Cross-shard referential checks can add network calls, coordination, and latency.
3. **High write throughput:** Foreign-key checks and cascading operations add work and can increase lock contention.
4. **Online migrations and bulk ingestion:** Large schema changes or data imports may be easier to coordinate when relationships are validated separately.
5. **Operational independence:** Teams may want to deploy, partition, or migrate services without coordinating every related schema change.

However, skipping foreign keys does **not** make a system automatically faster or more scalable. It trades database-enforced consistency for application and operational responsibility.

### How do systems compensate?

- **Application validation:** Validate that referenced entities exist when creating or updating relationships.
- **Ownership boundaries:** Keep strongly related data in the same service and preferably the same database when practical.
- **Idempotent operations:** Make retries safe so repeated requests do not create duplicate relationships or corrupt state.
- **Transactional writes:** Use transactions when related records are in the same database.
- **Reconciliation jobs:** Periodically detect orphaned rows, invalid references, and inconsistent counts.
- **Soft deletes and lifecycle rules:** Mark entities as deleted or inactive instead of physically deleting them when historical references must remain valid.
- **Monitoring and repair tooling:** Track integrity violations and provide safe ways to repair them.
- **Outbox and event-driven workflows:** For cross-service changes, publish events reliably and make consumers idempotent. Events alone do not guarantee immediate consistency; temporary inconsistency must be handled explicitly.

For example, an `orders` service may store `customer_id` without a foreign key to a customer service's database. It can validate the customer through the customer service when creating an order, but it must also define what happens if the customer is subsequently deactivated or deleted.

## 8. What is my final design approach?

I use database-enforced constraints by default when related data is within the same relational database and the constraints fit the workload.

- Use **foreign keys** to prevent orphaned references.
- Use **CASCADE** for genuinely dependent records whose lifecycle belongs to the parent.
- Use **RESTRICT** when deleting a parent would violate business or historical requirements.
- Use **SET NULL** only for genuinely optional relationships.
- Use **CHECK**, **UNIQUE**, and **NOT NULL** to enforce valid values, uniqueness, and required fields.
- Use **generated columns** to centralize deterministic derived values and make selected expressions indexable.
- Skip foreign keys only when there is a clear architectural or operational reason, with explicit compensating controls.

The key distinction is that application validation improves the user experience, while database constraints protect data integrity against every writer. In a distributed system, some relationships cannot be enforced by a single database, so the architecture must make the consistency guarantees and failure-recovery mechanisms explicit.

*Q16. NULL semantics and tricky query results. How does three-valued logic affect NOT IN with NULLs, COUNT(`*`) vs COUNT(col), GROUP BY, DISTINCT and UNIQUE indexes? What does ONLY_FULL_GROUP_BY do, and how do LEFT JOIN filters in WHERE vs ON change results?*

The most important thing to understand about `NULL` is that it does not mean zero, an empty string, or a normal value. It represents missing or unknown information. SQL uses **three-valued logic: `TRUE`, `FALSE`, and `UNKNOWN`**.

Many SQL bugs happen because developers assume that `NULL` behaves like an ordinary value or that filtering happens the same way across `WHERE`, `ON`, and aggregate expressions.

## 1. How does three-valued logic work?

Consider this query:

```sql
SELECT *
FROM employees
WHERE salary > 50000;
```

If an employee's salary is `NULL`, the expression `salary > 50000` evaluates to `UNKNOWN`, not `FALSE`.

A `WHERE` clause retains only rows for which its condition evaluates to `TRUE`. Rows evaluating to `FALSE` or `UNKNOWN` are excluded.

To check for missing values, use:

```sql
SELECT *
FROM employees
WHERE salary IS NULL;
```

Do not use `salary = NULL` or `salary <> NULL`; ordinary comparisons with `NULL` evaluate to `UNKNOWN`.

The basic logical rules are:

| Expression | Result |
|---|---|
| `NULL = NULL` | `UNKNOWN` |
| `NULL <> 10` | `UNKNOWN` |
| `TRUE AND UNKNOWN` | `UNKNOWN` |
| `FALSE AND UNKNOWN` | `FALSE` |
| `TRUE OR UNKNOWN` | `TRUE` |
| `NOT UNKNOWN` | `UNKNOWN` |

This becomes especially important with negation and subqueries.

## 2. Why is `NOT IN` dangerous when the subquery returns `NULL`?

Suppose we have these tables:

`customers`

| id | name |
|---:|---|
| 1 | Asha |
| 2 | Ravi |
| 3 | Meera |

`blocked_customers`

| customer_id |
|---:|
| 2 |
| `NULL` |

Now consider:

```sql
SELECT *
FROM customers
WHERE id NOT IN (
    SELECT customer_id
    FROM blocked_customers
);
```

Conceptually, the condition for customer 1 becomes:

```sql
1 <> 2 AND 1 <> NULL
```

The first comparison is `TRUE`, but the second is `UNKNOWN`. Therefore:

```sql
TRUE AND UNKNOWN = UNKNOWN
```

Customer 1 is excluded. The result can unexpectedly contain **zero rows**, even though customers 1 and 3 are not blocked.

### Safer solution: `NOT EXISTS`

```sql
SELECT c.*
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM blocked_customers b
    WHERE b.customer_id = c.id
);
```

This returns customers 1 and 3 because `NOT EXISTS` checks whether a matching row exists. A `NULL` in an unrelated `customer_id` does not invalidate the result.

Another option is to filter `NULL` values from the subquery:

```sql
SELECT *
FROM customers
WHERE id NOT IN (
    SELECT customer_id
    FROM blocked_customers
    WHERE customer_id IS NOT NULL
);
```

**Interview rule:** When expressing an anti-join, I generally prefer `NOT EXISTS`, especially when the subquery column may contain `NULL`. I still check execution plans rather than assuming one formulation is always faster.

## 3. `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT column)`

These expressions count different things.

Suppose a table contains:

| id | department |
|---:|---|
| 1 | IT |
| 2 | HR |
| 3 | `NULL` |
| 4 | IT |

```sql
SELECT
    COUNT(*) AS total_rows,
    COUNT(department) AS non_null_departments,
    COUNT(DISTINCT department) AS unique_departments
FROM employees;
```

The results are:

| Expression | Result | Reason |
|---|---:|---|
| `COUNT(*)` | 4 | Counts all rows |
| `COUNT(department)` | 3 | Ignores `NULL` values |
| `COUNT(DISTINCT department)` | 2 | Counts distinct non-`NULL` values |

This distinction matters in reports. For example, `COUNT(email)` does not count users whose email is missing, whereas `COUNT(*)` counts those users.

For counting missing values explicitly:

```sql
SELECT COUNT(*) - COUNT(department) AS missing_departments
FROM employees;
```

## 4. How do `GROUP BY` and `DISTINCT` handle `NULL`?

### `GROUP BY`

In MySQL, `GROUP BY` places `NULL` values in the same group.

```sql
SELECT department, COUNT(*) AS employee_count
FROM employees
GROUP BY department;
```

All employees with `department IS NULL` belong to one group.

### `DISTINCT`

`DISTINCT` removes duplicate result rows, treating `NULL` values as duplicates for this purpose.

```sql
SELECT DISTINCT department
FROM employees;
```

If several rows have a `NULL` department, the result contains one `NULL` row.

### Important distinction

`GROUP BY` and `DISTINCT` treat `NULL` as belonging to the same group for these operations, but this does **not** mean `NULL = NULL` evaluates to `TRUE`. Grouping and duplicate elimination have their own rules.

## 5. Why can a `UNIQUE` index allow multiple `NULL` values?

In MySQL, a unique index generally permits multiple `NULL` values.

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);
```

This allows multiple rows with `email = NULL`, but it rejects duplicate non-`NULL` email addresses.

If every user must have a unique email, declare both constraints:

```sql
email VARCHAR(255) NOT NULL UNIQUE
```

For a composite unique index, MySQL can also permit multiple rows where one or more indexed columns are `NULL`.

For example:

```sql
UNIQUE KEY uq_tenant_email (tenant_id, email)
```

If `email` is `NULL`, multiple rows may satisfy the index's uniqueness rules for that tenant. This is important when designing optional identifiers and multi-tenant schemas.

**Interview distinction:** `DISTINCT` removes duplicate values from a query result. `UNIQUE` prevents certain duplicate values from being stored. Their treatment of `NULL` is related but serves different purposes.

## 6. What does `ONLY_FULL_GROUP_BY` do?

`ONLY_FULL_GROUP_BY` is a MySQL SQL mode that rejects ambiguous aggregate queries.

Consider:

```sql
SELECT department, employee_name, COUNT(*)
FROM employees
GROUP BY department;
```

If a department contains multiple employees, which `employee_name` should MySQL return? The query does not specify.

With `ONLY_FULL_GROUP_BY` enabled, MySQL rejects this query unless it can establish that the selected nonaggregated column is functionally dependent on the grouping columns or otherwise satisfies the mode's rules.

### Correct approach 1: Group by both columns

```sql
SELECT department, employee_name, COUNT(*)
FROM employees
GROUP BY department, employee_name;
```

This changes the grouping granularity, so it is correct only if that is what the report requires.

### Correct approach 2: Aggregate the name-related information

```sql
SELECT
    department,
    COUNT(*) AS employee_count,
    MIN(employee_name) AS first_name_alphabetically
FROM employees
GROUP BY department;
```

Here, `MIN(employee_name)` has an explicit meaning: the alphabetically smallest name.

### Correct approach 3: Select the latest employee per department

If the actual requirement is to retrieve the latest employee, use a window function:

```sql
WITH ranked AS (
    SELECT
        department,
        employee_name,
        hired_at,
        ROW_NUMBER() OVER (
            PARTITION BY department
            ORDER BY hired_at DESC, id DESC
        ) AS rn
    FROM employees
)
SELECT department, employee_name, hired_at
FROM ranked
WHERE rn = 1;
```

The `id` tie-breaker makes the ordering deterministic when hiring timestamps are equal.

**My approach:** I keep `ONLY_FULL_GROUP_BY` enabled and write queries that express the intended result explicitly. Disabling it can hide ambiguous queries and produce arbitrary or misleading values.

## 7. How do `LEFT JOIN` filters in `WHERE` versus `ON` change results?

This is one of the most common SQL interview traps.

Suppose we want every customer and any orders they placed in the `completed` state, including customers who have no completed orders.

### Incorrect for this requirement: filter in `WHERE`

```sql
SELECT c.id, o.id AS order_id
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.id
WHERE o.status = 'completed';
```

A `LEFT JOIN` initially preserves unmatched customers by producing `NULL` values for the order columns. However, for those rows, `o.status = 'completed'` evaluates to `UNKNOWN`, so `WHERE` removes them.

This query effectively behaves like an inner join for that condition: customers without a completed order disappear.

### Correct: filter in `ON`

```sql
SELECT c.id, o.id AS order_id
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.id
   AND o.status = 'completed';
```

Now the database matches only completed orders but preserves every customer. A customer without a completed order appears with `order_id = NULL`.

### Why the difference matters

- `ON` determines which rows match during the join.
- `WHERE` filters the rows produced by the join.
- A condition in `WHERE` that rejects `NULL`-extended rows can eliminate the outer join's preservation behavior.

Not every `WHERE` condition has this effect. For example, `WHERE c.active = 1` filters customers based on the left table and does not inherently turn the join into an inner join.

### A subtle trap: `OR ... IS NULL`

This is not always equivalent to moving a condition into `ON`:

```sql
WHERE o.status = 'completed' OR o.id IS NULL
```

It preserves customers with no orders, but it can still exclude customers who have orders that are all non-completed. Those customers matched existing order rows, so the join did not produce a `NULL`-extended row.

When the requirement is "all customers, with only completed orders attached," filtering in `ON` is the clearest formulation.

## 8. Practical interview summary

When I debug surprising SQL results, I check these points:

1. **Three-valued logic:** Could a comparison evaluate to `UNKNOWN` because of `NULL`?
2. **Anti-joins:** Could a `NOT IN` subquery contain `NULL`? Would `NOT EXISTS` express the requirement more safely?
3. **Aggregates:** Do I need all rows (`COUNT(*)`), non-`NULL` values (`COUNT(col)`), or distinct non-`NULL` values (`COUNT(DISTINCT col)`)?
4. **Grouping and uniqueness:** Am I confusing query-result deduplication with storage-level uniqueness?
5. **Grouping correctness:** Is `ONLY_FULL_GROUP_BY` exposing an ambiguous query, or is there a clear aggregate or window-function solution?
6. **Outer joins:** Does a predicate belong in `ON` to preserve unmatched left-side rows, or in `WHERE` to filter the final result?

The underlying principle is that SQL's `NULL` handling, grouping, constraints, and join semantics are distinct rules. Writing reliable queries means understanding each rule rather than treating missing values as ordinary values.

*Q17. Bulk operations and upserts. Compare INSERT ... ON DUPLICATE KEY UPDATE, REPLACE INTO (delete plus insert, with its side effects) and INSERT IGNORE. How do you do fast bulk loading (LOAD DATA, multi-row inserts, batching), delete millions of rows without long locks, and handle auto-increment gaps and innodb_autoinc_lock_mode?*

For bulk operations in MySQL, I focus on three things: **correctness, throughput, and transaction impact**. The fastest SQL statement is not necessarily the safest one, especially when concurrent requests, foreign keys, triggers, replication, and millions of rows are involved.

## 1. `INSERT ... ON DUPLICATE KEY UPDATE` vs `REPLACE INTO` vs `INSERT IGNORE`

These statements handle duplicate-key conflicts differently.

### A. `INSERT ... ON DUPLICATE KEY UPDATE`

This is MySQL's standard upsert pattern: insert a row, or update the existing row if a primary-key or unique-index conflict occurs.

```sql
INSERT INTO products (id, name, price)
VALUES
    (101, 'Keyboard', 1500.00),
    (102, 'Mouse', 700.00)
AS incoming
ON DUPLICATE KEY UPDATE
    name = incoming.name,
    price = incoming.price;
```

The `AS incoming` row-alias syntax is supported in MySQL 8.0.19 and later. Older MySQL versions commonly use `VALUES(name)` and `VALUES(price)`, although that form is deprecated in newer MySQL versions.

**Advantages**
- Updates existing rows without explicitly deleting and recreating them.
- Supports conditional updates and can preserve existing column values.
- Works well for synchronization jobs and repeated ingestion of the same records.

**Risks**
- A conflict on any applicable primary or unique key can trigger the update path, not just the key you intended.
- The update may fire update triggers and modify `updated_at` or other audit fields.
- Concurrent upserts can still contend on unique indexes and deadlock.
- A multi-row statement can encounter conflicts with different unique keys; its behavior should be tested against the actual schema.

I use this when the intended semantics are “create or update this entity.”

### B. `REPLACE INTO`

`REPLACE` is not an ordinary update. When a conflicting row exists, MySQL can delete the conflicting row and insert the new row.

```sql
REPLACE INTO products (id, name, price)
VALUES (101, 'Keyboard', 1500.00);
```

Potential side effects include:

- `DELETE` triggers and foreign-key delete actions may execute, followed by insert-related behavior.
- With `ON DELETE CASCADE`, dependent rows can be deleted.
- Columns omitted from the replacement may receive defaults rather than retaining their old values.
- The new row may receive a different auto-increment value if the conflicting key is not explicitly supplied.
- Delete and insert operations can generate more logging and locking work than an update.

For example, if a product is referenced by order history and a foreign key uses `ON DELETE CASCADE`, replacing the product can unintentionally delete dependent records.

**My rule:** I avoid `REPLACE` for ordinary business entities. I use it only when delete-and-reinsert semantics are explicitly intended and all trigger, foreign-key, and default-value effects are understood.

### C. `INSERT IGNORE`

`INSERT IGNORE` attempts an insert but suppresses certain errors, converting some errors into warnings and adjusting invalid values in some circumstances.

```sql
INSERT IGNORE INTO products (id, name, price)
VALUES
    (101, 'Keyboard', 1500.00),
    (102, 'Mouse', 700.00);
```

Duplicate-key rows may be skipped, while valid rows are inserted. However, the statement can also downgrade certain data errors or constraint violations to warnings, depending on the error.

That makes it dangerous to assume that every row was inserted correctly.

After a bulk operation, I inspect affected-row counts and warnings, and validate the resulting data. For critical imports, I prefer explicit staging and error reporting rather than broadly suppressing errors.

### Quick comparison

| Behavior | `ON DUPLICATE KEY UPDATE` | `REPLACE` | `INSERT IGNORE` |
|---|---|---|---|
| Duplicate key | Updates conflicting row | Deletes/reinserts conflicting row | Usually skips conflicting insert |
| Preserves unspecified existing columns | Usually, if the update clause leaves them unchanged | No; defaults or generated values may apply | Existing row stays unchanged |
| Delete triggers / FK delete actions | Not because of the upsert itself | May execute | Not for a skipped duplicate |
| Update triggers | Can execute | Not as an ordinary update; insert/delete behavior applies | No update for a skipped duplicate |
| Typical use | Synchronization and upserts | Intentional replacement | Best-effort imports with explicit validation |

Affected-row counts for upserts are not always a simple count of inserted records: inserts, updates, and unchanged updates can be reported differently. Client flags can also affect reporting.

## 2. How do you perform fast bulk loading?

The right strategy depends on the source format, row count, indexes, constraints, and whether the import must be atomic.

### A. Multi-row `INSERT`

Instead of sending one statement per row:

```sql
INSERT INTO events (event_type, created_at)
VALUES ('login', NOW());

INSERT INTO events (event_type, created_at)
VALUES ('logout', NOW());
```

Send multiple rows in a single statement:

```sql
INSERT INTO events (event_type, created_at)
VALUES
    ('login', NOW()),
    ('logout', NOW()),
    ('login', NOW()),
    ('purchase', NOW());
```

This reduces network round trips and statement-parsing overhead.

For large datasets, I batch the inserts instead of building one enormous statement. Batch size depends on row size, network limits, server capacity, and transaction duration. A practical starting point might be hundreds to a few thousand rows per batch, followed by benchmarking.

### B. `LOAD DATA`

For a large CSV or delimited file, `LOAD DATA` is often much faster than issuing individual inserts.

```sql
LOAD DATA LOCAL INFILE '/path/events.csv'
INTO TABLE events
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(event_type, created_at);
```

Important considerations:

- `LOCAL` reads the file from the client side; without `LOCAL`, the file is read from the server side.
- `LOCAL INFILE` must be permitted by the relevant client and server configuration.
- Validate file encoding, quoting, line endings, date formats, and malformed rows.
- Use least-privilege access and trusted input files.
- In production, staging tables are useful when the source needs validation, deduplication, or transformation before it reaches the final tables.

### C. Use transactions and batching deliberately

For transactional tables such as InnoDB, grouping related inserts into a transaction can reduce commit overhead.

```sql
START TRANSACTION;

INSERT INTO events (event_type)
VALUES ('login'), ('logout'), ('purchase');

COMMIT;
```

But an extremely large transaction can cause excessive undo/redo activity, lock retention, replication lag, and expensive rollback.

For very large imports, I usually:

1. Load into a staging table.
2. Validate types, required fields, and business rules.
3. Deduplicate and resolve conflicting keys.
4. Insert or upsert in measured batches.
5. Commit at controlled intervals where partial progress is acceptable.
6. Record rejected rows and reconcile expected versus actual counts.

### D. Indexes and constraints during loading

Every maintained index adds write work. For a large initial load into a new table, it can be more efficient to load data before building secondary indexes, provided the schema and integrity requirements permit it.

However, I do not casually drop foreign keys or unique constraints on a live production table to speed up a load. Doing so can admit invalid data, and rebuilding constraints afterward may be expensive.

The correct choice depends on whether this is a new isolated table, a production table receiving concurrent writes, or a replicated database.

## 3. How do you delete millions of rows without long locks?

A single large delete can create a long-running transaction, retain locks, generate substantial undo/redo logs, increase replication lag, and make rollback costly.

### A. Delete in batches

For example, suppose old events must be removed:

```sql
DELETE FROM events
WHERE created_at < '2025-01-01'
ORDER BY id
LIMIT 5000;
```

Repeat the statement until no qualifying rows remain, committing between batches when appropriate.

The exact batch size should be tuned based on execution time, lock waits, replication lag, and workload. `LIMIT` alone does not make the operation safe: the predicate needs an appropriate index.

For predictable traversal, I often use a primary-key cursor or fixed ID ranges:

```sql
DELETE FROM events
WHERE id > 100000
  AND id <= 105000
  AND created_at < '2025-01-01';
```

I advance through ranges in a controlled job, making progress observable and restartable. I ensure the chosen ranges and deletion predicate match the intended data.

### B. Index the deletion predicate

If the condition is:

```sql
WHERE created_at < '2025-01-01'
```

an index on `created_at` may help avoid a full table scan. The best index depends on selectivity and whether ordering or additional predicates are involved.

I inspect `EXPLAIN` and measure the actual delete workload. A full scan repeated for every batch can be worse than a single well-planned operation.

### C. Consider partitioning for time-based retention

For a very large time-series table partitioned by date, dropping or truncating an expired partition can be much more efficient than deleting millions of individual rows.

This requires the table to have been partitioned appropriately in advance. MySQL partitioning also imposes unique-key requirements, including that every unique key contain the partitioning columns. It is an architectural decision, not a quick substitute for indexing.

### D. Foreign keys and triggers matter

Deleting parent rows can trigger cascades and additional writes. A small batch of parent rows can still affect a very large number of child rows.

Before running a purge, I inspect foreign-key relationships, triggers, and expected fan-out. I schedule large cleanup jobs away from peak load where possible and monitor transaction duration and replica health.

**Important:** InnoDB normally uses row-level locking, but “row-level” does not mean only the rows returned by a query are always locked. Depending on the isolation level, index access path, and predicate, range or gap locks can also matter.

## 4. Why do auto-increment values have gaps?

Auto-increment values are identifiers, not guaranteed sequential business numbers.

Gaps can appear because:

- An insert reserves an ID but later fails or rolls back.
- An upsert takes an insert path that reserves an ID before resolving a duplicate.
- Bulk inserts reserve ranges of values.
- Concurrent inserts interleave.
- Rows are deleted.
- The server restarts or uses replication and allocation behavior that leaves unused values.

For example, if IDs 1, 2, and 3 exist and a transaction reserves ID 4 but rolls back, the next inserted row may receive ID 5.

I do not try to eliminate these gaps. Doing so can introduce contention and correctness problems without providing business value. If the business requires a gapless document number, I design a separate transactional numbering mechanism and understand that strict gaplessness can reduce concurrency.

## 5. What is `innodb_autoinc_lock_mode`?

This setting controls how InnoDB allocates auto-increment values, particularly under concurrent insert workloads.

| Mode | Name | Behavior |
|---|---|---|
| `0` | Traditional | Uses a table-level `AUTO-INC` lock for insert statements that need auto-increment allocation; holds it to statement completion |
| `1` | Consecutive | Uses the table-level `AUTO-INC` lock for bulk inserts that need it; simple inserts generally use a lighter allocation mechanism |
| `2` | Interleaved | Avoids the table-level `AUTO-INC` lock for auto-increment allocation, allowing concurrent statements to interleave values |

### Why does this matter?

- **Mode 0:** Most restrictive; offers predictable statement-level allocation but limits concurrency.
- **Mode 1:** Balances predictable allocation for bulk inserts with better concurrency for simple inserts.
- **Mode 2:** Provides greater concurrency, especially for bulk workloads, but values allocated to different concurrent statements may interleave.

Mode 2 is generally the default in modern MySQL 8.0 deployments. It is important to understand the replication implications: statement-based replication requires particular care with interleaved allocation, so row-based replication is the safer general choice for mode 2.

Even in modes 0 and 1, auto-increment gaps are still possible. None of these modes guarantees gapless IDs.

## 6. How would I choose the right strategy in production?

My decision process is:

1. **Define semantics first.** Choose upsert, replacement, or ignore based on the intended behavior—not merely speed.
2. **Choose the loading path.** Use multi-row inserts for moderate batches and `LOAD DATA` for large file imports where appropriate.
3. **Control transaction size.** Avoid both excessive per-row commits and enormous transactions.
4. **Make destructive jobs restartable.** Use indexed predicates, bounded batches, checkpoints, and progress metrics.
5. **Measure concurrency effects.** Monitor locks, deadlocks, undo/redo pressure, replication lag, and query latency.
6. **Validate results.** Reconcile input counts, inserted/updated/skipped counts, warnings, and rejected records.
7. **Treat IDs as surrogate keys.** Never assume auto-increment values are gapless or use them as proof of transaction ordering.

The key principle is that high-throughput database operations need explicit semantics and bounded resource usage. `LOAD DATA`, multi-row inserts, batched deletes, and the appropriate upsert strategy improve performance, but only when they preserve the integrity and recovery guarantees the application actually needs.

*Q18. Query cache removal, buffer pool and server tuning. Why was the query cache removed in 8.0? How do you size innodb_buffer_pool_size, tune innodb_flush_log_at_trx_commit and sync_binlog (durability vs speed), and use performance_schema and sys schema views to find top queries, locks and I/O waits?*

When tuning MySQL, I start by identifying the bottleneck rather than randomly changing configuration variables. I examine query latency, execution plans, buffer-pool hit behavior, disk I/O, lock waits, transaction durability requirements, and workload characteristics.

The goal is to improve throughput and latency without weakening the database's required reliability guarantees.

## 1. Why was the MySQL Query Cache removed in MySQL 8.0?

The MySQL Query Cache stored the results of SQL statements so that an identical query could return cached results without executing the query again.

For example:

```sql
SELECT name, price
FROM products
WHERE id = 101;
```

If the query and cache conditions matched, MySQL could return the cached result.

However, the query cache had significant limitations:

- **Invalidation overhead:** When a table changed, cached results involving that table could need invalidation.
- **Contention:** Concurrent queries and cache invalidations could compete for shared cache resources.
- **Limited usefulness for write-heavy workloads:** Frequent updates invalidated cached results, reducing the benefit.
- **Restricted cache matching:** Queries needed to meet the cache's matching rules; semantically equivalent SQL was not necessarily the same cache entry.
- **Poor scalability:** On highly concurrent systems, maintaining the cache could cost more than executing inexpensive queries.

The query cache was deprecated in MySQL 5.7 and removed in MySQL 8.0.

### What do I use instead?

I distinguish between several different kinds of caching:

- **InnoDB buffer pool:** Caches database pages in memory, reducing disk reads while queries still execute.
- **Application or distributed cache:** Redis or an application-level cache can store frequently requested results with explicit expiration and invalidation strategies.
- **Execution-plan and metadata optimizations:** MySQL manages statement preparation and internal metadata independently of the removed query cache.

The buffer pool is not a replacement for the query cache in the exact same sense. It caches data and index pages, not complete SQL result sets.

## 2. How do I size `innodb_buffer_pool_size`?

The InnoDB buffer pool holds cached table data and index pages. A sufficiently large buffer pool reduces physical reads when the working dataset is accessed repeatedly.

I size it based on available RAM, the database working set, concurrency, and memory consumed by the operating system and other processes.

### A practical starting point

For a dedicated MySQL server, a common initial target is approximately **60–75% of system RAM**, sometimes higher on a well-understood database-only host. It is not a universal rule.

For example, on a dedicated server with 32 GiB of RAM:

```ini
[mysqld]
innodb_buffer_pool_size = 22G
```

This is a starting configuration, not a guaranteed optimum.

I leave enough memory for:

- The operating system and filesystem cache.
- Per-connection buffers and sort/join operations.
- Temporary tables and other internal allocations.
- Performance Schema and replication.
- Monitoring agents and backup processes.

Memory allocated for certain query operations is per connection or per operation. Setting these buffers too high can create memory pressure when many queries execute concurrently.

### How do I validate the setting?

First, inspect the current configuration:

```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages%';
```

Then examine trends over a representative workload rather than relying on a single snapshot.

Useful indicators include:

- Physical reads relative to logical buffer-pool reads.
- Buffer-pool occupancy and free pages.
- Disk read latency and I/O queue depth.
- Query latency and throughput under peak concurrency.
- Operating-system memory pressure, swapping, and out-of-memory events.

A high buffer-pool hit rate is useful, but it does not prove that the database is optimally tuned. A workload can still be slow because of random I/O, inefficient queries, locks, or a working set that exceeds memory.

**My approach:** I tune the buffer pool after checking indexes and execution plans. Adding RAM cannot reliably compensate for a query that scans millions of unnecessary rows.

## 3. What does `innodb_flush_log_at_trx_commit` control?

This setting controls how InnoDB flushes its redo log during transaction commits.

| Value | Behavior | Durability implications |
|---|---|---|
| `1` | Writes redo log records and flushes them to stable storage at each commit | Strongest normal InnoDB commit durability |
| `2` | Writes the redo log at commit but generally flushes it to stable storage about once per second | Recent committed transactions can be lost after an operating-system crash or power failure |
| `0` | Generally writes and flushes redo logs about once per second, independently of individual commits | Recent transactions can be lost after a MySQL server crash as well as more severe failures |

The exact timing depends on the implementation and workload; the approximately one-second behavior is not a hard real-time guarantee.

### Configuration examples

For stronger durability:

```ini
[mysqld]
innodb_flush_log_at_trx_commit = 1
```

For a workload that explicitly accepts a potential loss of recent committed transactions:

```ini
[mysqld]
innodb_flush_log_at_trx_commit = 2
```

### What is the trade-off?

With value `1`, the database performs more frequent log flushes, which can increase commit latency when storage flushes are expensive.

Values `0` and `2` can improve throughput in some workloads by reducing the frequency of flush operations, but they weaken crash durability.

I do not change this setting solely to make a benchmark faster. For financial transactions, payments, orders, and other critical data, I normally retain value `1` unless the business explicitly accepts the risk.

Also, durability depends on the storage stack honoring flush requests. A controller or device that falsely reports data as safely persisted can undermine the intended guarantee.

## 4. How does `sync_binlog` affect durability?

MySQL's binary log records changes for replication and point-in-time recovery. `sync_binlog` controls how often MySQL synchronizes the binary log to stable storage.

| Setting | Meaning |
|---|---|
| `sync_binlog = 1` | Synchronizes the binary log at each commit |
| `sync_binlog = 0` | Leaves synchronization timing to the operating system |
| `sync_binlog = N` | Synchronizes after approximately every N binary-log commit groups |

For stronger durability, a common configuration is:

```ini
[mysqld]
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```

These settings protect different logs:

- The **InnoDB redo log** supports crash recovery of transactional data.
- The **binary log** supports replication and point-in-time recovery.

Using `innodb_flush_log_at_trx_commit = 1` without appropriately synchronizing the binary log can leave a different durability risk for replication or recovery workflows. Using `sync_binlog = 1` alone does not guarantee that InnoDB's data changes are durable.

### How do I choose values?

- **Critical transactional systems:** Usually `1` for both.
- **Workloads that explicitly tolerate a recovery-point loss:** May consider less frequent flushing after measuring the risk and performance impact.
- **Bulk imports or disposable data:** May justify different settings in an isolated, controlled environment, followed by restoration of the normal durability policy.

I measure commit latency and storage behavior before changing these values. I also verify backup and point-in-time recovery procedures, because durability settings do not replace tested recovery plans.

## 5. How do I use Performance Schema to find expensive queries?

Performance Schema collects instrumentation about statements, waits, locks, and other server activity. It provides the data used by many diagnostic views in the `sys` schema.

First, I check whether the relevant instrumentation and consumers are enabled. Then I use statement summaries to identify query patterns with high cumulative cost.

For example:

```sql
SELECT
    DIGEST_TEXT,
    COUNT_STAR AS executions,
    ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_seconds,
    ROUND(AVG_TIMER_WAIT / 1e12, 4) AS avg_seconds,
    SUM_ROWS_EXAMINED AS rows_examined,
    SUM_ROWS_SENT AS rows_sent
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

The timer values in this Performance Schema summary are reported in picoseconds, hence division by \(10^{12}\) to display seconds.

This query helps identify:

- Queries consuming the most total execution time.
- Queries with high average latency.
- Queries examining many rows but returning few.
- Frequently executed queries whose cumulative cost is substantial.

I also compare total execution time with execution count. A query taking 20 milliseconds but executing hundreds of thousands of times may deserve more attention than a query taking one second and running once a day.

### Useful `sys` schema views

| View | Purpose |
|---|---|
| `sys.statement_analysis` | Aggregated statement performance, execution counts, latency, and rows examined |
| `sys.statements_with_runtimes_in_95th_percentile` | Queries with high tail latency |
| `sys.statements_with_full_table_scans` | Statements associated with full table scans |
| `sys.statements_with_sorting` | Queries that perform sorting |
| `sys.statements_with_temp_tables` | Queries that use temporary tables |
| `sys.innodb_lock_waits` | InnoDB lock waits and their associated transactions |
| `sys.io_global_by_file_by_latency` | I/O activity and latency by file |
| `sys.schema_table_statistics` | Table-level read and write activity |

View availability and columns can vary with MySQL version and configuration, so I check the server's installed `sys` schema before relying on a particular view.

## 6. How do I investigate lock waits and deadlocks?

When an application reports timeouts or intermittent slow writes, I inspect current lock waits and transaction state before changing isolation levels or indexes.

For example:

```sql
SELECT *
FROM sys.innodb_lock_waits;
```

This view helps identify waiting transactions, blocking transactions, and the affected objects. I then inspect the relevant SQL statements and transaction duration.

For deeper diagnosis, I use:

```sql
SHOW ENGINE INNODB STATUS;
```

The output includes the latest detected deadlock and other InnoDB diagnostic information.

I look for:

- Transactions left open while the application performs unrelated work.
- Inconsistent row-lock acquisition order.
- Missing indexes that force broad scans during updates or deletes.
- Large batch operations competing with interactive traffic.
- Long-running transactions retaining locks or preventing purge progress.

The fix depends on the cause. It may involve an index, shorter transactions, consistent lock ordering, smaller batches, or retrying a deadlocked transaction safely.

I avoid blindly increasing lock timeouts because that can hide the underlying contention and allow requests to accumulate.

## 7. How do I find I/O bottlenecks?

I start with the Performance Schema file summaries:

```sql
SELECT
    FILE_NAME,
    COUNT_READ,
    ROUND(SUM_TIMER_READ / 1e12, 2) AS read_seconds,
    COUNT_WRITE,
    ROUND(SUM_TIMER_WRITE / 1e12, 2) AS write_seconds
FROM performance_schema.file_summary_by_instance
ORDER BY SUM_TIMER_READ DESC
LIMIT 10;
```

This shows files with high cumulative read wait time. I also inspect write latency when diagnosing commit stalls, checkpoint pressure, or heavy logging.

The result is cumulative since the relevant instrumentation or server statistics were initialized, so I compare measurements over a known time interval when possible.

I correlate database measurements with operating-system metrics:

- Disk latency and queue depth.
- CPU utilization and I/O wait.
- Memory pressure and swapping.
- Storage throughput and IOPS.
- Redo-log and binary-log write behavior.

High I/O wait can indicate insufficient memory, inefficient access patterns, excessive writes, or slow storage. It is not automatically proof that the buffer pool is too small.

## 8. My production tuning workflow

I follow a repeatable process:

1. **Establish a baseline:** Measure throughput, p50/p95/p99 latency, CPU, memory, disk I/O, and connection counts.
2. **Find the expensive statements:** Use `sys.statement_analysis` and Performance Schema digest summaries.
3. **Inspect execution plans:** Use `EXPLAIN` and, where safe, `EXPLAIN ANALYZE` to identify scans, poor indexes, and expensive joins.
4. **Investigate contention:** Check lock waits, deadlocks, long transactions, and metadata locks.
5. **Evaluate the buffer pool:** Compare physical reads, working-set size, memory headroom, and storage behavior.
6. **Review durability settings:** Keep redo and binary-log synchronization aligned with recovery-point objectives.
7. **Change one variable or bottleneck at a time:** Benchmark against the baseline and observe the effects under realistic concurrency.
8. **Validate recovery and rollback:** Confirm that the configuration meets the business's durability requirements and that recovery procedures are tested.

My main principle is that server tuning should follow measurement. The query cache was removed because a global result cache did not scale well across many workloads; modern tuning focuses on efficient SQL, useful indexes, the InnoDB buffer pool, appropriate durability guarantees, and evidence from Performance Schema.

*Q19. Security and user management. How do you design least-privilege users and roles with GRANT, and why avoid app connections as root? What do caching_sha2_password, TLS connections, SQL injection prevention with prepared statements (server-side vs client-side emulation), and encryption at rest (tablespace encryption) each protect against?*

I approach MySQL security using **least privilege, strong authentication, encrypted communication, parameterized queries, and encryption at rest**. These controls address different threats, so I don't treat any single one as sufficient.

## 1. How do you design least-privilege users and roles with `GRANT`?

The principle of least privilege means each account receives only the permissions required for its specific responsibility.

I separate database accounts by workload instead of giving every application, developer, and background job full access.

For example:

| Account | Typical permissions |
|---|---|
| Application runtime | `SELECT`, `INSERT`, `UPDATE`, `DELETE` on required tables |
| Reporting service | Read-only access to approved tables or views |
| Migration account | Required schema changes, used only during deployments |
| Backup account | Specific backup and metadata privileges required by the backup tool |
| Database administrator | Administrative privileges for operational tasks |

### Create an application user

```sql
CREATE USER 'app_runtime'@'10.0.1.%'
IDENTIFIED BY 'use-a-generated-secret';

GRANT SELECT, INSERT, UPDATE, DELETE
ON restaurant_db.*
TO 'app_runtime'@'10.0.1.%';
```

This account can perform normal CRUD operations within `restaurant_db`, but it does not automatically receive permission to create users, change schemas, or administer the server.

The host component also matters. `'app_runtime'@'10.0.1.%'` limits matching connections by host pattern, although network firewalls and database network configuration should provide additional restrictions.

For a reporting account:

```sql
CREATE USER 'report_reader'@'10.0.2.%'
IDENTIFIED BY 'use-another-generated-secret';

GRANT SELECT
ON restaurant_db.*
TO 'report_reader'@'10.0.2.%';
```

For better isolation, I grant access to specific tables or views rather than the entire schema whenever practical.

### Use roles for reusable permissions

Roles make permission management easier when several accounts need the same access.

```sql
CREATE ROLE 'restaurant_app_role';

GRANT SELECT, INSERT, UPDATE, DELETE
ON restaurant_db.*
TO 'restaurant_app_role';

GRANT 'restaurant_app_role'
TO 'app_runtime'@'10.0.1.%';

SET DEFAULT ROLE 'restaurant_app_role'
TO 'app_runtime'@'10.0.1.%';
```

MySQL supports roles from version 8.0. The role must be granted to the account, and setting it as the default role ensures it is activated automatically at login.

### Why avoid connecting as `root`?

Using `root` for normal application connections creates an unnecessarily large blast radius.

If the application is compromised through SQL injection, a leaked credential, or a remote-code-execution vulnerability, an attacker could potentially perform destructive administrative actions, create accounts, change permissions, or access unrelated databases.

A dedicated application user limits the damage to the permissions it actually holds.

I also keep schema migration privileges separate from runtime privileges. A deployed application should not need `CREATE USER`, `DROP DATABASE`, or unrestricted schema modification rights to process an ordinary request.

## 2. What does `caching_sha2_password` protect against?

`caching_sha2_password` is the default authentication plugin for MySQL 8.0 and later.

It uses SHA-256-based password authentication with a server-side authentication cache. The cache helps avoid repeating the more expensive full authentication exchange for eligible subsequent connections.

It improves authentication security compared with the legacy `mysql_native_password` mechanism, which uses a weaker password-hashing scheme.

It does **not** mean that every connection is encrypted or that all database traffic is protected.

Important distinctions:

- Authentication establishes the client's identity.
- Authorization determines what that identity can do.
- TLS protects communication in transit.
- Encryption at rest protects stored data under specified threat conditions.

For secure deployments, I use strong, unique credentials, keep the authentication plugin current, and configure TLS where required. The full authentication exchange for `caching_sha2_password` requires a secure transport such as TLS or an appropriately configured RSA-based exchange.

## 3. What do TLS connections protect against?

TLS encrypts traffic between the application and MySQL and, when certificate validation is correctly configured, helps authenticate the server.

Without TLS, a network attacker who can observe traffic may be able to capture sensitive query data, results, or authentication-related information, depending on the authentication mechanism and connection configuration.

TLS helps protect against:

- Passive network eavesdropping.
- Modification of traffic in transit.
- Server impersonation when certificate verification is correctly enforced.

### Example configuration

A client can require an encrypted connection and validate the server certificate using an appropriate CA certificate. For example, with a compatible MySQL client:

```bash
mysql \
  --host=db.example.com \
  --user=app_runtime \
  --ssl-mode=VERIFY_IDENTITY \
  --ssl-ca=/path/to/ca.pem
```

`VERIFY_IDENTITY` verifies the server certificate and its hostname. The CA file must correspond to the certificate chain used by the server.

On the server, TLS certificates and private keys must be configured correctly, and account or server policies can require encrypted connections.

**Important:** TLS does not prevent SQL injection, excessive database privileges, or a compromised application from reading data it is authorized to access. It protects the transport channel, not the correctness of SQL or the trustworthiness of the application.

## 4. How do prepared statements prevent SQL injection?

SQL injection occurs when untrusted input changes the structure or meaning of a SQL statement.

Unsafe example:

```javascript
const sql =
  "SELECT * FROM users WHERE email = '" + email + "'";
```

If `email` contains malicious SQL syntax, string concatenation may allow that input to alter the query.

Instead, use parameterized queries:

```javascript
const [rows] = await connection.execute(
  "SELECT * FROM users WHERE email = ?",
  [email]
);
```

The SQL structure and parameter values are handled separately. The email is treated as data rather than executable SQL syntax.

### Server-side vs client-side prepared statements

This distinction is important in interviews.

**Server-side prepared statements**

The client sends a statement to MySQL for preparation and then supplies parameter values separately through the prepared-statement protocol.

Advantages include:

- Clear separation between SQL structure and parameter data.
- Reduced repeated parsing overhead in suitable workloads.
- Potential reuse of a prepared statement.

The actual performance benefit depends on the workload, driver, and statement lifecycle.

**Client-side emulation**

Some database drivers emulate prepared statements by safely escaping or substituting values in the client before sending the resulting SQL text to the server.

This can still prevent injection when implemented correctly, but it is not the same protocol-level separation as server-side preparation. Its security depends on the driver's implementation and correct handling of character sets, SQL modes, and values.

When security requirements are strict, I verify whether the driver uses native server-side preparation or emulation and configure it appropriately.

### What prepared statements do not protect against

Prepared statements parameterize **values**, not arbitrary SQL identifiers or syntax.

For example, this does not safely parameterize a column name:

```sql
SELECT * FROM users ORDER BY ?;
```

The placeholder represents a value, not a column identifier. For dynamic column names or sort directions, I use an explicit allowlist:

```javascript
const allowedSortColumns = {
  name: "name",
  created: "created_at"
};

const sortColumn = allowedSortColumns[requestedSort] ?? "created_at";
const direction = requestedDirection === "asc" ? "ASC" : "DESC";

const sql =
  `SELECT id, name FROM users ORDER BY ${sortColumn} ${direction}`;
```

Only predefined identifiers and directions enter the SQL string. User-supplied values still use parameters.

## 5. What does encryption at rest protect?

Encryption at rest protects database files and storage media when the threat involves someone obtaining the underlying encrypted data without the necessary keys.

In MySQL, InnoDB tablespace encryption can encrypt supported tablespaces and their associated data and index pages. Encryption key management depends on the MySQL edition, keyring configuration, and deployment architecture.

It helps protect against scenarios such as:

- Stolen database disks or storage volumes.
- Unauthorized access to copied database files.
- Exposure of encrypted tablespace files through certain backup or storage-handling failures, provided the encryption keys are not exposed alongside them.

### Example: tablespace encryption

A table can be created with encryption enabled where supported and properly configured:

```sql
CREATE TABLE payment_records (
    id BIGINT PRIMARY KEY,
    amount DECIMAL(12, 2) NOT NULL
) ENCRYPTION = 'Y';
```

The exact behavior depends on the tablespace type, server configuration, and key-management setup. Enabling the table option alone is not a complete encryption deployment.

### What encryption at rest does not protect against

Encryption at rest does not stop an authorized application account from querying decrypted data through MySQL. It also does not inherently protect against a compromised database server, an attacker with sufficient database privileges, or leaked encryption keys.

I therefore combine storage encryption with least-privilege access, TLS, secure key management, backup encryption, and restricted operating-system access.

## 6. How do these controls work together?

| Control | Primary protection | Does not replace |
|---|---|---|
| Least-privilege users and roles | Limits what an account can do | Authentication and encryption |
| `caching_sha2_password` | Stronger password authentication | TLS and authorization |
| TLS | Confidentiality and integrity in transit; server identity verification when configured | SQL injection prevention |
| Prepared statements | Prevents input values from altering SQL structure | Authorization and input validation |
| Tablespace encryption | Protects supported stored database files when keys remain secure | Runtime access control and TLS |
| Key and secret management | Reduces exposure of credentials and encryption keys | Database permissions and monitoring |

For example, a payment API should use a dedicated runtime account, parameterized SQL, TLS with certificate verification, and encrypted storage where required. Database credentials and encryption keys should be stored outside source control, with access restricted to the services that need them.

## 7. My production security checklist

1. Create separate runtime, reporting, migration, and administrative accounts.
2. Grant only required privileges at the narrowest practical scope.
3. Use strong credentials and a supported authentication plugin.
4. Require TLS for remote connections and verify server certificates.
5. Use parameterized SQL for all user-controlled values.
6. Allowlist dynamic SQL identifiers such as column names and sort directions.
7. Encrypt database files and backups where required, with independent key management.
8. Restrict network access through firewalls and private networking.
9. Rotate credentials, audit privileged access, and monitor failed logins and suspicious queries.
10. Test account permissions and recovery procedures as part of deployment.

**The key principle:** Authentication establishes who is connecting, authorization limits what they can do, prepared statements preserve SQL structure, TLS protects data in transit, and encryption at rest protects stored files. A secure MySQL deployment needs all applicable layers working together.

*Q20. JSON, full-text search and special data. When is a JSON column appropriate vs a normalized table? How do you query and index JSON paths (->>, generated columns, multi-valued indexes)? How do FULLTEXT indexes in natural language vs boolean mode work, and when do you move to Elasticsearch? What about spatial types for location queries?*

I choose the storage and indexing strategy based on the data's structure, query patterns, consistency requirements, and search complexity. MySQL supports more than relational data: it provides native JSON operations, full-text indexes, and spatial types. However, these features do not eliminate the need for good schema design.

## 1. JSON column vs. normalized tables

A MySQL `JSON` column is appropriate when data is semi-structured, its attributes vary between records, or the application needs to preserve flexible metadata. Normalized tables are better when attributes have stable meaning, participate in relationships, require strong constraints, or are frequently queried and updated independently.

**Example: storing restaurant menu item attributes**

```sql
CREATE TABLE menu_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(150) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    attributes JSON
);
```

An item might store:

```json
{
  "spice_level": "medium",
  "is_vegan": true,
  "allergens": ["peanuts", "soy"]
}
```

JSON works well here because different menu items may have different attributes.

However, I would normalize frequently used entities such as ingredients, suppliers, and inventory movements into separate tables because they need relationships, constraints, and reliable joins.

For example:

```sql
CREATE TABLE ingredients (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE menu_item_ingredients (
    menu_item_id BIGINT NOT NULL,
    ingredient_id BIGINT NOT NULL,
    quantity DECIMAL(10,3) NOT NULL,
    PRIMARY KEY (menu_item_id, ingredient_id),
    FOREIGN KEY (menu_item_id) REFERENCES menu_items(id),
    FOREIGN KEY (ingredient_id) REFERENCES ingredients(id)
);
```

**Trade-offs:**

- **JSON:** flexible schema, fewer tables, convenient for metadata and variable attributes.
- **Normalized tables:** stronger relational integrity, easier joins, explicit data types, and better control over uniqueness and updates.
- **Hybrid approach:** keep core business fields relational and use JSON for optional or variable attributes.

JSON is not automatically faster or more scalable than relational modeling. I avoid putting entire business entities into JSON simply to avoid designing tables.

## 2. Querying JSON data

MySQL validates JSON documents when they are stored in a native `JSON` column. JSON paths let me access individual properties.

```sql
SELECT
    id,
    name,
    attributes->>'$.spice_level' AS spice_level
FROM menu_items
WHERE attributes->>'$.spice_level' = 'medium';
```

The operators are important:

- `->` extracts a JSON value and returns JSON.
- `->>` extracts a JSON value and returns its unquoted SQL representation.

The `->>` operator is shorthand for an unquoting JSON extraction operation. For numeric comparisons, I use an explicit cast when needed:

```sql
SELECT *
FROM menu_items
WHERE CAST(
    attributes->>'$.calories' AS UNSIGNED
) < 500;
```

I also use functions such as `JSON_EXTRACT()`, `JSON_CONTAINS()`, `JSON_SET()`, and `JSON_ARRAY()` when the query requires more specific JSON operations.

For example, to find items whose JSON array contains `"peanuts"`:

```sql
SELECT *
FROM menu_items
WHERE JSON_CONTAINS(
    attributes->'$.allergens',
    JSON_QUOTE('peanuts')
);
```

### Important performance consideration

A normal index on the JSON document does not automatically index every JSON path. If I repeatedly filter by the same property, I make that property indexable.

### Option A: Generated column plus index

```sql
ALTER TABLE menu_items
ADD COLUMN spice_level VARCHAR(20)
    GENERATED ALWAYS AS (
        attributes->>'$.spice_level'
    ) STORED,
ADD INDEX idx_spice_level (spice_level);
```

Now the query can use the generated column:

```sql
SELECT *
FROM menu_items
WHERE spice_level = 'medium';
```

A generated column derives its value from another column. A `STORED` generated column persists the derived value; a `VIRTUAL` one is calculated when read, although it can also be indexed when supported by the expression and index definition.

This is useful for frequently queried JSON properties such as category, status, or a business-specific attribute.

### Option B: Functional index

For supported expressions in MySQL 8.0.13 and later, I can index a JSON expression directly:

```sql
CREATE INDEX idx_item_spice
ON menu_items (
    (CAST(attributes->>'$.spice_level' AS CHAR(20)))
);
```

The query expression must be compatible with the indexed expression for MySQL to use the index. I verify the result using `EXPLAIN ANALYZE` rather than assuming the index will be selected.

### Option C: Multi-valued index

A multi-valued index lets a single JSON array produce multiple index entries. It is useful for searching array elements without creating a separate relational table for every use case.

For example:

```sql
CREATE INDEX idx_item_tag_ids
ON menu_items (
    (CAST(attributes->'$.tag_ids' AS UNSIGNED ARRAY))
);
```

This is intended for JSON arrays containing values compatible with the declared element type. A query can use the corresponding membership operation:

```sql
SELECT *
FROM menu_items
WHERE 42 MEMBER OF (attributes->'$.tag_ids');
```

Multi-valued indexes were introduced in MySQL 8.0.17. They support specific JSON array operations and have limitations; they are not a general-purpose replacement for normalized many-to-many tables.

**My rule:** if an array represents a genuine business relationship—such as a menu item containing multiple ingredients—I generally prefer a relational mapping table. If it represents flexible tags or identifiers and the query pattern fits, a multi-valued index may be appropriate.

## 3. Full-text search: `FULLTEXT`

A normal B-tree index is excellent for exact matches, ranges, and prefix searches. It is not designed to find relevant documents based on words throughout a long text field.

For that, MySQL provides `FULLTEXT` indexes on supported character columns, including `CHAR`, `VARCHAR`, and `TEXT`, in supported storage engines such as InnoDB.

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    FULLTEXT INDEX ft_product_search (name, description)
);
```

### Natural language mode

Natural language mode is the default. MySQL treats the search input as natural-language terms and calculates relevance scores.

```sql
SELECT
    id,
    name,
    MATCH(name, description)
        AGAINST('spicy paneer wrap') AS relevance
FROM products
WHERE MATCH(name, description)
        AGAINST('spicy paneer wrap')
ORDER BY relevance DESC;
```

The search returns matching documents and ranks them by relevance. Tokenization, stopwords, minimum token lengths, and language configuration influence the results.

### Boolean mode

Boolean mode supports operators that let the application express more explicit search requirements.

```sql
SELECT
    id,
    name,
    MATCH(name, description)
        AGAINST('+paneer +wrap -frozen' IN BOOLEAN MODE) AS score
FROM products
WHERE MATCH(name, description)
        AGAINST('+paneer +wrap -frozen' IN BOOLEAN MODE);
```

In this example:

- `+paneer` requires the term.
- `+wrap` requires the term.
- `-frozen` excludes documents containing that term.

Other supported operators can modify term weighting, require prefixes, or group expressions. Boolean mode does not apply exactly the same relevance and query semantics as natural language mode.

### What should I know in an interview?

- Full-text search indexes words, not arbitrary substrings.
- It is not equivalent to `LIKE '%paneer%'`.
- Stopwords and minimum token lengths can cause seemingly valid terms not to match.
- InnoDB and MyISAM have different historical full-text behaviors; I check the storage engine and MySQL version.
- For InnoDB, full-text indexes support transactional tables, but index maintenance still adds write overhead.

I use MySQL full-text search for straightforward product catalogs, documentation, descriptions, and moderate search requirements where keeping search in the primary database is operationally simpler.

## 4. When would I move to Elasticsearch?

I do not introduce Elasticsearch merely because a table contains millions of rows. First, I identify the missing search capabilities and measure the existing implementation.

I consider Elasticsearch or another dedicated search engine when requirements include:

- Advanced relevance tuning, synonyms, fuzziness, typo tolerance, and language analyzers.
- Complex faceted search and aggregations across large document collections.
- Rich filtering and ranking over many fields.
- Search workloads that need to scale independently from transactional database workloads.
- Specialized search capabilities that MySQL full-text search cannot provide efficiently.

| Requirement | MySQL | Elasticsearch |
|---|---|---|
| Transactional source of truth | Excellent | Usually not the primary choice |
| Exact lookup and relational joins | Excellent | Less suitable for relational joins |
| Basic full-text search | Good | Good |
| Advanced relevance and language analysis | More limited | Strong |
| Faceted search and search-oriented aggregations | Suitable for simpler cases | Strong |
| Immediate consistency with transactional updates | Easier within the same database transaction | Usually eventually consistent when synchronized asynchronously |

A common architecture is to keep authoritative product and order data in MySQL and maintain a searchable projection in Elasticsearch.

For synchronization, I prefer a **transactional outbox** or a reliable change-data-capture pipeline. The application commits the business change and an outbox event in the same MySQL transaction; a worker then updates the search index.

I also design for retries, idempotency, delete propagation, and reindexing. Since search updates may lag behind database commits, the application must tolerate eventual consistency. I do not treat Elasticsearch as the authoritative source for inventory or financial transactions.

## 5. Spatial data and location queries

MySQL supports spatial data types based on geometry, including:

- `POINT` — a single location.
- `LINESTRING` — a path or route.
- `POLYGON` — an area or boundary.
- `MULTIPOINT`, `MULTILINESTRING`, and `MULTIPOLYGON` — collections of geometries.

These are useful for nearby-store discovery, delivery zones, geofencing, and geographic filtering.

### Example: storing restaurant coordinates

```sql
CREATE TABLE restaurant_locations (
    restaurant_id BIGINT PRIMARY KEY,
    location POINT NOT NULL SRID 4326,
    SPATIAL INDEX idx_location (location)
);
```

`SRID 4326` identifies the WGS 84 geographic coordinate reference system commonly used by GPS.

When constructing a point from longitude and latitude, I use the coordinate order expected by the spatial function. With an SRID-aware geographic point, I can explicitly use longitude-latitude order:

```sql
SET @my_location = ST_GeomFromText(
    'POINT(77.2090 28.6139)',
    4326,
    'axis-order=long-lat'
);
```

Here, the example coordinates represent a location in Delhi. Longitude comes first in the WKT point because I explicitly specify `axis-order=long-lat`.

### Find nearby restaurants

For example, calculate the distance from the user's location to each restaurant:

```sql
SELECT
    restaurant_id,
    ST_Distance(location, @my_location) AS distance_meters
FROM restaurant_locations
WHERE ST_Distance(location, @my_location) <= 5000
ORDER BY distance_meters;
```

For appropriate geographic spatial reference systems, `ST_Distance()` returns distance in the SRS's associated linear unit—meters for SRID 4326 in MySQL's supported geographic calculations. I verify the function behavior for the actual MySQL version and SRID.

This is a clear example, but applying a distance function to every row can be expensive on a large dataset. I use spatial-index-compatible bounding or distance predicates where appropriate, inspect the query plan, and then calculate exact distances for candidates.

### Find locations inside a delivery zone

If delivery zones are represented as polygons, a spatial predicate can test whether a restaurant or customer point lies inside a zone:

```sql
SELECT restaurant_id
FROM restaurant_locations
WHERE ST_Contains(@delivery_zone, location);
```

Here, `@delivery_zone` must be a polygon geometry with a compatible spatial reference system. Boundary behavior matters: `ST_Contains()` does not treat a point on the polygon boundary the same way as every other spatial predicate.

### Spatial-index considerations

A MySQL `SPATIAL` index uses an R-tree-based structure for supported spatial data. Spatial indexes require appropriate geometry columns and SRID handling; for geographic coordinates, an explicit SRID helps ensure correct geographic semantics and index eligibility.

I use spatial types when queries involve actual geometry, distance, intersection, containment, or geographic areas. For a simple application with only a few coordinates, ordinary latitude and longitude columns may be sufficient, but they do not offer the same spatial-query capabilities.

## 6. How I choose in practice

For a restaurant ERP, my approach would be:

1. **Core business data:** normalized tables for orders, order items, inventory, ingredients, and suppliers.
2. **Flexible metadata:** JSON for optional menu attributes that differ between products.
3. **Frequently filtered JSON properties:** generated columns or functional indexes after measuring query performance.
4. **Basic menu search:** MySQL `FULLTEXT` for product names and descriptions.
5. **Advanced typo-tolerant search:** Elasticsearch if the search requirements justify the additional infrastructure and synchronization complexity.
6. **Nearby restaurants and delivery zones:** spatial types and spatial predicates, with query-plan verification.

The main principle is to select the simplest data model and index strategy that satisfies the actual query patterns. JSON provides flexibility, full-text indexing provides word-based retrieval, and spatial indexing provides geometry-aware queries. None is a substitute for proper relational design, workload measurement, or `EXPLAIN ANALYZE`.

*Q21. Multi-tenancy and SaaS data design in MySQL. Compare shared-schema (tenant_id), schema-per-tenant and database-per-tenant on isolation, migrations, noisy neighbours, backups and index design (tenant_id as the leading column of composite indexes). How do you prevent cross-tenant leaks, and what’s your plan when one tenant grows huge?*

Multi-tenancy means multiple customers (tenants) share the same SaaS application and infrastructure while their data remains logically or physically isolated. The main design decision is how to partition tenant data: shared tables, separate schemas, or separate databases.

For a typical SaaS product such as a restaurant ERP, I would usually start with a **shared schema using `tenant_id` on every tenant-owned record**. It is cost-effective, operationally simpler, and easy to scale initially. However, I would enforce tenant isolation throughout the application and database access patterns rather than relying on developers to remember a filter in every query.

## 1. Three multi-tenancy models

### Model A: Shared schema — shared tables

All tenants use the same database and tables. Each tenant-owned row has a `tenant_id`.

```sql
CREATE TABLE orders (
    id BIGINT NOT NULL AUTO_INCREMENT,
    tenant_id BIGINT NOT NULL,
    order_number VARCHAR(50) NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (id),
    UNIQUE KEY uq_tenant_order (tenant_id, order_number),
    KEY idx_tenant_created (tenant_id, created_at)
);
```

Tenant A and Tenant B's orders coexist in the same table, but queries are scoped to the authenticated tenant.

**Advantages**
- Lowest infrastructure and operational overhead.
- One schema migration updates all tenants.
- Efficient resource utilization and connection pooling.
- Easy to run cross-tenant operational reports when authorized.

**Disadvantages**
- A missing `tenant_id` filter can expose another tenant's data.
- Large tenants can dominate shared resources.
- Indexes must account for tenant-scoped query patterns.
- Backing up or restoring only one tenant is more complicated.

This is my default for a startup or small-to-medium SaaS product with many tenants.

### Model B: Schema-per-tenant

Each tenant gets its own logical database namespace containing its tables. In MySQL, a schema is effectively synonymous with a database.

For example:

```text
tenant_101.orders
tenant_101.order_items

tenant_202.orders
tenant_202.order_items
```

**Advantages**
- Better namespace separation and simpler tenant-specific exports.
- Tenant-specific schema changes can be isolated when necessary.
- Less risk of accidental cross-tenant queries caused by forgetting a row filter.

**Disadvantages**
- Migrations must run across many schemas.
- Thousands of schemas create operational and metadata-management overhead.
- Connection management, monitoring, backups, and schema-version tracking become more complex.
- It is not a complete security boundary: database privileges, application routing, and administrative access still matter.

I would consider this model when tenants require some structural customization but separate database servers are not yet necessary.

### Model C: Database-per-tenant

Each tenant has a separate MySQL database, potentially on a different server or cluster.

```text
mysql-cluster-a
  ├── tenant_101_db
  └── tenant_202_db

mysql-cluster-b
  └── tenant_303_db
```

**Advantages**
- Stronger operational isolation and a smaller failure or maintenance domain.
- Tenant-specific backup, restore, migration, and retention policies are easier to implement.
- Large tenants can be moved to dedicated hardware or clusters.
- Separate credentials and grants can restrict access to individual tenant databases.

**Disadvantages**
- Higher infrastructure and administrative costs.
- Provisioning, migrations, connection pools, and monitoring become more complicated.
- Cross-tenant reporting requires aggregation across databases.
- A shared database server can still cause resource contention, even if each tenant has a separate database.

Separate databases on the same MySQL server do not provide the same resource isolation as separate servers or clusters.

## 2. Comparison at a glance

| Concern | Shared tables | Schema-per-tenant | Database-per-tenant |
|---|---|---|---|
| Isolation | Logical, enforced by tenant keys and access controls | Namespace separation | Stronger database-level separation when privileges are configured correctly |
| Migration | One schema, all tenants | Run across schemas | Run across tenant databases |
| Noisy neighbours | Highest shared-workload risk | Still possible on shared server | Reduced with dedicated resources; not guaranteed on shared hardware |
| Backup/restore | Tenant-specific restore is complex | Tenant export/restore is more manageable | Tenant-specific backup and restore are more natural |
| Index design | Usually tenant-aware composite indexes | Indexes are local to each tenant's tables | Indexes are local to each tenant's tables |
| Cost | Lowest | Medium operational overhead | Highest, especially with dedicated infrastructure |
| Best fit | Many standard tenants | Tenants needing separate namespaces | Enterprise, regulated, or very large tenants |

A key distinction: **data separation and resource isolation are different problems**. Separate schemas do not prevent a tenant from consuming excessive CPU, memory, disk I/O, or connections on the same server.

## 3. Composite indexes and `tenant_id` as the leading column

In a shared-table design, most business queries filter by tenant and then by another attribute. My indexes should reflect those query patterns.

Consider:

```sql
SELECT id, total_amount, created_at
FROM orders
WHERE tenant_id = 101
  AND created_at >= '2026-10-01'
ORDER BY created_at DESC
LIMIT 50;
```

A suitable index is:

```sql
CREATE INDEX idx_orders_tenant_created
ON orders (tenant_id, created_at);
```

The index first narrows the search to one tenant and then supports a range on `created_at`. Because the second column matches the ordering, MySQL may also use the index to satisfy the ordering efficiently, depending on the plan and index definition.

### Why column order matters

For a composite B-tree index on `(tenant_id, created_at)`, MySQL can efficiently use the leftmost prefix:

- `WHERE tenant_id = ?`
- `WHERE tenant_id = ? AND created_at >= ?`
- `WHERE tenant_id = ? ORDER BY created_at`

It generally cannot use that index as efficiently for a query filtering only by `created_at`, because the leading `tenant_id` is unconstrained.

For another common query:

```sql
SELECT id, status, created_at
FROM orders
WHERE tenant_id = 101
  AND status = 'pending'
ORDER BY created_at DESC;
```

I might add:

```sql
CREATE INDEX idx_orders_tenant_status_created
ON orders (tenant_id, status, created_at);
```

I would not blindly add every possible composite index. Each index consumes disk space and buffer-pool memory and increases the cost of inserts, updates, and deletes. I would use real query patterns and `EXPLAIN ANALYZE` to determine which indexes are justified.

### Tenant-scoped uniqueness

Uniqueness usually needs to be defined within a tenant, rather than globally.

```sql
CREATE UNIQUE INDEX uq_tenant_order_number
ON orders (tenant_id, order_number);
```

This permits two different tenants to use the same order number while preventing duplicates within one tenant.

The same principle applies to business identifiers such as SKU codes, employee codes, and menu-item codes when those identifiers are tenant-specific.

## 4. Preventing cross-tenant data leaks

I treat tenant isolation as an authorization invariant, not just a query convention.

### A. Resolve the tenant from trusted authentication context

After authentication, the backend determines which tenant the user is authorized to access. I do not trust a client-provided `tenant_id` as proof of access.

For example, a request containing:

```json
{
  "tenant_id": 202,
  "order_id": 9001
}
```

must not allow a user authenticated for Tenant 101 to access Tenant 202 merely by changing the request body.

If users can legitimately belong to multiple tenants, the backend validates that the requested tenant is among their authorized memberships.

### B. Scope every read and write

Unsafe:

```sql
SELECT *
FROM orders
WHERE id = ?;
```

Safer:

```sql
SELECT *
FROM orders
WHERE tenant_id = ?
  AND id = ?;
```

The `tenant_id` parameter must come from the server's validated tenant context.

I apply the same rule to `UPDATE`, `DELETE`, exports, background jobs, and bulk operations:

```sql
UPDATE orders
SET status = 'cancelled'
WHERE tenant_id = ?
  AND id = ?;
```

For sensitive operations, I also verify the affected-row count and handle missing or unauthorized records consistently.

### C. Include tenant identity in relationships

A simple foreign key to `orders(id)` guarantees that the referenced order exists, but it does not itself prove that an order belongs to the same tenant as its child record.

For stronger database-level protection, I can include `tenant_id` in the relationship:

```sql
CREATE TABLE orders (
    tenant_id BIGINT NOT NULL,
    id BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, id)
);

CREATE TABLE order_items (
    tenant_id BIGINT NOT NULL,
    order_id BIGINT NOT NULL,
    id BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, id),

    CONSTRAINT fk_item_order
        FOREIGN KEY (tenant_id, order_id)
        REFERENCES orders (tenant_id, id)
);
```

Now a child row cannot reference an order under a different tenant key. This pattern adds key width and schema complexity, so I apply it deliberately, especially where cross-tenant relationship mistakes would be serious.

### D. Centralize data access and authorization

I avoid scattering tenant filtering throughout controller code. Instead, I enforce tenant scope through service and repository patterns, tenant-aware query builders, or other centrally controlled data-access mechanisms.

I also:
- Restrict database credentials to required operations and schemas.
- Validate tenant ownership on object-level access, including nested resources.
- Scope caches by tenant, for example `tenant:101:order:9001`, rather than using globally ambiguous cache keys.
- Include tenant context in queue messages, scheduled jobs, search-index documents, and object-storage paths.
- Test exports, file downloads, websocket subscriptions, and background processing for cross-tenant leaks.
- Log tenant context for auditing without exposing sensitive data.

MySQL does not provide PostgreSQL-style built-in row-level security policies. In a shared-table MySQL design, I therefore rely on carefully enforced application authorization, database constraints, privileges, and testing rather than assuming the database automatically filters rows by tenant.

## 5. Handling migrations safely

With shared tables, a migration is a change affecting all tenants. I prefer backward-compatible, staged migrations for production systems.

For example, if I need to add a new non-null business attribute:

1. Add the column in a way compatible with the currently deployed application.
2. Deploy code that can handle both old and new data.
3. Backfill existing records in controlled batches.
4. Verify completeness and correctness.
5. Enforce stricter constraints if required.
6. Remove old compatibility code in a later deployment.

For schema-per-tenant or database-per-tenant, I track a schema version per tenant, run migrations in batches, and record failures so they can be retried safely. I avoid assuming every tenant database has successfully migrated just because the deployment completed.

For large tables, I assess the locking and table-rebuild behavior of the intended DDL. If necessary, I use an appropriate online schema-change strategy and validate its impact before production execution.

## 6. Backups and tenant-specific restoration

A full shared-database backup is straightforward operationally, but restoring one tenant without affecting others is harder.

For shared tables, I would plan tenant-specific exports or a recovery process that restores a backup into a separate environment, extracts the tenant's records and dependencies, and reinserts them in a controlled order. This must account for foreign keys, audit records, files, and related data.

For database-per-tenant, a single tenant database can often be restored independently, subject to the backup design and dependencies on shared services.

I distinguish **backup availability** from **tested recoverability**. I define recovery point objectives (RPO) and recovery time objectives (RTO), retain point-in-time recovery capability where needed, and regularly test restoration.

## 7. When one tenant becomes huge

I would not immediately redesign the entire SaaS architecture. I would first measure which resource is saturated: query latency, CPU, I/O, buffer-pool pressure, connection limits, write contention, or table size.

My scaling plan is progressive:

**Stage 1 — Optimize the existing tenant-aware workload**
- Fix inefficient queries and N+1 patterns.
- Add appropriate composite indexes.
- Archive or purge data according to retention rules.
- Paginate large results and batch background processing.
- Review connection-pool and concurrency limits.

**Stage 2 — Partition or separate large datasets where justified**

MySQL partitioning can help with certain maintenance and pruning patterns, such as time-based retention. It is not a general-purpose tenant-isolation feature, and partitioning has restrictions—for example, unique keys generally must include all columns used in the partitioning expression.

For a few very large tables, I may separate historical events, audit logs, or analytics data from hot transactional data.

**Stage 3 — Move the large tenant to a dedicated database or cluster**

I introduce a tenant-routing layer that maps each tenant to its assigned database:

```text
Tenant 101 ──► Shared MySQL cluster
Tenant 202 ──► Shared MySQL cluster
Tenant 303 ──► Dedicated MySQL cluster
```

The application continues to use the same domain services, while tenant placement changes through controlled configuration.

A migration plan would include:
1. Provision the target database and apply the correct schema version.
2. Copy the tenant's data and validate row counts, relationships, and checksums where practical.
3. Capture ongoing writes through a carefully designed change-data-capture or dual-write strategy.
4. Synchronize changes and verify consistency.
5. Pause or coordinate writes for the cutover, or use a rigorously tested online migration protocol.
6. Update tenant routing and monitor errors, latency, and replication lag.
7. Keep a rollback plan and retire the old copy only after validation.

I avoid naive dual writes because a failure between two database commits can leave the systems inconsistent. A controlled CDC/outbox-based migration or a short coordinated write pause is safer, depending on the application's availability requirements.

**Stage 4 — Scale read and analytical workloads independently**

Where appropriate, I use read replicas for read-heavy workloads, and separate analytical reporting from transactional traffic. Replicas introduce lag, so read-after-write operations and correctness-sensitive decisions may still need the primary.

## 8. Final design decision

For a restaurant ERP SaaS with many small and medium-sized customers, I would begin with shared tables, `tenant_id` on every tenant-owned record, tenant-aware composite indexes, centrally enforced authorization, and tested backups.

I would keep the code and provisioning architecture capable of assigning exceptional tenants to dedicated databases later. This gives me the operational efficiency of a shared platform without permanently locking every customer into the same physical database.

The most important point is that **a multi-tenant architecture must provide both logical data isolation and a credible path to resource isolation**. Composite indexes improve tenant-scoped queries; they do not prevent authorization bugs or noisy-neighbour problems. Those require separate controls.

*Q22. Operational troubleshooting scenarios. The server is at 100% CPU, replication is 10 minutes behind, Too many connections appears, or a migration is stuck on a metadata lock. For each, what do you check first (SHOW PROCESSLIST, performance_schema, SHOW REPLICA STATUS, long-running transactions), and what are the safe fixes?*

When troubleshooting MySQL in production, I follow a consistent process: **identify the bottleneck, determine the cause, apply the least disruptive fix, and verify recovery**. I avoid restarting MySQL or killing sessions as my first response because that can interrupt transactions, trigger recovery, and make the incident worse.

My first checks are usually:

```sql
SHOW FULL PROCESSLIST;
SHOW GLOBAL STATUS;
```

I also inspect `performance_schema`, database metrics, application logs, and operating-system CPU, memory, disk I/O, and network usage. The exact commands and available fields depend on the MySQL version.

## 1. Scenario: MySQL server is at 100% CPU

### Step 1: Identify what is consuming CPU

First, I distinguish database CPU saturation from CPU consumed by the application, background workers, or another process on the host.

At the operating-system level, I use tools such as:

```bash
top
pidstat -u 1
iostat -xz 1
```

`iostat` helps determine whether disk I/O is also a bottleneck. High CPU and high disk utilization can occur together, but they require different fixes.

Inside MySQL:

```sql
SHOW FULL PROCESSLIST;
```

I look for:
- Many concurrent expensive queries.
- Queries running for a long time.
- Large joins, sorting, grouping, or temporary-table operations.
- Repeated queries from the same application endpoint.
- Sessions waiting on locks rather than actively consuming CPU.

**Important:** a long-running query is not necessarily CPU-intensive. `SHOW PROCESSLIST` provides a quick overview, not a complete resource profile.

### Step 2: Find expensive query patterns

I inspect statement statistics in `performance_schema`:

```sql
SELECT
    DIGEST_TEXT,
    COUNT_STAR AS executions,
    ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_seconds,
    ROUND(AVG_TIMER_WAIT / 1e9, 2) AS avg_ms,
    SUM_ROWS_EXAMINED,
    SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

This helps identify query patterns with high cumulative execution time and excessive rows examined.

I also check `sys` views when available, such as `sys.statement_analysis`, and use `EXPLAIN ANALYZE` on a representative query when it is safe to execute.

Typical causes include:
- Missing or ineffective indexes.
- Full table scans on large tables.
- Poor join order or inefficient joins.
- Excessive sorting or temporary tables.
- N+1 queries or unexpectedly high query volume.
- Too much concurrency on expensive operations.

### Step 3: Apply a targeted fix

My preferred order is:

1. Stop or throttle the application workload responsible for the spike, if identified.
2. Optimize the worst query or correct a missing index after validating the execution plan.
3. Reduce unnecessary polling, duplicate requests, and excessive concurrency.
4. Apply caching or batch operations when appropriate.
5. Scale resources if the workload is legitimate and the system is already efficient.

I terminate a query only when I understand its impact and have selected the specific session to interrupt. Killing a connection may roll back its active transaction, which can itself consume substantial time and I/O.

**Interview takeaway:** I identify the highest-impact query or workload before changing server settings. Increasing CPU capacity or buffer-pool size will not fix a fundamentally inefficient query.

## 2. Scenario: Replication is 10 minutes behind

First, I establish whether the replica is receiving events slowly, applying them slowly, or intentionally falling behind due to resource contention.

### Step 1: Check replica status

On the replica, for MySQL 8.0.22 and later:

```sql
SHOW REPLICA STATUS\G
```

On older MySQL versions, the equivalent command is:

```sql
SHOW SLAVE STATUS\G
```

I examine:
- `Replica_IO_Running` and `Replica_SQL_Running` (or the legacy equivalents).
- `Seconds_Behind_Source` (or `Seconds_Behind_Master`).
- `Last_IO_Error` and `Last_SQL_Error`.
- Retrieved versus executed GTID sets when GTID replication is enabled.
- Relay-log and source-log positions.
- Replication filters and parallel-apply configuration.

`Seconds_Behind_Source` is useful, but it is not a perfect measurement of actual data freshness. It can be `NULL` when replication is stopped or the replica cannot determine the delay, and it may be misleading during certain disconnections or backlog patterns. I correlate it with replication worker status, GTID progress, and application-visible freshness.

### Step 2: Determine where the backlog originates

| Observation | Likely direction of investigation |
|---|---|
| Receiver thread stopped | Network connectivity, authentication, source availability, or replication configuration |
| Receiver running but retrieved events are not advancing | Source connection, network throughput, or source binlog availability |
| Receiver is current, but executed events lag | Replica apply capacity, long transactions, lock contention, or write workload |
| One replication worker is stuck | Transaction conflict, long-running statement, or a worker error |
| CPU or disk I/O is saturated on the replica | Resource bottleneck or insufficient apply throughput |

For MySQL versions that expose replication worker details, I inspect:

```sql
SELECT *
FROM performance_schema.replication_applier_status_by_worker\G
```

I also review `SHOW FULL PROCESSLIST`, InnoDB transaction state, and replica CPU/disk metrics. Where appropriate, I check `SHOW ENGINE INNODB STATUS` for lock waits and deadlocks.

### Step 3: Fix the actual cause

- **Long-running transaction on the source:** identify why it was long-running and reduce future transaction duration. A large transaction can create a large unit of replication work.
- **Replica is resource-constrained:** improve query/index efficiency, increase suitable resources, or reduce competing workloads.
- **Apply is serialized by contention:** inspect conflicting writes and replication worker behavior before changing parallel replication settings.
- **Network or receiver failure:** fix connectivity, credentials, or configuration and verify that the receiver resumes.
- **Replica workload is interfering:** move expensive reporting off the replica or throttle it.

I do not skip relay-log events or blindly reset replication to make the lag indicator disappear. That can permanently diverge replica data from the source.

If a replica is used for reads, I also consider routing freshness-sensitive reads to the primary until the replica catches up. I verify whether the replica is safe to use before directing production traffic back to it.

**Interview takeaway:** replication lag is a symptom. I distinguish event-retrieval problems from event-application problems before changing replication settings.

## 3. Scenario: `Too many connections`

MySQL has reached its configured connection limit, typically `max_connections`, or available connection capacity has otherwise been exhausted.

### Step 1: Inspect connection capacity

```sql
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Threads_running';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';

SHOW VARIABLES LIKE 'max_connections';
```

Interpretation:
- `Threads_connected`: currently open connections.
- `Threads_running`: threads actively running rather than simply sleeping.
- `Max_used_connections`: peak simultaneous connections since the server started or the status was reset.
- `max_connections`: configured connection limit.

I then inspect:

```sql
SHOW FULL PROCESSLIST;
```

I look for:
- Hundreds of sleeping sessions.
- Many active expensive queries.
- Sessions waiting on locks.
- Connections concentrated in one application instance.
- Connection leaks or unbounded connection creation.

Sleeping connections are not automatically problematic. A connection pool is expected to keep some connections open. The issue is whether the aggregate pool sizes and connection lifetimes exceed available server capacity.

### Step 2: Check application connection pools

Suppose an application runs 10 instances, each configured for a maximum of 100 database connections. The application alone could attempt to open 1,000 connections.

I calculate aggregate connection demand across:
- All application instances.
- Background workers and scheduled jobs.
- Administrative clients.
- Monitoring tools.
- Replication and other internal activity.

I also verify that connections are returned to the pool, transactions are not left open, and slow queries are not causing the pool to fill.

### Step 3: Recover safely

My response is:

1. Reduce or pause connection creation from the offending service.
2. Fix leaked connections, oversized pools, or excessively long requests.
3. Terminate only confirmed-abandoned or harmful sessions when necessary.
4. Increase `max_connections` only if memory, thread overhead, and overall server capacity support it.
5. Add application-side backpressure and bounded connection pools to prevent recurrence.

I do not set `max_connections` arbitrarily high. Each additional connection can consume resources, and allowing too many concurrent queries can cause memory pressure and worsen CPU or I/O saturation.

For example, I might inspect idle connections older than a threshold:

```sql
SELECT
    ID,
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    STATE
FROM information_schema.PROCESSLIST
WHERE COMMAND = 'Sleep'
ORDER BY TIME DESC;
```

I investigate before terminating them; an idle connection can be a valid pooled connection. For an emergency, `KILL CONNECTION connection_id` disconnects a selected session, but it can interrupt a transaction and trigger rollback.

**Interview takeaway:** the durable fix is usually correct connection-pool sizing and workload control, not simply increasing the database connection limit.

## 4. Scenario: A migration is stuck on a metadata lock

MySQL uses metadata locks (MDLs) to coordinate access to table definitions. DDL such as `ALTER TABLE` may wait for an existing transaction or statement that holds a conflicting metadata lock.

A common incident looks like this:

1. Session A starts a transaction and reads from `orders`.
2. Session A remains open without committing.
3. Session B attempts `ALTER TABLE orders ...`.
4. Session B waits for a metadata lock.
5. New queries may queue behind the pending DDL, depending on the lock requests and server behavior.

The migration can appear stuck even though the actual blocker is another session.

### Step 1: Inspect process state

```sql
SHOW FULL PROCESSLIST;
```

I look for statements such as `Waiting for table metadata lock`, identify the affected table, and note which sessions are waiting versus executing.

Then I inspect open transactions:

```sql
SELECT
    trx_id,
    trx_mysql_thread_id,
    trx_started,
    trx_state,
    trx_query
FROM information_schema.innodb_trx
ORDER BY trx_started;
```

This reveals active InnoDB transactions and their start times. It does not reveal every possible metadata-lock holder, because a session can hold an MDL without having a currently active InnoDB transaction.

### Step 2: Identify metadata-lock owners and waiters

On supported MySQL 8.0 installations with the relevant Performance Schema instrumentation enabled, I query:

```sql
SELECT
    ml.OBJECT_SCHEMA,
    ml.OBJECT_NAME,
    ml.LOCK_TYPE,
    ml.LOCK_DURATION,
    ml.LOCK_STATUS,
    t.PROCESSLIST_ID,
    t.PROCESSLIST_USER,
    t.PROCESSLIST_TIME,
    t.PROCESSLIST_INFO
FROM performance_schema.metadata_locks AS ml
LEFT JOIN performance_schema.threads AS t
    ON ml.OWNER_THREAD_ID = t.THREAD_ID
WHERE ml.OBJECT_SCHEMA = 'my_database'
  AND ml.OBJECT_NAME = 'orders';
```

I look for `PENDING` locks and correlate them with granted locks, thread IDs, and the transaction or statement holding the lock. I may use `sys.schema_table_lock_waits` when available because it can make blocker/waiter relationships easier to inspect.

The exact available columns and instrumentation depend on the MySQL version and configuration. I ensure the relevant metadata-lock instrumentation is enabled before relying on the output.

### Step 3: Resolve the blocker carefully

My preferred response is:

1. Identify the oldest or otherwise relevant blocking transaction.
2. Contact the owning application or service if it is safe to do so.
3. Commit or roll back the transaction through its normal owner where possible.
4. If necessary, terminate the specific blocking session after assessing the impact.
5. Reassess the migration and application queue before retrying.

For example:

```sql
KILL CONNECTION 12345;
```

This is an illustrative command only; I would replace `12345` with the verified blocker connection ID.

Killing the waiting migration connection is different from killing the blocker. Killing the migration may release its queued metadata-lock request, but it does not resolve the underlying long-running transaction that caused the wait.

### Step 4: Prevent recurrence

- Keep transactions short and avoid leaving interactive transactions open.
- Run DDL during an appropriate deployment window.
- Check for long-running transactions before high-impact schema changes.
- Use online DDL options where supported and appropriate.
- Set a deliberate `lock_wait_timeout` for metadata-lock acquisition when a migration should fail rather than wait indefinitely.
- Use a controlled online schema-change tool when native DDL locking or table-rebuild behavior is unsuitable.

Online DDL is not lock-free: even an online operation may need brief exclusive metadata locks at the start or end. I assess the specific operation, table size, MySQL version, and workload before choosing a migration strategy.

**Interview takeaway:** a metadata-lock wait is often caused by an older transaction, not by a defective migration. Find the blocker before killing sessions or retrying the DDL.

## 5. A concise incident-response checklist

| Incident | First checks | Safe initial action |
|---|---|---|
| CPU at 100% | OS process metrics, `SHOW FULL PROCESSLIST`, statement digests, query plans | Identify and reduce the most expensive workload |
| Replication 10 minutes behind | `SHOW REPLICA STATUS`, worker status, GTID progress, replica CPU/I/O | Determine whether receiving or applying events is behind |
| `Too many connections` | `Threads_connected`, `Threads_running`, `Max_used_connections`, process list, pool metrics | Stop connection growth and correct pool or leak behavior |
| Migration waiting for MDL | Process list, `innodb_trx`, `performance_schema.metadata_locks`, lock-wait views | Identify and safely resolve the actual blocker |

## 6. What I verify after recovery

I do not consider the incident resolved merely because the error disappears. I verify:
- Query latency, CPU, and disk I/O have returned to an acceptable range.
- Replication lag is declining and eventually reaches the expected level.
- Connection usage has headroom under the configured limit.
- The migration completed and the schema is in the expected state.
- Application error rates and data consistency remain acceptable.

Finally, I preserve relevant metrics, slow-query samples, process information, and incident timestamps. I use those observations to address the root cause—whether it is query inefficiency, resource contention, connection-pool configuration, transaction management, or deployment sequencing—rather than relying on repeated emergency interventions.

*Q23. Query execution lifecycle and the optimizer. What happens between sending a query and getting rows (parser, preprocessor, cost-based optimizer, executor, storage engine API)? How does the optimizer estimate costs from statistics, and how do you use EXPLAIN FORMAT=TREE or the optimizer trace to see why it chose a bad plan?*

# MySQL Query Execution Lifecycle and the Optimizer — Interview Answer

When I send a SQL query to MySQL, the server does not immediately execute it as written. It parses the SQL, validates its meaning, chooses an execution plan using the optimizer, and then executes that plan through the storage engine API. The storage engine, typically InnoDB, handles the underlying data and index access.

The key point is that **the optimizer chooses a plan based on estimated costs, not guaranteed actual execution time**. If its cardinality estimates are wrong, it can choose a plan that is much slower than an alternative.

## 1. Query execution lifecycle

<text color="secondary" size="sm">Simplified lifecycle for a typical MySQL SELECT query</text>

<box border radius="xl" padding={3} gap={1} align="center">
  <box background="surface-secondary" radius="md" padding={3} width="100%" align="center">
    **1. Client sends SQL**

    <text size="sm" color="secondary">Connection, authentication, protocol handling</text>
  </box>
  <icon name="arrow-down" size="lg" color="secondary" />
  <box background="surface" border radius="md" padding={3} width="100%" align="center">
    **2. Parser and preprocessor**

    <text size="sm" color="secondary">Syntax, names, columns, types, permissions</text>
  </box>
  <icon name="arrow-down" size="lg" color="secondary" />
  <box background="surface" border radius="md" padding={3} width="100%" align="center">
    **3. Query rewrite and optimization**

    <text size="sm" color="secondary">Transformations, join order, access paths, cost estimates</text>
  </box>
  <icon name="arrow-down" size="lg" color="secondary" />
  <box background="surface" border radius="md" padding={3} width="100%" align="center">
    **4. Execution plan and executor**

    <text size="sm" color="secondary">Run iterators and request rows as needed</text>
  </box>
  <icon name="arrow-down" size="lg" color="secondary" />
  <box background="surface" border radius="md" padding={3} width="100%" align="center">
    **5. Storage engine API → InnoDB**

    <text size="sm" color="secondary">Index lookups, row access, MVCC, locks and I/O</text>
  </box>
  <icon name="arrow-down" size="lg" color="secondary" />
  <box background="surface-secondary" radius="md" padding={3} width="100%" align="center">
    **6. Rows returned to the client**

    <text size="sm" color="secondary">Filtering, projection, sorting and protocol output as required</text>
  </box>
</box>

This is a conceptual model rather than a strictly linear implementation. MySQL can transform queries during optimization, and execution can interleave row retrieval with filtering, joining, and output.

### Stage 1: Connection and request handling

The client sends a SQL statement through the MySQL client/server protocol. The server handles the connection, authentication, session context, and relevant permissions.

For prepared statements, parsing and preparation can be separated from parameter binding and repeated execution. The exact amount of work reused depends on the statement type and server behavior.

### Stage 2: Parsing and semantic validation

The parser checks SQL grammar and builds an internal representation of the statement.

For example:

```sql
SELECT o.id, o.total_amount
FROM orders AS o
WHERE o.tenant_id = 101
  AND o.status = 'pending';
```

The preprocessor and related validation steps resolve table and column references, check types and permissions, and detect errors such as an unknown column.

At this point, MySQL understands what the query means. It has not necessarily decided how to retrieve the rows.

### Stage 3: Query rewriting and optimization

The optimizer considers equivalent ways to execute the query. Depending on the query and MySQL version, transformations can include:
- Simplifying predicates and constant expressions.
- Merging or materializing derived tables and CTEs where appropriate.
- Reordering joins.
- Choosing between table scans, index lookups, and range scans.
- Considering semijoin strategies for certain `IN` and `EXISTS` queries.
- Choosing whether sorting, grouping, or temporary structures are needed.

The optimizer attempts to minimize estimated execution cost while preserving SQL semantics.

### Stage 4: Execution

The executor runs the selected plan. In modern MySQL versions, execution is represented through iterator-based operations for many query types. Operators request rows from child operators, apply conditions, and produce results.

For example, a nested-loop join repeatedly retrieves matching rows from an inner access path for rows produced by the outer access path.

### Stage 5: Storage engine API

The server delegates data access to the selected storage engine through the handler/storage-engine interface.

With InnoDB, this may involve:
- Traversing B-tree indexes.
- Fetching clustered-index records.
- Checking row visibility using MVCC.
- Acquiring locks when required by the statement and isolation level.
- Reading pages from the buffer pool or storage.
- Maintaining transactional consistency.

The optimizer decides the access strategy, while InnoDB implements the physical data access and transactional behavior.

### Stage 6: Result production

The server applies any remaining operations—such as filtering, sorting, aggregation, or projection—and sends result rows to the client.

Some operations can stream rows incrementally; others, such as certain sorts and aggregations, may need to consume substantial input before producing final results.

---

## 2. How the cost-based optimizer chooses a plan

The optimizer estimates the work required by alternative plans and chooses a low-cost plan.

Its cost model considers factors such as:
- Estimated rows examined and produced.
- Index and table access costs.
- Expected page reads and memory-cache behavior.
- CPU work for predicates, joins, and other operations.
- Sorting and temporary-table costs.
- The cost of repeatedly executing an inner operation in a nested-loop join.

These are model-based estimates, not precise predictions of elapsed time. Actual latency also depends on concurrency, cache state, storage performance, and other runtime conditions.

### Example: choosing an index

Suppose `orders` contains 10 million rows:

```sql
SELECT id, total_amount
FROM orders
WHERE tenant_id = 101
  AND status = 'pending';
```

Assume there is an index on `(tenant_id, status)`.

The optimizer might estimate that the index matches 2,000 rows. An index access path may then be cheaper than scanning the entire table.

But suppose its statistics estimate 1.5 million matching rows. A table scan may appear cheaper, especially if many matching rows require fetching records from the clustered index.

The choice depends on estimated cardinality, row-access costs, and the selected plan. A table scan is not inherently wrong; it can be the correct choice when a large fraction of the table must be read.

### Cardinality and selectivity

**Cardinality** estimates how many rows an access path or intermediate result will produce.

**Selectivity** describes the fraction of rows satisfying a predicate. A highly selective condition matches a small fraction of rows.

For example, a unique order ID typically has high selectivity, whereas a status such as `active` may match a large fraction of a table.

MySQL estimates these values using information such as:
- Index statistics and estimated distinct values.
- InnoDB persistent statistics.
- Histograms, when created and applicable.
- Predicate characteristics and available metadata.

InnoDB statistics are estimates and can become inaccurate after substantial data changes or when the data distribution is skewed.

### Refresh statistics

When the optimizer appears to choose an implausible plan, I first check whether its statistics reflect the current data.

```sql
ANALYZE TABLE orders;
```

This refreshes table/index statistics. It can help after substantial data changes, but it does not guarantee that every selectivity estimate becomes accurate. The operation itself also has operational costs, so I plan it appropriately for production.

For skewed non-indexed columns, histograms may improve estimates:

```sql
ANALYZE TABLE orders
UPDATE HISTOGRAM ON status
WITH 100 BUCKETS;
```

Histograms help the optimizer estimate distributions for supported predicates, but they do not replace indexes and are not equally useful for every query shape. I verify whether the resulting plan actually improves.

---

## 3. Use `EXPLAIN FORMAT=TREE`

`EXPLAIN` shows the plan MySQL expects to use without running the statement normally. `FORMAT=TREE` presents the plan as a hierarchy of operations, making join order and operator relationships easier to understand.

```sql
EXPLAIN FORMAT=TREE
SELECT o.id, o.total_amount
FROM orders AS o
WHERE o.tenant_id = 101
  AND o.status = 'pending'
ORDER BY o.created_at DESC
LIMIT 50;
```

An illustrative plan might resemble:

```text
Limit: 50 row(s)
  -> Sort: o.created_at DESC
      -> Filter: (o.status = 'pending')
          -> Index lookup on o using idx_tenant
             (tenant_id = 101)
```

This is an example of the shape of a plan, not output guaranteed for this query. Actual output depends on the schema, indexes, data distribution, and MySQL version.

I inspect the plan from the leaves upward:

1. **Access method:** Is MySQL using a table scan, index lookup, or range scan?
2. **Estimated rows:** Does each operator produce a plausible number of rows?
3. **Filter placement:** Is filtering happening early enough to reduce downstream work?
4. **Join order:** Does the plan start with a selective input, where appropriate?
5. **Sorting and materialization:** Is a sort or temporary intermediate result avoidable?
6. **Repeated work:** Could an inner operation be executed many times in a nested-loop join?

For the example above, an index beginning with `(tenant_id, status, created_at)` might allow MySQL to locate the relevant tenant and status rows in order, potentially avoiding an explicit sort.

However, I would not add that index automatically. I would first confirm the real workload, the plan, the write overhead, and whether the index's column order matches other important queries.

### `EXPLAIN` versus `EXPLAIN ANALYZE`

These are different diagnostic tools.

```sql
EXPLAIN FORMAT=TREE
SELECT ...;
```

Shows the planned operations and estimated cardinalities without executing the query normally.

```sql
EXPLAIN ANALYZE
SELECT ...;
```

Executes the query and reports actual runtime information alongside estimates, including actual rows and timing for operators.

For example, a plan may estimate 100 rows but actually produce 500,000. That discrepancy is a strong signal that the optimizer's assumptions are wrong or that an intermediate operation is far more expensive than expected.

**Caution:** `EXPLAIN ANALYZE` really executes the statement. On a large production query, that can consume substantial CPU, memory, and I/O. I use it on a representative workload when the execution risk is acceptable, and I avoid using it casually on destructive statements.

---

## 4. How I investigate a bad execution plan

Consider a tenant-scoped query:

```sql
SELECT id, total_amount, created_at
FROM orders
WHERE tenant_id = 101
  AND status = 'pending'
ORDER BY created_at DESC
LIMIT 50;
```

Suppose it is slow. My diagnostic sequence is:

**Step 1 — Capture the plan.**

```sql
EXPLAIN FORMAT=TREE
SELECT id, total_amount, created_at
FROM orders
WHERE tenant_id = 101
  AND status = 'pending'
ORDER BY created_at DESC
LIMIT 50;
```

**Step 2 — Check actual behavior where safe.**

```sql
EXPLAIN ANALYZE
SELECT id, total_amount, created_at
FROM orders
WHERE tenant_id = 101
  AND status = 'pending'
ORDER BY created_at DESC
LIMIT 50;
```

I compare estimated and actual rows and inspect the operator timings.

**Step 3 — Review existing indexes.**

```sql
SHOW INDEX FROM orders;
```

I check the leading columns, cardinality estimates, index width, and whether the index supports both filtering and ordering.

**Step 4 — Refresh or improve statistics.**

```sql
ANALYZE TABLE orders;
```

If the status distribution is heavily skewed, I evaluate whether a histogram is useful.

**Step 5 — Test a better access path.**

I may test a composite index such as:

```sql
CREATE INDEX idx_orders_tenant_status_created
ON orders (tenant_id, status, created_at);
```

With equality conditions on the first two columns, the third column can support ordering by `created_at` in the same direction for this query pattern, subject to the actual plan and engine behavior.

I test before retaining the index because additional indexes increase storage and write costs.

---

## 5. Use optimizer trace to understand *why* MySQL chose a plan

`EXPLAIN` tells me which plan was selected. When I need to understand why the optimizer preferred it over alternatives, I use the optimizer trace.

For a session-level investigation:

```sql
SET optimizer_trace = 'enabled=on';

SELECT id, total_amount
FROM orders
WHERE tenant_id = 101
  AND status = 'pending';

SELECT TRACE
FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE\G

SET optimizer_trace = 'enabled=off';
```

I execute the target statement after enabling tracing, then inspect the trace generated for it. Trace availability and detail depend on the MySQL version, configuration, and trace limits.

I look for:
- Candidate access paths that were considered.
- Estimated rows and costs for those paths.
- Why a candidate index was rejected.
- Range-analysis results.
- Join-order alternatives and their costs.
- Predicate selectivity and transformations.
- Whether an index was considered unsuitable for ordering or filtering.

For example, the trace may show that a table scan had a lower estimated cost than an index lookup because the optimizer expected many matching rows. That points me toward checking cardinality statistics, selectivity, and the real distribution of the data rather than immediately forcing the index.

Optimizer trace can be verbose and has overhead, so I use it for targeted investigation rather than leaving it enabled indiscriminately.

### Should I force an index?

I treat hints such as `FORCE INDEX` as a last resort or a controlled experiment, not the default fix.

Forcing an index can improve one query on today's data while making it slower after the table grows or the distribution changes. I first investigate statistics, index design, query shape, and actual execution. If a hint is justified, I document the reason and monitor it.

---

## 6. Common reasons optimizer estimates go wrong

| Problem | What I investigate |
|---|---|
| Stale statistics | Run `ANALYZE TABLE` and compare the plan |
| Skewed data distribution | Histograms, predicate selectivity, and actual row counts |
| Missing composite index | Query filters, join columns, and ordering |
| Correlated predicates | Whether independent selectivity assumptions are misleading |
| Large intermediate join result | Join order, cardinality estimates, and join conditions |
| Function applied to indexed column | Whether the expression prevents ordinary index use |
| Parameter-sensitive data distribution | Whether different values produce very different row counts and plans |
| Sorting or materialization | Whether a compatible index or query rewrite can avoid the extra work |

Not every bad plan is caused by stale statistics. Sometimes the schema lacks an appropriate index, the query requires reading a large proportion of the table, or the optimizer's cost model cannot accurately capture the workload.

## 7. Interview summary

I explain the lifecycle as **parse and validate → rewrite and optimize → execute → access data through the storage engine → return results**.

The optimizer chooses among candidate plans using estimated cardinalities and costs. When the plan performs poorly, I use `EXPLAIN FORMAT=TREE` to understand the selected operators, `EXPLAIN ANALYZE` to compare estimates with actual execution, and `optimizer_trace` to investigate the alternatives and cost decisions behind the selection.

My goal is to fix the underlying cause—incorrect estimates, ineffective indexes, an expensive query shape, or an unsuitable access path—rather than blindly forcing indexes or increasing server resources.