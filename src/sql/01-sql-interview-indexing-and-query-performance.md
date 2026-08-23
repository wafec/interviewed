---
layout: default
title: "SQL Interview: Indexing & Query Performance"
---

# SQL Interview — Indexing & Query Performance

Twenty questions on SQL indexing and query performance: how indexes work
internally (B-trees, composite indexes, covering indexes), how to detect and
diagnose a slow query in production (EXPLAIN plans, statistics, slow query
logs), locking/concurrency internals, and the trade-offs senior engineers are
expected to reason about (index type choice, isolation levels, clustered vs
secondary indexes, when *not* to index).

### Q1. What is a database index, and why does it make reads faster?

**Question:**
What is a database index? Why does adding one to a column speed up queries
that filter on it?

**Good answer:**
An index is a separate, ordered data structure (most commonly a B-tree) that
the database maintains alongside a table, mapping column values to the
physical location (or primary key) of the rows that contain them. Without an
index, the database must perform a sequential scan — reading every row in
the table to check if it matches the query's condition, which is O(n).
With an index, the database can binary-search a balanced tree structure to
jump almost directly to the matching rows, which is roughly O(log n) for the
lookup, plus the cost of retrieving the matching rows themselves.

The trade-off is that an index isn't free: it has to be created (an
additional structure to write and maintain), it costs disk space, and every
`INSERT`/`UPDATE`/`DELETE` on the indexed column(s) now has to also update
the index, adding write overhead. So indexes should be added deliberately,
matched to actual query patterns, not blindly on every column.

**Code example:**
```sql
-- Without an index: full sequential scan of `orders`
SELECT * FROM orders WHERE customer_id = 42;

-- With an index, the planner can jump straight to matching rows
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

**Follow-up question:**
If a table is small (say, a few hundred rows), does adding an index still
help?

**Follow-up good answer:**
Usually not — and it can even be counterproductive. A table that small
typically fits in a handful of 8KB pages, all of which end up cached in
memory, so a sequential scan just reads a few pages in one cheap pass. Adding
an index costs real write overhead (as noted above) without a matching read
benefit, since the constant-factor cost of traversing index pages and then
following each match back to the heap can be *more* work than just scanning
everything directly when n is small. This is why the query planner is
cost-based rather than "always prefer an index if one exists" — it correctly
chooses a sequential scan over a usable index on small tables, because its
cost model accounts for page reads and I/O, not just algorithmic complexity.

**Glossary:**
- **B-tree** — a self-balancing tree data structure that keeps data sorted and allows searches, insertions, and deletions in O(log n).
- **Sequential scan (seq scan)** — reading every row of a table in physical order to evaluate a condition.
- **Selectivity** — the fraction of rows in a table that match a given condition; low selectivity (few matching rows) is where indexes help most.

**Mental model:**
This question checks whether the candidate understands indexes as a
time/space trade-off, not just "a thing you add to make queries fast." A
senior answer volunteers the write-cost side without being asked.

**References:**
- [PostgreSQL: Chapter 11. Indexes](https://www.postgresql.org/docs/current/indexes.html)

---

### Q2. How does a B-tree index actually work internally?

**Question:**
Walk me through how a B-tree index is structured internally, and why lookups
are O(log n).

**Good answer:**
A B-tree is a balanced tree where each node can hold multiple keys and
pointers to child nodes. All leaf nodes sit at the same depth, which is what
keeps the tree "balanced" — no branch is disproportionately deep. Internal
nodes act as a routing structure: given a search key, you compare it against
the keys in the root node to decide which child pointer to follow, then
repeat at each level until you reach a leaf node containing the actual
key → row-pointer (or key → primary-key, depending on the engine) entries.

Because each node holds many keys (not just 2 like a binary tree), the tree
stays very shallow even for huge tables — a table with millions of rows might
only need 3-4 levels of tree traversal to find a match, hence O(log n) with a
large base. Inserts/updates that don't fit in a node cause a **page split**:
the node is divided into two, and a new separator key is pushed up to the
parent, which can cascade upward and occasionally increases tree depth by
one level. This is also why heavy random-order inserts (e.g. UUIDv4 primary
keys) cause more page splits and index bloat than sequential inserts.

**Follow-up question:**
Why do B-trees support range queries (`BETWEEN`, `<`, `>`) well, but a hash
index does not?

**Follow-up good answer:**
A B-tree's leaf nodes are stored in sorted key order and linked together
(typically as a doubly-linked list), so once you've located the start of a
range via a single tree descent, you can walk sequentially forward (or
backward) through the leaves to collect every value in the range — the
sortedness is inherent to the structure. A hash index instead computes a
hash of the key and uses that hash to pick a storage bucket; a good hash
function deliberately scatters similar keys into unrelated buckets to keep
lookups uniform, which destroys any notion of "next value" — there's no way
to know which bucket holds the next-larger key without hashing every
candidate value, so a hash index can only answer "does this exact key
exist," not "what's near this key."

**Glossary:**
- **Page split** — when a B-tree node is full and a new key has to be inserted, causing the node to divide into two and a key to propagate to the parent.
- **Leaf node** — the bottom level of the tree, holding the actual indexed values and their row references; leaf nodes are typically linked together for efficient range scans.
- **Tree depth/height** — the number of levels from root to leaf; determines the worst-case number of node reads for a lookup.

**Mental model:**
Tests whether the candidate actually understands the data structure rather
than treating the index as a black box — this is the "explain it like you've
read the source" bar that separates people who've memorized "indexes make
things faster" from people who understand the mechanism.

**References:**
- [PostgreSQL: 11.2. Index Types (B-Tree)](https://www.postgresql.org/docs/current/indexes-types.html)

---

### Q3. How does column order matter in a composite (multicolumn) index?

**Question:**
You create an index on `(a, b, c)`. Does that index help a query that filters
only on `b`? What about a query filtering on `a` and `c` but not `b`?

**Good answer:**
For a B-tree composite index, column order follows the "leftmost prefix"
rule: the index is organized first by `a`, then within each `a` value by
`b`, then within each `(a, b)` pair by `c` — like a phone book sorted by
last name, then first name, then middle name.

- A query filtering only on `b` (skipping `a`) generally **cannot** use the
  index efficiently, because the index isn't sorted by `b` independent of
  `a` — the planner would have to scan the whole index.
- A query filtering on `a` and `c` but not `b` **can** use the index to
  narrow by `a` (the leftmost column), but the `c` condition can only be
  applied as a filter after that — it doesn't reduce the portion of the
  index scanned, since `c` isn't reachable without pinning `b` first.
- The general rule: equality constraints on leading columns narrow the scan
  range; a constraint on any column after the first "gap" only filters, it
  doesn't reduce work.

Because of this, the *order you declare columns in* should match your most
common/most selective query patterns, not the order they appear in the
`WHERE` clause of any one query.

**Code example:**
```sql
CREATE INDEX idx_orders_status_created ON orders (status, created_at);

-- Uses the index efficiently: status is the leading column
SELECT * FROM orders WHERE status = 'pending' AND created_at > now() - interval '1 day';

-- Cannot use the index efficiently: created_at alone skips the leading column
SELECT * FROM orders WHERE created_at > now() - interval '1 day';
```

**Follow-up question:**
When would you create two separate single-column indexes instead of one
composite index?

**Follow-up good answer:**
When your queries filter on those columns independently rather than
together — e.g. one hot query filters only on `a`, another filters only on
`c`, and no query filters on both at once. A composite `(a, c)` index would
only serve the `a`-only query efficiently (via the leftmost prefix rule) and
would be useless for the `c`-only query, so you'd need a separate index on
`c` anyway, making the composite index partly redundant. Two single-column
indexes are also more flexible when the *set* of columns queried varies
unpredictably, since PostgreSQL can combine multiple single-column indexes
via a bitmap AND/OR when a query filters on several of them together,
recovering some of the benefit a composite index would have given directly.

**Glossary:**
- **Leftmost prefix rule** — a composite B-tree index can only be used efficiently for query conditions that constrain a contiguous prefix of its columns, starting from the first.
- **Composite/multicolumn index** — a single index built over more than one column.

**Mental model:**
Column-order mistakes in composite indexes are one of the most common
real-world performance bugs. This question tests whether the candidate can
reason about *which* queries a given index actually serves, not just
"whether an index exists."

**References:**
- [PostgreSQL: 11.3. Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)
- [PostgreSQL: 11.5. Combining Multiple Indexes](https://www.postgresql.org/docs/current/indexes-bitmap-scans.html)

---

### Q4. What is a covering index / index-only scan?

**Question:**
What's an index-only scan, and how would you design an index to make one
happen for a specific hot query?

**Good answer:**
Normally, an index only tells the database *which rows* match a condition;
the engine still has to go to the table's heap/data pages to fetch the
other columns the query asked for (a "heap fetch"). An **index-only scan**
happens when every column the query needs — both the filter columns and the
selected columns — is already present in the index itself, so the engine can
answer the query using only the index, skipping heap access entirely. This
is a big win because it turns potentially many random heap-page reads into
sequential index reads.

To make this happen deliberately, you can build a **covering index**: an
index on the filter column(s), with additional "payload" columns attached
(e.g. via `INCLUDE` in PostgreSQL) so they ride along in the index without
being part of the search key.

In PostgreSQL specifically, an index-only scan additionally requires the
**visibility map** to confirm the relevant heap page's rows are all visible
to the current transaction (i.e., no pending vacuum work) — otherwise it
still has to check the heap for visibility, which negates most of the
benefit. That's why index-only scans work best on tables that are vacuumed
regularly and don't churn heavily.

**Code example:**
```sql
-- x is the search key, y rides along so this query never touches the heap
CREATE INDEX idx_tab_x_include_y ON tab (x) INCLUDE (y);

-- Answerable entirely from the index
SELECT y FROM tab WHERE x = 'key';
```

**Follow-up question:**
Why can a GIN index not support index-only scans the way a B-tree can?

**Follow-up good answer:**
An index-only scan requires the index to store (or be able to fully
reconstruct) the original column value for each entry, since the whole point
is to answer the query without visiting the heap. A B-tree entry always
holds a complete copy of the indexed value, so it qualifies. A GIN entry
typically holds only *part* of the original value — for example, GIN indexes
an array or `tsvector` column by creating one index entry per individual
element/lexeme, not one entry per row's full value — so no single index
entry (or even a small set of them) is guaranteed to let PostgreSQL
reconstruct the original composite value. Since it can't reliably answer
"what was the full original value in this row" from the index alone, it
can't skip the heap.

**Glossary:**
- **Heap** — the actual table storage holding full rows, as opposed to the index.
- **Covering index** — an index that includes all columns a query needs, so the query can be answered from the index alone.
- **Visibility map** — a PostgreSQL structure tracking whether all rows on a heap page are visible to every transaction, used to skip heap lookups during index-only scans.

**Mental model:**
Distinguishes candidates who know indexes only speed up *filtering* from
those who understand they can also eliminate heap I/O entirely — a common
"advanced" lever for hot-path query optimization.

**References:**
- [PostgreSQL: 11.9. Index-Only Scans and Covering Indexes](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)

---

### Q5. How do you read an `EXPLAIN ANALYZE` output to find a bad plan?

**Question:**
A query is slow. You run `EXPLAIN ANALYZE` on it. What are you looking for
in the output?

**Good answer:**
`EXPLAIN` alone shows the planner's *estimated* plan — cost, estimated row
count, and width — without running the query. `EXPLAIN ANALYZE` actually
executes the query and adds *actual* timing and row counts per plan node, so
you can compare estimate vs. reality.

Key things to look for:
1. **Seq Scan on a large table** where you'd expect an index scan — often
   means a missing or unusable index, or the planner decided a scan was
   cheaper (which can itself indicate a selectivity/statistics problem).
2. **Large gap between estimated and actual rows** — a strong signal that
   table statistics are stale (need `ANALYZE`) or the planner mis-estimated
   selectivity, which cascades into bad join order/join method choices.
3. **"Filter" vs "Index Cond"** — a condition applied as `Filter` means rows
   were fetched first and then filtered afterward (wasteful); `Index Cond`
   means the index itself narrowed the scan.
4. **Nested Loop over a large row count** — nested loops are fine for small
   inputs but degrade badly (effectively O(n·m)) when the outer row estimate
   was wrong.
5. **Buffers: read= vs hit=** (with `EXPLAIN (ANALYZE, BUFFERS)`) — high
   `read` (physical I/O) vs `hit` (cache) tells you whether you're actually
   I/O bound.

The overall goal is to find the plan node where actual time balloons and
work backward to why the planner chose that path.

**Code example:**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;
```

**Follow-up question:**
The planner estimated 10 rows but actually got back 500,000. What do you do
next?

**Follow-up good answer:**
That's a 50,000x misestimate, which is almost always the root cause of the
slow plan rather than a symptom of it — a plan that's efficient for 10 rows
(e.g. a nested loop) is disastrous for 500,000. First step is to run
`ANALYZE` on the table(s) involved and re-check the plan: if the estimate
becomes accurate, stale statistics were the culprit, and I'd check whether
autovacuum/auto-analyze is keeping up on that table. If the estimate is
still wildly off after a fresh `ANALYZE`, the likely cause is a condition the
planner structurally can't estimate well — e.g. correlated columns (the
planner assumes independence between conditions by default), a non-sargable
expression, or a value distribution too skewed for the default statistics
target — in which case I'd look at extended statistics
(`CREATE STATISTICS`) for correlated columns, increase
`default_statistics_target` on the relevant column, or rewrite the query to
be more directly estimable.

**Glossary:**
- **Plan node** — one step of the query execution plan (e.g. a scan, join, or sort).
- **Cost** — the planner's internal, unit-less estimate of how expensive a plan node is, used to compare candidate plans.
- **Cardinality estimation** — the planner's prediction of how many rows a given step will produce.

**Mental model:**
This is a direct proxy for "have you actually debugged a slow query in
production, or only in theory." The interviewer wants concrete diagnostic
vocabulary (Seq Scan, Filter vs Index Cond, actual vs estimated rows), not a
vague "I'd look at the query plan."

**References:**
- [PostgreSQL: 14.1. Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL: 14.2. Statistics Used by the Planner](https://www.postgresql.org/docs/current/planner-stats.html)

---

### Q6. How do you find your worst-performing queries in a live production database?

**Question:**
You're on call and the database is under heavy load. How do you identify
which queries are the actual problem, without guessing?

**Good answer:**
You want aggregate, historical evidence, not just "look at what's slow right
now." In PostgreSQL, the standard tool is the `pg_stat_statements`
extension: it tracks every distinct normalized query (constants replaced
with placeholders) along with call count, total/mean execution time, rows
returned, and I/O (cache hits vs. physical reads). You'd sort by
`total_exec_time DESC` to find what's costing the *cluster* the most time in
aggregate (a moderately slow query called 100,000 times/hour can hurt more
than a rare 10-second query), and separately by `mean_exec_time` to find
individually slow queries.

In MySQL, the equivalent is the **slow query log** (`slow_query_log`,
`long_query_time` threshold), summarized with `mysqldumpslow` or `pt-query-digest`.

Beyond the DB layer, you'd also correlate with an APM tool (e.g. traces) to
see if the slow query is even the bottleneck, or if it's queueing behind
lock contention, connection pool exhaustion, or a noisy-neighbor process —
so the methodology is: aggregate stats → identify top offenders → drill into
the specific query with `EXPLAIN ANALYZE` → check for lock waits →
correlate with app-level tracing before concluding root cause.

**Code example:**
```sql
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**Follow-up question:**
`pg_stat_statements` shows a query with high `total_exec_time` but low
`mean_exec_time`. What does that tell you, and what would you do about it?

**Follow-up good answer:**
High total time with low mean time means the query itself is individually
cheap, but it's being called an enormous number of times (`total_exec_time`
is effectively `calls × mean_exec_time`), so the aggregate cost to the
cluster is still large — "death by a thousand cuts" rather than one bad
query. This points away from query-plan tuning and toward reducing *call
volume*: the classic causes are an N+1 query pattern from an ORM, a
per-request query that could be batched/cached, or a polling loop hitting
the database far more often than necessary. Optimizing the query itself
(e.g. adding an index) would only shave a small amount off each call; the
bigger win is fixing the access pattern that causes so many calls in the
first place.

**Glossary:**
- **pg_stat_statements** — a PostgreSQL extension that aggregates execution statistics per normalized query.
- **Slow query log** — a MySQL feature logging queries that exceed a configured time threshold.
- **Query normalization** — replacing literal values in a query with placeholders so structurally identical queries are grouped together.

**Mental model:**
Tests real operational experience: does the candidate reach for
data/tooling first, or start guessing/eyeballing? This is the "which tools
do you use, what's your method" performance-diagnosis angle interviewers are
leaning on heavily right now.

**References:**
- [PostgreSQL: F.32. pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [MySQL 8.4 Reference Manual: 7.4.5 The Slow Query Log](https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html)

---

### Q7. Why do stale table statistics cause bad query plans?

**Question:**
A query was fast last month and is slow today, even though the query and
indexes haven't changed. What could cause that, and how would you confirm
it?

**Good answer:**
Query planners don't execute the query to decide how to run it — they
estimate cost using statistics about the data: row counts, most-common
values, and value-distribution histograms, gathered by `ANALYZE` (in
PostgreSQL) or equivalent statistics-gathering in other engines. If the
table's data has changed significantly (grown, or the value distribution
shifted) since the last `ANALYZE`, the planner's row estimates become wrong,
which can flip its decision — e.g. choosing a sequential scan when an index
scan would now be far cheaper, or picking a nested loop join when a hash
join would now be better, because it thinks one side of the join is small
when it's actually huge.

You'd confirm this with `EXPLAIN ANALYZE` by comparing the *estimated* row
count against the *actual* row count for the relevant node — a large
divergence is the signature of stale/insufficient statistics. The fix is
usually to run `ANALYZE` manually, check that autovacuum/auto-analyze is
actually keeping up (it can fall behind under heavy write load), or increase
`default_statistics_target` for columns with skewed/irregular distributions.

**Follow-up question:**
Autovacuum is enabled and running, but statistics are still stale on one
specific huge table. What are the likely reasons?

**Follow-up good answer:**
A few common causes on large tables specifically: (1) the default
autovacuum/auto-analyze thresholds (`autovacuum_analyze_scale_factor`, a
*percentage* of table size, plus a small fixed base amount) mean a huge
table has to accumulate a proportionally huge number of changed rows before
auto-analyze triggers again — so it runs, but rarely relative to how fast the
table's data is actually shifting; (2) a long-running or idle-in-transaction
session elsewhere is holding back cleanup/visibility progress and can delay
or starve autovacuum workers generally; (3) autovacuum is running but taking
so long on this one huge table (or being repeatedly canceled by conflicting
DDL) that it never completes an ANALYZE pass; or (4) the table is a heavy
target for concurrent autovacuum workers competing for
`autovacuum_max_workers`, so this specific table's turn comes up rarely. The
fix is usually to set per-table overrides
(`ALTER TABLE ... SET (autovacuum_analyze_scale_factor = ...)`) with a
smaller scale factor for that one large table, or run `ANALYZE` manually on
a schedule as a stopgap.

**Glossary:**
- **ANALYZE** — the command that gathers and stores planner statistics about a table's contents.
- **Cardinality estimation** — the planner's predicted row count for a plan node, derived from statistics.
- **Autovacuum/auto-analyze** — the background process that automatically runs VACUUM and ANALYZE as tables change.

**Mental model:**
Checks whether the candidate understands that query plans are *estimates
based on stale data*, not live computation — a subtlety that explains a
whole category of "it just got slow for no reason" incidents.

**References:**
- [PostgreSQL: 14.2. Statistics Used by the Planner](https://www.postgresql.org/docs/current/planner-stats.html)
- [PostgreSQL: 19.10. Automatic Vacuuming](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html)

---

### Q8. What is MVCC, and why does it mean readers don't block writers?

**Question:**
In PostgreSQL, a long-running `SELECT` doesn't block an `UPDATE` on the same
rows, and vice versa. Why?

**Good answer:**
PostgreSQL (like most modern relational databases) uses **Multiversion
Concurrency Control (MVCC)**. Instead of updating a row in place and making
concurrent readers wait, an `UPDATE` creates a new version of the row while
the old version stays intact and visible to transactions that started before
the update committed. Each transaction/statement effectively reads a
consistent "snapshot" of the database as of a point in time. Because reads
never need to acquire a lock that would conflict with a writer's lock (they
just look at an appropriately-visible row version), reads never block writes
and writes never block reads.

This is *not* free, though — it's the reason old row versions ("dead
tuples") accumulate and need to be cleaned up by `VACUUM`, and why very
long-running transactions are dangerous: they hold back the point up to
which old row versions can be safely reclaimed, causing table/index bloat.

**Follow-up question:**
If reads and writes don't block each other, can two concurrent `UPDATE`
statements on the *same row* still block each other? Why?

**Follow-up good answer:**
Yes. MVCC solves the reader-vs-writer conflict by giving readers a
consistent snapshot instead of blocking them, but it doesn't (and can't)
solve the writer-vs-writer conflict the same way, because two concurrent
updates to the same row can't both "win" — allowing both to create new row
versions independently would silently lose one of the updates. So PostgreSQL
still uses row-level locking for writers: the second `UPDATE` targeting a
row already being updated by an uncommitted transaction blocks until the
first transaction commits or rolls back, at which point it either proceeds
against the new row version (Read Committed) or, depending on isolation
level, can raise a serialization/update-conflict error instead.

**Glossary:**
- **MVCC** — Multiversion Concurrency Control; maintaining multiple versions of a row so readers and writers don't block each other.
- **Snapshot** — the consistent view of the database a transaction sees, as of a specific point in time.
- **Dead tuple** — an old row version no longer visible to any active transaction, eligible for reclamation by VACUUM.

**Mental model:**
This is the "what happens internally" concurrency-primitives question for
the SQL domain — the equivalent of asking about locks/mutexes in a systems
context, testing whether the candidate understands *why* the database
behaves the way it does under concurrent load, not just that it does.

**References:**
- [PostgreSQL: 13.1. Introduction](https://www.postgresql.org/docs/current/mvcc-intro.html)

---

### Q9. Walk through PostgreSQL's row-level lock modes and how `SELECT ... FOR UPDATE` affects concurrency.

**Question:**
What's the difference between `SELECT ... FOR UPDATE` and
`SELECT ... FOR SHARE`? How can using `FOR UPDATE` too broadly hurt
throughput?

**Good answer:**
Both acquire row-level locks to coordinate concurrent transactions, but with
different strictness:
- `FOR UPDATE` acquires an exclusive row lock — it blocks any other
  transaction from updating, deleting, or acquiring any other row lock
  (including another `FOR UPDATE`) on those rows until the transaction ends.
  It's what you use when you intend to modify the row and want to prevent
  anyone else from doing so concurrently (classic "read then update"
  pattern, avoiding lost updates).
- `FOR SHARE` acquires a shared lock — multiple transactions can hold it on
  the same rows simultaneously, but it still blocks `UPDATE`/`DELETE` from
  proceeding until the shared-lock holders finish.

The concurrency risk: if you wrap `SELECT ... FOR UPDATE` around more rows
than necessary, or hold the transaction open longer than necessary (e.g.
doing slow application logic before committing), every other transaction
touching those rows queues up behind the lock, which serializes work that
could otherwise run concurrently and can cascade into connection pool
exhaustion under load. This is also a classic source of deadlocks when two
transactions lock the same rows in different order.

**Code example:**
```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- ... application does the balance check/update logic here ...
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

**Follow-up question:**
Two transactions each run `UPDATE` on two rows in opposite order and
deadlock. How do you prevent this at the application level?

**Follow-up good answer:**
The standard defense is to enforce a **consistent lock acquisition order**
across every code path that touches multiple rows: always lock/update rows
in the same deterministic order (e.g. sorted by primary key), regardless of
the order the application logic happens to receive them in. If both
transactions in the example had updated the rows in the same order, one
would simply wait for the other to finish instead of deadlocking. Where
enforcing a global order isn't feasible, the fallback is to catch the
deadlock error and retry the transaction — PostgreSQL automatically detects
deadlocks and aborts one of the participating transactions, so the
application just needs to be written to retry on that specific error rather
than treating it as a hard failure. Keeping transactions short (not holding
locks open across app-level waits like user input) also shrinks the window
in which a deadlock can occur.

**Glossary:**
- **Row-level lock** — a lock held on individual rows rather than the whole table.
- **Deadlock** — a cycle of transactions each waiting on a lock held by another, none of which can proceed.
- **Lost update** — a race condition where two transactions read the same value, and one transaction's update overwrites the other's without seeing it.

**Mental model:**
Probes real transactional/locking experience beyond "I write SQL that
works on one connection" — whether the candidate has actually reasoned about
what happens under concurrent access.

**References:**
- [PostgreSQL: 13.3. Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)

---

### Q10. Why is an index lookup O(log n) and a full scan O(n) — and when does that stop mattering?

**Question:**
In big-O terms, why is indexed lookup so much faster than a table scan, and
is that always true in practice?

**Good answer:**
A B-tree index lookup is O(log n) in the number of rows, because each
comparison at a tree node eliminates a large fraction of the remaining
search space — the tree's branching factor keeps depth shallow even for
huge n. A full table (sequential) scan is O(n) because it has to examine
every row regardless of how selective the condition is.

In practice this stops being a clean win when **selectivity is low** — if a
condition matches a large fraction of the table (say, more than roughly
5-20%, depending on data layout), the extra random I/O of jumping between
index entries and then back to the heap for each matching row can be *more*
expensive overall than one sequential pass, because sequential I/O is much
cheaper per byte than random I/O on most storage. This is exactly why query
planners use cost-based optimization instead of "always prefer the index" —
and why you can observe a planner correctly *choosing* a sequential scan
over an available index for a low-selectivity condition.

**Follow-up question:**
Why does a `LIMIT` clause sometimes make the planner switch from a
sequential scan to an index scan for the same `WHERE` condition?

**Follow-up good answer:**
Without `LIMIT`, the planner has to produce *every* matching row, so its
cost estimate is based on scanning however much of the table/index is needed
to find them all — for a low-selectivity condition, a sequential scan can
win because it avoids the random I/O of repeated index-to-heap hops. With a
small `LIMIT`, the planner only needs to find enough matching rows to
satisfy the limit, and it can assume (based on statistics) that it will find
them relatively quickly by walking the index — so it estimates a much
cheaper *partial* index scan that stops early, rather than costing a full
scan of everything. This is also exactly why `LIMIT` without an accompanying
`ORDER BY` can return different rows across runs/plans — there's no
guarantee about which subset the planner happens to stop at.

**Glossary:**
- **Big-O notation** — a way to describe how an algorithm's cost scales with input size, ignoring constant factors.
- **Selectivity** — the fraction of rows a condition matches; low selectivity means few rows match.
- **Random vs sequential I/O** — random I/O jumps between disk locations (expensive per operation); sequential I/O reads contiguous data (cheap per byte).

**Mental model:**
This is the "software engineering theory mixed with technology-specific
practice" question — it wants the candidate to connect textbook algorithmic
complexity to a real cost-based optimizer's actual behavior, not just recite
"indexes are faster."

**References:**
- [PostgreSQL: 14.1. Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)

---

### Q11. What real-world problem do indexes solve, and what did databases do before them?

**Question:**
Why do indexes exist as a separate feature instead of the database just
being fast at scanning?

**Good answer:**
Before (or without) indexes, every query that filters, joins, or sorts has
to touch every row of a table to guarantee correctness — this is fine for
small tables but becomes untenable as data grows, because the cost scales
linearly with table size while user expectations for query latency stay
roughly constant. Real production systems need queries that return in
milliseconds regardless of whether the table has a thousand rows or a
billion.

Indexes solve this by trading write cost and storage for read speed: paying
a small, ongoing cost on every write (to keep the index up to date) in
exchange for near-constant-time lookups on read, which is almost always the
right trade in read-heavy systems (most OLTP applications read far more than
they write). The broader principle — precomputing/maintaining an auxiliary
structure to avoid repeated expensive work — shows up throughout systems
design (caches, materialized views, denormalization), and indexes are the
canonical database-level instance of it.

**Follow-up question:**
Give an example of a system where you'd deliberately *avoid* adding an
index even though it would speed up a specific query.

**Follow-up good answer:**
A high-throughput write-heavy table used mainly for ingestion — e.g. an
event/telemetry logging table receiving thousands of inserts per second,
where the only reads are rare, ad-hoc analytical queries run by a handful of
people. Adding an index to speed up those rare reads would add write
overhead to *every single insert*, on the table's hot path, to benefit a
query pattern that's infrequent and where users can tolerate a slower report
running for a few extra seconds. In that situation, the right trade is to
leave the operational table unindexed (or minimally indexed) and instead run
analytical queries against a replica, a data warehouse, or a periodically
refreshed materialized view — keeping the write path as cheap as possible
where it matters most.

**Glossary:**
- **OLTP (Online Transaction Processing)** — workloads characterized by many small, concurrent read/write transactions, as opposed to analytical (OLAP) workloads.
- **Denormalization** — deliberately duplicating data to avoid expensive joins/lookups at read time, at the cost of write complexity.

**Mental model:**
Tests whether the candidate can articulate the *why* behind a feature they
use daily — connecting it to a general systems-design principle (trading
write cost for read speed) rather than treating it as an arbitrary database
feature.

**References:**
- [PostgreSQL: Chapter 11. Indexes](https://www.postgresql.org/docs/current/indexes.html)

---

### Q12. What's the risk of over-indexing a table?

**Question:**
A teammate suggests "just add an index on every column we filter on,
there's no downside." Do you agree?

**Good answer:**
No — every index adds real, ongoing costs:
1. **Write amplification**: every `INSERT`/`UPDATE`/`DELETE` that touches an
   indexed column has to also update every affected index, so write latency
   and total I/O grow with the number of indexes on a table.
2. **Storage**: each index is its own on-disk structure and can be a
   significant fraction of (or larger than) the table itself.
3. **Planner overhead and confusion**: more indexes means more choices for
   the query planner to evaluate, and in some cases can lead it to pick a
   worse plan due to overlapping/competing options.
4. **Maintenance operations get slower**: `VACUUM`, `REINDEX`, and bulk
   loads all have to do more work per index.

The right approach is to index deliberately based on actual query patterns
(informed by `pg_stat_statements`/slow query logs), prefer well-chosen
composite indexes over many redundant single-column ones, and periodically
audit for unused indexes (e.g. via `pg_stat_user_indexes` — indexes with
near-zero scans are pure overhead).

**Follow-up question:**
How would you identify and safely remove unused indexes in a production
database?

**Follow-up good answer:**
Query `pg_stat_user_indexes` and look at `idx_scan` (the number of times the
index has actually been used to answer a query) — an index with `idx_scan`
near zero over a long observation window (long enough to cover periodic
jobs like month-end reports) is a strong unused-index candidate, aside from
unique/PK-backing indexes and ones enforcing a constraint, which must stay
regardless of scan count. Before dropping, I'd double-check it isn't a
recently-added index still waiting to prove itself, and confirm across all
replicas/environments since usage can differ by node. To remove it safely
under load, use `DROP INDEX CONCURRENTLY`, which avoids taking the exclusive
lock a plain `DROP INDEX` would hold, so reads/writes on the table aren't
blocked during the drop.

**Glossary:**
- **Write amplification** — extra write work caused by having to update auxiliary structures (indexes) in addition to the primary data.
- **REINDEX** — rebuilding an index from scratch, sometimes needed to reclaim bloat.

**Mental model:**
Tests judgment and restraint — a senior engineer should default to "indexes
are not free" rather than reflexively adding them, and should know how to
justify an index with data rather than intuition.

**References:**
- [PostgreSQL: Chapter 11. Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL: DROP INDEX](https://www.postgresql.org/docs/current/sql-dropindex.html)

---

### Q13. Why does wrapping an indexed column in a function or implicit cast stop the index from being used?

**Question:**
You have an index on `orders(created_at)`, but
`WHERE DATE(created_at) = '2026-08-01'` does a sequential scan. Why, and how
would you fix it?

**Good answer:**
A standard B-tree index stores the *raw* column values in sorted order.
When you wrap the column in a function (`DATE(created_at)`) or force an
implicit type conversion (e.g. comparing a `text` column to a number), the
database can no longer directly map your condition to a contiguous range in
the index — it would have to compute the function's result for every row to
check a match, which defeats the purpose of the index, so the planner falls
back to a sequential scan.

The fix is one of:
1. Rewrite the condition to operate on the raw column instead
   (`created_at >= '2026-08-01' AND created_at < '2026-08-02'` instead of
   `DATE(created_at) = ...`).
2. Create a **functional/expression index** that indexes the *result* of the
   function directly, so the planner can use it as long as the query uses
   the exact same expression.

**Code example:**
```sql
-- Sequential scan: DATE() wraps the column
SELECT * FROM orders WHERE DATE(created_at) = '2026-08-01';

-- Option A: rewrite as a sargable range condition
SELECT * FROM orders
WHERE created_at >= '2026-08-01' AND created_at < '2026-08-02';

-- Option B: expression index matching the exact function call
CREATE INDEX idx_orders_created_date ON orders (DATE(created_at));
```

**Follow-up question:**
What does "sargable" mean, and why is it a useful concept when writing
`WHERE` clauses?

**Follow-up good answer:**
"Sargable" (from "Search ARGument ABLE") describes a condition written in a
form the database can evaluate directly against an index — typically
`column <op> constant-or-bound-value`, with the column left bare and
unwrapped. It's useful as a mental checklist while writing queries: before
relying on an index, ask "is my condition sargable?" — i.e. is the indexed
column on one side by itself, with no function call, arithmetic, or type
coercion applied to it? If not (e.g. `DATE(created_at) = ...`,
`price * 1.1 > 100`, or comparing a `text` column to an `int`), the index
can't be used as a direct range/lookup, and you can usually restructure the
query — move the transformation to the constant side of the comparison
instead of the column side — to make it sargable again, which is often the
fastest fix for a "why is this suddenly slow" query without touching schema
at all.

**Glossary:**
- **Sargable (Search ARGument ABLE)** — a condition that can be evaluated using an index directly, without transforming the indexed column.
- **Expression/functional index** — an index built on the result of an expression rather than a raw column.

**Mental model:**
This is one of the most common real-world "why isn't my index being used"
footguns — tests whether the candidate can debug a subtle mismatch between
query shape and index shape, a very practical/pitfall-oriented skill.

**References:**
- [PostgreSQL: 11.3. Multicolumn Indexes](https://www.postgresql.org/docs/current/indexes-multicolumn.html)

---

### Q14. What's a partial index, and when would you use one?

**Question:**
What's a partial index? Give a real scenario where it would be a better
choice than a full index on the same column.

**Good answer:**
A partial index only includes rows that satisfy a specified predicate,
instead of indexing every row in the table. This is useful when queries
consistently only care about a subset of the data — indexing the rest is
pure overhead.

A classic example: an `orders` table where 95% of rows are `status =
'completed'` and rarely queried directly, but the application constantly
queries `WHERE status = 'pending'`. A full index on `status` wastes space
indexing the completed rows and slows every write. A partial index only on
the pending rows is much smaller, cheaper to maintain, and just as fast (or
faster, since it's smaller and more likely to stay cached) for the query
that actually matters.

Partial indexes are also commonly used to enforce a **unique constraint on a
subset** of rows — e.g. "only one row per user can have `is_primary =
true`" — which a normal unique index can't express.

**Code example:**
```sql
CREATE INDEX orders_pending_idx ON orders (created_at)
WHERE status = 'pending';

CREATE UNIQUE INDEX one_primary_address_per_user
ON addresses (user_id)
WHERE is_primary = true;
```

**Follow-up question:**
For a partial index to be used, what has to be true about the query's
`WHERE` clause relative to the index's predicate?

**Follow-up good answer:**
The planner has to be able to prove, from the query's `WHERE` clause alone,
that every row the query could match is guaranteed to satisfy the index's
predicate — i.e. the query's condition must logically imply the index's
condition. The simplest and most reliable way to guarantee this is for the
query to include the exact same condition as the index's predicate (e.g.
querying `WHERE status = 'pending'` against an index built
`WHERE status = 'pending'`); PostgreSQL can also handle some simple logical
implications (e.g. a query condition that's a stricter subset of the
predicate), but it doesn't do arbitrary theorem-proving, so in practice you
should keep the predicate simple and mirror it closely in the queries you
expect to benefit from the index.

**Glossary:**
- **Partial index** — an index built over only the rows matching a given predicate.
- **Predicate** — the boolean condition defining which rows are included in a partial index.

**Mental model:**
Advanced-feature question — separates candidates who only know "create an
index on the column" from those who've had to optimize a real skewed
dataset.

**References:**
- [PostgreSQL: 11.8. Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)

---

### Q15. B-tree vs. Hash index — what's the actual trade-off?

**Question:**
When would you choose a hash index over a B-tree index, if ever?

**Good answer:**
A B-tree supports equality *and* range/ordering operators (`<`, `<=`, `=`,
`>=`, `>`, `BETWEEN`, sorted `ORDER BY`, anchored `LIKE 'prefix%'`), and can
answer queries needing sorted output directly from the index. A hash index
supports **only equality (`=`)** comparisons — it can't help with ranges or
sorting at all, since hashing intentionally destroys ordering information.

In practice, B-tree is the sane default for almost everything because it's
strictly more capable for a modest overhead, and most workloads eventually
need range queries or sorting even if the initial use case was pure
equality. A hash index is a narrow optimization: it can be marginally faster
for very high-volume, equality-only lookups on large keys (e.g. long
strings) where a B-tree's extra comparisons/rebalancing overhead is
measurable — but this is a fairly narrow win, and hash indexes historically
had additional limitations (e.g. not being WAL-logged / crash-safe before
PostgreSQL 10), so most engineers default to B-tree unless they have
measured a specific reason not to.

**Follow-up question:**
Why can't a hash index be used to satisfy an `ORDER BY` on the indexed
column?

**Follow-up good answer:**
A hash index stores entries keyed by the *hash* of the column value, and a
good hash function is deliberately designed to scatter inputs across the
output space with no relationship between input order and output order —
two adjacent keys (like 5 and 6) can hash to completely unrelated, far-apart
buckets. So walking the hash index's buckets in storage order tells you
nothing about the original values' order; there's no way to produce sorted
output from it without effectively re-sorting the results after the fact,
which defeats the purpose of using an index to avoid a sort. A B-tree, by
contrast, physically stores entries in key order, so an index scan
inherently produces sorted output for free.

**Glossary:**
- **Hash index** — an index that maps a hash of the key to a bucket, supporting only equality lookups.
- **WAL (Write-Ahead Log)** — the log PostgreSQL uses for crash recovery and replication; indexes must be WAL-logged to survive a crash.

**Mental model:**
Classic trade-off/comparison question — tests whether the candidate can
articulate *why* a default exists instead of just knowing what the default
is.

**References:**
- [PostgreSQL: 11.2. Index Types](https://www.postgresql.org/docs/current/indexes-types.html)

---

### Q16. What's the difference between a clustered and a non-clustered (secondary) index?

**Question:**
In MySQL/InnoDB, what does it mean that the primary key is the "clustered
index," and why does that make secondary index lookups a two-step process?

**Good answer:**
In InnoDB, the table's rows are physically stored *in* the structure of the
clustered index — by default, ordered by primary key. There's no separate
"heap" file the way there is in PostgreSQL; the clustered index *is* the
table. This makes primary-key lookups and range scans on the primary key
very fast, since the data is right there in leaf nodes.

Every other (secondary) index stores the indexed column's value plus the
**primary key value** of the matching row, not the row's physical location.
So a secondary index lookup is a two-step process: (1) search the secondary
index to find the matching primary key value(s), then (2) look up those
primary key values in the clustered index to fetch the full row — this
second step is sometimes called a "bookmark lookup." This is exactly why a
short, stable primary key matters in InnoDB: every secondary index
duplicates it internally, so a wide or frequently-changing primary key
bloats every secondary index and makes every secondary lookup do more work.

**Follow-up question:**
Why is using a large random UUID as an InnoDB primary key often a
performance anti-pattern, beyond just index bloat?

**Follow-up good answer:**
Because the clustered index *is* the table's physical storage in InnoDB,
every insert has to place the new row in primary-key order. With a
sequential key (e.g. auto-increment), inserts always land at the
high/right-hand edge of the tree, so pages fill up efficiently — InnoDB's
own documentation notes sequentially-inserted index pages end up roughly
15/16 full, versus as low as 1/2 full for randomly-ordered inserts, since a
random UUID scatters new rows across arbitrary existing pages, constantly
forcing costly mid-tree page splits and leaving pages half-empty. Beyond the
wasted space this implies, that randomness also destroys **locality of
reference**: sequential inserts keep recently-written data clustered on a
small, hot set of pages that stay cached in the buffer pool, while random
UUID inserts spread writes across the entire keyspace, meaning far more of
each insert/lookup misses the cache and requires physical disk I/O — a much
larger effect on sustained throughput than the extra secondary-index
bytes alone.

**Glossary:**
- **Clustered index** — an index whose leaf nodes contain the actual row data, determining physical row storage order.
- **Secondary index** — any other index; in InnoDB, it stores primary key values instead of physical row pointers.
- **Bookmark lookup** — the second-step lookup from a secondary index entry back into the clustered index to retrieve the full row.

**Mental model:**
This is a "different engine, different internals" question — checks that
the candidate's mental model of indexing isn't hard-coded to one database's
implementation, and understands *why* primary key design choices ripple
into every other index.

**References:**
- [MySQL 8.4 Reference Manual: 17.6.2.1 Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html)
- [MySQL 8.4 Reference Manual: 17.6.2.2 The Physical Structure of an InnoDB Index](https://dev.mysql.com/doc/refman/8.4/en/innodb-physical-structure.html)

---

### Q17. What is parameter sniffing, and how does it cause a query to suddenly get slow?

**Question:**
A stored procedure runs fast for most callers but suddenly becomes very slow
for everyone, even though nothing was deployed. What could cause that?

**Good answer:**
This is a classic symptom of **parameter sniffing**. When a database
compiles/caches a query plan for a parameterized query or stored procedure,
it typically builds that plan based on the *specific parameter values* used
the first time it was compiled (or after the plan was evicted and
recompiled) — the optimizer "sniffs" those values to make cardinality
estimates. If that first execution happened to use an atypical, highly
selective value, the cached plan might be great for that value but terrible
for the far more common values used afterward (e.g. a plan optimized for a
rare status value gets reused for a status value matching 80% of the table).
Since the plan is cached and reused, *every* subsequent call pays the cost
of a plan optimized for the wrong data shape, until the plan is evicted or
statistics change enough to trigger a recompile.

The practical fix depends on the engine: forcing recompilation
(`OPTION (RECOMPILE)` in SQL Server), hinting a specific plan/parameter
value, restructuring the query to avoid one-size-fits-all parameterization
for skewed columns, or (on newer SQL Server versions) enabling **Parameter
Sensitive Plan (PSP) optimization**, which allows multiple cached plan
variants per statement based on parameter value buckets.

**Follow-up question:**
Why doesn't this problem happen the same way with ad-hoc (non-parameterized)
queries?

**Follow-up good answer:**
Parameter sniffing is specifically a consequence of *reusing* a cached plan
across different literal values. An ad-hoc query sent with literal values
baked directly into the SQL text (not bind parameters) is typically
optimized fresh for those exact literal values every time it's compiled —
the optimizer's cardinality estimates are based on the actual values in the
statement, not a placeholder, so each distinct query text gets its own
plan tailored to its own values. The trade-off is the opposite one: constant
recompilation costs CPU/planning overhead on every execution, which is why
parameterization and plan caching exist in the first place — parameter
sniffing is the price paid for that reuse, and ad-hoc queries simply aren't
paying for reuse.

**Glossary:**
- **Parameter sniffing** — the optimizer using the actual parameter values from a specific compilation to build a cached execution plan, which may not suit other parameter values.
- **Plan cache** — the store of compiled execution plans an engine reuses to avoid recompiling identical queries.

**Mental model:**
Tests whether the candidate has encountered a real "it got slow for no
apparent reason" production incident and understands cached-plan reuse
across different data shapes, not just single-query optimization.

**References:**
- [Microsoft Learn: Parameter Sensitive Plan Optimization - SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/performance/parameter-sensitive-plan-optimization)

---

### Q18. What is the N+1 query problem, and how do you detect and fix it?

**Question:**
An API endpoint that lists 50 orders is making 51 database queries. What's
happening, and how do you fix it?

**Good answer:**
This is the classic **N+1 query problem**, common with ORMs (Hibernate/JPA,
ActiveRecord, etc.) using lazy loading: one query fetches the N parent
records (the 50 orders), and then, because a related entity (e.g. each
order's customer) is lazily loaded, the ORM issues one *additional* query
per parent record the first time that relation is accessed — 50 more
queries, for 51 total, instead of one query (or one join) that could have
fetched everything.

You detect it by looking at query counts/logs for a single request — a
sudden proportional relationship between "number of rows returned" and
"number of queries issued" is the signature — or via `pg_stat_statements`
showing a query executed with a suspiciously high `calls` count for the
traffic volume.

The fix is to eagerly fetch the related data up front instead of lazily:
in JPA/Hibernate, use `JOIN FETCH` in the query, an `@EntityGraph`, or
batch fetching, so the related entities come back in one query (or a small,
fixed number of batched queries) instead of one per row.

**Code example:**
```java
// N+1: each order.getCustomer() triggers a separate query
List<Order> orders = orderRepository.findAll();
orders.forEach(o -> o.getCustomer().getName());

// Fixed: eagerly fetch customer in a single query via JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();
```

**Follow-up question:**
Why can eager-loading *everything* by default also be a performance
problem?

**Follow-up good answer:**
Eager-loading trades query *count* for query/result *size*, and that trade
isn't free either. Joining in every related entity for every request pulls
back data the caller often doesn't need, which means more bytes over the
wire, more memory to materialize into objects, and — critically — if you
eagerly fetch multiple one-to-many relations at once via joins, you get a
**Cartesian product** effect: joining an order to both its 10 line items and
its 5 status-history entries in one query returns 50 duplicated rows instead
of 15, which the ORM then has to de-duplicate in memory. The right default
is closer to "load only what this specific use case needs" — lazy by
default with deliberate, targeted eager-fetching (e.g. `JOIN FETCH`) added
per query where you know the access pattern, rather than blanket eager
loading everywhere.

**Glossary:**
- **Lazy loading** — deferring the loading of related data until it's actually accessed.
- **Eager loading / JOIN FETCH** — loading related data up front, typically via a join, to avoid separate follow-up queries.
- **ORM (Object-Relational Mapper)** — a library mapping database rows to application objects (e.g. Hibernate, ActiveRecord).

**Mental model:**
This is a "does the candidate understand what their ORM is doing under the
hood" question — a very common real-world performance bug that has nothing
to do with missing indexes and everything to do with query *count*.

**References:**
- [Hibernate ORM User Guide — Fetching](https://docs.hibernate.org/orm/current/userguide/html_single/Hibernate_User_Guide.html#fetching)

---

### Q19. Why does PostgreSQL need `VACUUM`, and how does bloat degrade performance?

**Question:**
A table that should be small based on row count is taking up far more disk
space than expected, and queries against it have gotten slower over time.
What's likely happening, and how do you fix it?

**Good answer:**
This is a classic symptom of **table/index bloat**, caused by PostgreSQL's
MVCC design: an `UPDATE` or `DELETE` doesn't remove the old row version
immediately (other transactions might still need to see it) — it just marks
it as a "dead tuple." Those dead tuples accumulate space and, until
reclaimed, both the table and any indexes referencing that column keep
growing, forcing scans (sequential or index) to read through more physical
pages than the live data alone would require — more I/O per query, worse
cache locality, slower everything.

`VACUUM` reclaims dead tuple space for reuse by future writes (though a
standard `VACUUM` typically doesn't shrink the file on disk except in
special cases) and updates the visibility map (enabling index-only scans).
`autovacuum` normally handles this automatically, so persistent bloat
usually means autovacuum is falling behind — often due to a long-running
transaction holding back the oldest needed snapshot, autovacuum settings
being too conservative for a high-churn table, or a very large table
accumulating dead tuples faster than default autovacuum thresholds trigger a
cleanup. Diagnosis involves checking `pg_stat_user_tables` for
`n_dead_tup`/last autovacuum time, and checking for long-running/idle-in-
transaction sessions holding back cleanup. If bloat is already severe,
`VACUUM FULL` (or `pg_repack` to avoid the exclusive lock `VACUUM FULL`
requires) rewrites the table to actually reclaim disk space.

**Follow-up question:**
Why can a single long-running transaction cause bloat across the *entire*
database, not just the tables it touches?

**Follow-up good answer:**
VACUUM can only remove a dead tuple once it's certain no current or future
transaction snapshot could still need to see it — and that safety line
(often called the "xmin horizon") is determined by the oldest still-open
transaction *anywhere in the cluster*, not per-table. As long as that one
long-running transaction is open, PostgreSQL has to assume it might still
query any table in the database using its original snapshot, so VACUUM is
blocked from reclaiming newly-dead tuples on *every* table being actively
written to, not just the ones the long transaction happens to touch. This is
why a single forgotten `idle in transaction` session can cause bloat
cluster-wide — the fix is to actively monitor for and terminate long-running
or idle-in-transaction sessions (e.g. via `pg_stat_activity`, or the
`idle_in_transaction_session_timeout` setting), not just tune autovacuum
parameters.

**Glossary:**
- **Dead tuple** — an old, no-longer-visible row version left behind by MVCC after an UPDATE/DELETE.
- **Bloat** — wasted space in a table or index from unreclaimed dead tuples.
- **Autovacuum** — PostgreSQL's background process that runs VACUUM/ANALYZE automatically based on configured thresholds.

**Mental model:**
Advanced/internals question that connects a "why is this slow" symptom
directly back to the MVCC mechanism discussed earlier — testing whether the
candidate can trace a performance symptom to its root architectural cause.

**References:**
- [PostgreSQL: 24.1. Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [PostgreSQL: 19.11. Client Connection Defaults (idle_in_transaction_session_timeout)](https://www.postgresql.org/docs/current/runtime-config-client.html)

---

### Q20. How do transaction isolation levels trade correctness for concurrency/performance?

**Question:**
What's the difference between `READ COMMITTED` and `SERIALIZABLE`, and why
wouldn't you just always use `SERIALIZABLE` to be safe?

**Good answer:**
The SQL standard defines isolation levels by which anomalies they prevent
when transactions run concurrently:
- **Read Uncommitted** — allows dirty reads (seeing another transaction's
  uncommitted changes). PostgreSQL treats this identically to Read
  Committed.
- **Read Committed** (PostgreSQL's default) — prevents dirty reads, but each
  statement within a transaction sees a fresh snapshot, so the *same query*
  run twice in one transaction can see different data if another transaction
  committed in between (non-repeatable reads, phantom reads still possible).
- **Repeatable Read** — the whole transaction sees one fixed snapshot from
  its start, preventing non-repeatable reads (and, in PostgreSQL
  specifically, phantom reads too, which is stronger than the SQL standard
  requires).
- **Serializable** — guarantees the outcome of concurrent transactions is
  equivalent to *some* serial (one-at-a-time) execution order, preventing
  all anomalies including serialization anomalies that Repeatable Read still
  allows.

The trade-off: stronger isolation costs more. `SERIALIZABLE` in PostgreSQL
uses predicate locking and has to detect and abort transactions that would
violate serializability, meaning your application must be prepared to retry
transactions that fail with a serialization error — and higher isolation
levels generally mean more lock contention / more aborted transactions under
concurrent load. Most applications default to Read Committed because it's
"correct enough" for the vast majority of operations and has the best
concurrency; you reach for Repeatable Read or Serializable only for specific
operations where the stronger guarantee (e.g. preventing a
read-then-write race across multiple statements) actually matters, since
you're trading throughput for that safety.

**Follow-up question:**
Give a concrete example of a bug that Read Committed allows but Repeatable
Read would prevent.

**Follow-up good answer:**
Classic example: a transaction reads an account balance, checks it's
sufficient, then later in the *same* transaction runs a second query that
also reads related data based on that balance (e.g. re-reading it to
compute a transfer amount), and finally issues the write. Under Read
Committed, each statement gets a fresh snapshot, so if another transaction
commits an update to that balance in between the two reads, the second read
sees the *new* value — the transaction's logic silently operates on two
different, inconsistent views of the same row within what should be one
atomic unit of work, potentially allowing an overdraft the first check
thought it had prevented. Under Repeatable Read, the entire transaction sees
one fixed snapshot taken at its start, so both reads return the same value;
if the underlying row was changed by another committed transaction, a
subsequent *write* by this transaction would instead fail with a
serialization error, forcing an explicit retry rather than silently
proceeding on stale logic.

**Glossary:**
- **Dirty read** — reading another transaction's uncommitted changes.
- **Non-repeatable read** — re-reading a row within a transaction and seeing different data because another transaction committed a change in between.
- **Phantom read** — re-running a query within a transaction and getting a different set of rows because another transaction inserted/deleted matching rows.
- **Serialization anomaly** — a result from concurrent transactions that couldn't have happened under any serial (one-at-a-time) execution order.

**Mental model:**
Classic trade-off question connecting SE theory (isolation guarantees) to
practical performance consequences — tests whether the candidate can reason
about *why* you wouldn't just pick the strictest option "to be safe."

**References:**
- [PostgreSQL: 13.2. Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
