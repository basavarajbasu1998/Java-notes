# SQL & Database Performance (Expert Interview Notes)

> Primary engine: **MySQL 8.0 / InnoDB**. Differences for **PostgreSQL** and **Oracle** are called out with `[PG]` / `[ORA]`.
> Output blocks tagged **[VALIDATED]** were produced by actually running the SQL (SQLite 3.45 via Python `sqlite3`; window functions, recursive CTEs, row-value comparison work there). Blocks tagged **[ILLUSTRATIVE]** are MySQL/PG outputs written by hand to show the *shape* - exact costs/row counts on your server will differ. No MySQL/PG server was available while writing.
> Related notes: `Spring_Transactional.md` (isolation from Spring's side), `Coding_SQL.md` (short problem list), `Collections.md` (HashMap ideas reappear in hash indexes).

## Table of contents
1. 60-second mental model
2. Storage and B+ tree indexes (deep)
3. Index design rules (composite, covering, prefix, functional, invisible, stats)
4. The optimizer, join algorithms, EXPLAIN / EXPLAIN ANALYZE
5. Why an index is not used + SARGable rewrites
6. Query patterns: N+1, pagination, COUNT, IN/EXISTS/JOIN, subqueries
7. SQL semantics: NULL, GROUP BY/HAVING, joins, UNION, window functions, CTEs
8. Transactions: ACID, MVCC, isolation levels with two-session timelines
9. InnoDB locking: record/gap/next-key, FOR UPDATE, deadlocks
10. Durability: redo/undo/binlog, crash recovery, flush settings
11. Replication, partitioning vs sharding, pooling
12. Schema design, constraints, online DDL, backup/PITR
13. Observability: slow log, performance_schema, sys
14. Caching, OLTP vs OLAP, NoSQL decision guide
15. Production war stories
16. 40 classic SQL interview queries (validated)
17. Interview questions (50+) with follow-ups and wrong answers
18. One-page cheat sheet

---

# 1. 60-second mental model

**Analogy - a library.**
- The **table** is a warehouse of books (rows). A **full scan** = walking every shelf.
- An **index** is the card catalogue: sorted cards pointing to shelf locations. In InnoDB the *primary key index is the warehouse itself*, shelved in PK order. A **secondary index** is a catalogue whose cards say "book with PK = 42", so you look up the card, then walk to the PK shelf (second traversal).
- The **optimizer** is a librarian estimating: "50 books needed out of 10 million: use the catalogue. 4 million out of 10 million: just walk the shelves".
- **MVCC** = every reader gets a photocopy of the library as it was when they walked in; writers put new editions on the shelf and keep the old edition in a back room (undo log) until no reader needs it.
- **Locks** = "do not touch" tags on books (record) and on the *empty space between books* (gap), so nobody can slot a new book in while you are checking a range.
- **WAL (redo log)** = the librarian first writes "I am moving book 42" in a diary (sequential, cheap, fsynced), then moves the book lazily. After a power cut, replay the diary.

**Five sentences to remember**
1. Reads are fast when the DB can **seek** a sorted structure (B+ tree) instead of scanning; writes pay to keep every index sorted.
2. Height of a B+ tree is 3-4 for hundreds of millions of rows; the cost is page reads, not comparisons.
3. The optimizer is **cost-based** on **statistics**; stale stats or non-SARGable predicates produce bad plans.
4. InnoDB default isolation is **REPEATABLE READ** (snapshot for plain SELECT, locks for writes/locking reads); PG and Oracle default to READ COMMITTED.
5. Durability = redo log fsync at commit (`innodb_flush_log_at_trx_commit=1`) + binlog fsync (`sync_binlog=1`); replication is asynchronous unless configured otherwise, so replicas lag.

---

# 2. Storage and B+ tree indexes (deep)

## 2.1 Pages: the unit of I/O
InnoDB reads and writes **pages** of `innodb_page_size` = **16 KB** by default. Tablespace = extents (1 MB = 64 pages) = segments. The **buffer pool** caches pages in RAM (LRU with midpoint insertion so a big scan does not evict hot pages). A "logical read" that finds the page in the buffer pool costs microseconds; a physical read costs ~0.1 ms (NVMe) to ~5-10 ms (spinning disk / network storage).

Rule: **query cost ~ number of distinct pages touched**, not number of rows compared.

## 2.2 B+ tree structure
```
                       [ root page: keys 1000 | 2000 | 3000 ]            level 2 (root)
                        /          |          |          \
              [ 200 | 500 | 800 ] [1200|1500]  [2200|2600]  [3200|...]  level 1 (internal)
               /     |    |    \
     leaf pages (doubly linked list, sorted by key)                      level 0 (leaf)
  [1..199] <-> [200..499] <-> [500..799] <-> [800..999] <-> ...
   (row data in clustered index; PK values in secondary index)
```
- **Internal pages** hold (key, child page number) only. **Leaf pages** hold the payload and are chained left/right, so a **range scan** = descend once, then walk the leaf list. A B-tree (non-plus) would store data in internal nodes too, lowering fan-out and making range scans awkward.
- Insert into a full leaf -> **page split** (half the entries move to a new page, parent gets a new key). Random-key inserts split all over the tree; monotonically increasing keys only split the right edge.
- Pages are 15/16 full after sequential loads (`innodb_fill_factor` default 100 keeps 1/16 free), roughly **50-70% full** after random inserts + splits. Deletes leave holes; under 50% full pages **merge** (`MERGE_THRESHOLD` = 50%).

## 2.3 Fan-out and height: worked arithmetic
Assumptions (typical): page 16384 B, `BIGINT` PK = 8 B, child page pointer = 4 B (InnoDB page no.) plus ~ 5-6 B record header/overhead => ~ **18 B per internal entry** => fan-out ~ 16384 / 18 ~ **900**. Use **~1000** as the round interview number (honest range 400-1200 depending on key width).

**Case A: table of 1,000,000 rows, ~1 KB rows (clustered leaf holds full row)**
```
rows per leaf page          = 16 KB / 1 KB            ~ 16
leaf pages                  = 1,000,000 / 16          ~ 62,500       (~1 GB of data)
level-1 (internal) pages    = 62,500 / 1000           ~ 63
root                        = 63 / 1000  -> 1 page
height = 3 levels  (root -> internal -> leaf)  => 3 page reads worst case, upper 2 levels (64 pages ~ 1 MB) always cached
=> ~1 physical read per PK lookup
```
**Case B: 100,000,000 rows, ~1 KB rows**
```
leaf pages                  = 100M / 16               ~ 6,250,000    (~100 GB)
level-2 pages               = 6.25M / 1000            ~ 6,250        (~100 MB, cacheable)
level-1 pages               = 6,250 / 1000            ~ 7
root                        = 1
height = 4 levels => still only 4 page reads; ~1 physical read if internal pages are cached
```
Each extra level multiplies capacity by ~1000, so **log_1000(N)**: 10^9 rows still height 4-5.

**Case C: secondary index on `email VARCHAR(100)` utf8mb4, 100M rows** (index leaf entry = email ~ 30 B avg + PK 8 B + ~6 B overhead ~ 45 B):
```
entries per leaf ~ 16384/45 ~ 360 ; leaf pages ~ 100M/360 ~ 280,000 (~4.4 GB)
internal fan-out ~ 16384 / (30+4+6=40) ~ 400 ; level1 ~ 700 pages ; root 2 -> height 3-4
```
This is why wide secondary-index keys hurt: bigger keys -> lower fan-out -> more pages -> larger buffer pool footprint.

**Contrast:** a full scan of Case B reads 6.25M pages = 100 GB; a PK lookup reads 4 pages. That ratio (1.5 million x) is the whole point.

## 2.4 Clustered index and secondary indexes (InnoDB)
- **Clustered index = the table.** Leaf pages of the PK B+ tree contain the full rows. If there is no PK, InnoDB uses the first `UNIQUE NOT NULL` index, else a hidden 6-byte `DB_ROW_ID`.
- **Secondary index leaf** = (indexed columns, **PK columns**). Not a physical address. Because rows can move (page splits), the PK is the stable pointer.
- Consequence 1: **wide PK -> every secondary index gets fatter.** A 36-char UUID string PK (utf8mb4: up to 144 B) copies into every secondary entry.
- Consequence 2: a lookup via a secondary index that needs a non-indexed column does **two traversals** (the "bookmark lookup" / "back to table" / "random I/O").

```
SELECT name, city FROM users WHERE email = 'a@x.com';   -- idx_email(email), PK(id)

 idx_email B+ tree                         clustered (PK) B+ tree
 root -> internal -> leaf                  root -> internal -> leaf
 leaf entry: ('a@x.com', id=8123)  ------> seek id=8123 -> full row (name, city, ...)
        traversal #1 (3 pages)                    traversal #2 (3 pages, likely 1 physical miss)
```
For 500 matching rows via a secondary index that is not covering, that is up to 500 random PK traversals, which is why the optimizer abandons the index when it expects to fetch more than roughly **20-30% of the table** (rule of thumb; really cost based - sequential scans are much cheaper per page than random lookups).

**Covering index / index-only scan:** if the index contains every column the query needs, the second traversal is skipped. MySQL shows `Extra: Using index`. Since secondary entries already contain the PK, `SELECT id ...` is covered by any secondary index.
```sql
CREATE INDEX idx_cust_status_created ON orders (customer_id, status, created_at);
-- covered: needs customer_id, status, created_at, and id (PK is inside every secondary entry)
SELECT id, status, created_at FROM orders WHERE customer_id = 42;   -- Extra: Using index
```
`[PG]` heap tables + indexes point to a tuple id (ctid); no clustered index by default (`CLUSTER` reorders once, not maintained). Index-only scans also need the **visibility map** to be up to date (VACUUM), else they still visit the heap. PG 11+ supports `CREATE INDEX ... INCLUDE (col)` to add payload columns without making them part of the key; MySQL has no INCLUDE (append columns to the key instead).
`[ORA]` heap tables by default; **index-organized tables (IOT)** are the clustered analog. Oracle also has bitmap indexes (OLAP, bad for concurrent DML) and function-based indexes.

## 2.5 UUID vs auto-increment primary key
| Aspect | `BIGINT AUTO_INCREMENT` | Random UUIDv4 as `CHAR(36)` | UUID as `BINARY(16)` (v1 swapped / v7 time-ordered) |
|---|---|---|---|
| Insert pattern | append at right edge, pages ~15/16 full | random leaf each insert: splits, ~50-70% fill, cache misses | v7/ordered: append-like; v4: random |
| PK size copied into every secondary index | 8 B | 36 B (144 B utf8mb4 worst case) | 16 B |
| Buffer pool | only right edge hot | whole index must be hot | depends on ordering |
| Guessable / enumerable | yes (leaks volume, IDOR risk) | no | no |
| Multi-node generation | needs coordination (or Snowflake ids) | free | free |

Recommendation: internal PK = `BIGINT` (or time-ordered UUIDv7/ULID as `BINARY(16)`), expose an opaque public id in a separate unique column if needed. MySQL 8 has `UUID_TO_BIN(uuid, 1)` (swap-flag rearranges time bits so v1 UUIDs sort roughly by time). Auto-increment caveats: gaps after rollbacks, `innodb_autoinc_lock_mode=2` (interleaved, default in 8.0) can interleave values in bulk inserts, auto-inc counter persisted since 8.0 (before, reset on restart to MAX+1).

## 2.6 Hash indexes (conceptual)
- A hash index maps `hash(key) -> row`: O(1) equality, **no ranges, no ordering, no prefix, no leftmost use**. Worse under collisions and resize.
- MySQL: `MEMORY` engine supports HASH; **InnoDB has an Adaptive Hash Index (AHI)**: built automatically in memory over hot B+ tree pages so repeated equality lookups skip tree descent. It can become a mutex bottleneck (`innodb_adaptive_hash_index` can be turned off). You cannot create hash indexes on InnoDB tables by DDL. (`USING HASH` on InnoDB is accepted syntactically but creates a B-tree.)
- `[PG]` `CREATE INDEX ... USING hash` exists (WAL-logged since PG 10) but B-tree nearly always wins. Other PG index types: GIN (arrays, jsonb, full-text), GiST, BRIN (huge append-only, correlated with physical order), SP-GiST.
- Interview connection: same idea as `HashMap` - equality fast, no ordering; B+ tree is like `TreeMap` optimized for disk pages.

---

# 3. Index design rules

## 3.1 Composite index and the leftmost-prefix rule
An index on `(country, city, age)` is one B+ tree sorted by country, then city within country, then age within city. Think of a phone book sorted by (last name, first name): you cannot find all "Priya" without scanning.

```
Sorted leaf order of idx(country, city, age):
 (IN, Bengaluru, 22)
 (IN, Bengaluru, 30)
 (IN, Chennai,   25)
 (IN, Mumbai,    28)
 (US, Austin,    31)
 ...
WHERE country='IN' AND city='Bengaluru'  -> contiguous slice (seek)
WHERE city='Bengaluru'                   -> entries scattered across every country (no seek)
```
| Predicate | Uses index? | Notes |
|---|---|---|
| `country=?` | yes | `key_len` covers country only |
| `country=? AND city=?` | yes | 2 columns |
| `country=? AND city=? AND age=?` | yes | 3 columns |
| `country=? AND age=?` | partially | seek on country, `age` filtered while scanning slice (Index Condition Pushdown may evaluate it inside the index: `Using index condition`) |
| `city=?` | no seek | MySQL 8.0.13+ can do **skip scan** for range access when the leading column has few distinct values (`Using index for skip scan`); do not rely on it |
| `country=? AND city > ? AND age=?` | seek on country+city range; `age` cannot narrow | **columns after the first range column are not used for seeking** |
| `ORDER BY country, city` | yes | index order |
| `ORDER BY city` | no | filesort |

**Column order rules (in priority order)**
1. Columns used with **equality** first, the (single) **range** column next, then columns used only for ORDER BY / covering.
2. Among equality columns, put the more **selective** first only if queries do not share a common prefix pattern; more important is *which prefixes your queries use*. `(a,b)` also serves `a`-only queries; `(b,a)` does not.
3. Do not create `(a)` and `(a,b)` both (redundant: `(a)` is a prefix). Keep `(a,b)`.
4. For `WHERE status='PAID' AND created_at > ? ORDER BY created_at`: `(status, created_at)` gives seek + already sorted output.

Selectivity = distinct values / rows. `gender` (2 values) is ~0.00000002 selective alone, but as **leading equality column** in `(gender, created_at)` it can still be great if the query always supplies it and then ranges over `created_at`. Selectivity of the *combination* matters.

## 3.2 EXPLAIN examples for leftmost prefix [ILLUSTRATIVE, MySQL 8]
```sql
CREATE TABLE person (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  country CHAR(2), city VARCHAR(50), age TINYINT, name VARCHAR(100),
  KEY idx_ccg (country, city, age)
);
EXPLAIN SELECT * FROM person WHERE country='IN' AND city='Pune';
```
```
id | select_type | table  | type | possible_keys | key     | key_len | ref         | rows | filtered | Extra
1  | SIMPLE      | person | ref  | idx_ccg       | idx_ccg | 212     | const,const | 1200 | 100.00   | NULL
```
`key_len` = 2*4+1 (CHAR(2) utf8mb4 nullable) = 9 for country, +(50*4+2+1)=203 for city => **212**; the exact number is less important than the *trend*: bigger key_len means more columns of the index are used. Compare:
```sql
EXPLAIN SELECT * FROM person WHERE city='Pune';
-- type: ALL (or index / skip scan), key: NULL, rows: ~ table size, Extra: Using where

EXPLAIN SELECT country, city, age FROM person WHERE country='IN';
-- type: ref, key: idx_ccg, Extra: Using index          <- covering, no PK lookup

EXPLAIN SELECT * FROM person WHERE country='IN' AND city > 'M' AND age = 30;
-- type: range, key: idx_ccg, key_len covers country+city only,
-- Extra: Using index condition   (age checked on the index entry, still scans the range)
```
**Real, validated query plans (SQLite, table `t(a,b,c)` 5000 rows, index `ix(a,b)`) [VALIDATED]:**
```
where a=3 and b=13            -> SEARCH t USING INDEX ix (a=? AND b=?)
where b=13                    -> SCAN t                              (leftmost prefix violated)
select a,b where a=3          -> SEARCH t USING COVERING INDEX ix (a=?)
where a=3 order by b          -> SEARCH t USING INDEX ix (a=?)        (no sort step, index order used)
where a=3 order by c          -> SEARCH ... + USE TEMP B-TREE FOR ORDER BY   (== MySQL "Using filesort")
where abs(a)=3                -> SCAN t                              (function on column)
```

## 3.3 Range, then ORDER BY: filesort avoidance
```sql
-- idx(customer_id, created_at)
SELECT * FROM orders WHERE customer_id=42 ORDER BY created_at DESC LIMIT 10;   -- no filesort, reads 10 index entries backward
SELECT * FROM orders WHERE customer_id=42 AND status='PAID' ORDER BY created_at DESC LIMIT 10;
-- index (customer_id, created_at) still no filesort but reads entries then filters status: may scan many.
-- better: idx(customer_id, status, created_at)   (equality, equality, sort key)
SELECT * FROM orders WHERE customer_id IN (1,2,3) ORDER BY created_at LIMIT 10;
-- IN is multiple equality seeks => result is NOT globally ordered => filesort (or merge). Common trap.
SELECT * FROM orders WHERE customer_id=42 AND created_at > '2025-01-01' ORDER BY total;   -- filesort: range col != sort col
```
Rule: the ORDER BY can use the index only if the columns after the equality prefix match the sort order (direction may be reversed for all columns at once; MySQL 8 has true descending indexes for mixed directions: `KEY (a ASC, b DESC)`). `Using filesort` does not necessarily mean a file on disk; it means an explicit sort step (memory sort buffer `sort_buffer_size`, spills to temp files when large).

## 3.4 Prefix indexes
```sql
CREATE INDEX idx_email20 ON users (email(20));       -- index first 20 characters
```
Saves space for long strings (URLs, emails), but: cannot be **covering** (needs the full value), cannot be used for ORDER BY / GROUP BY fully, and selectivity drops if prefixes collide. Pick length by measuring:
```sql
SELECT COUNT(DISTINCT LEFT(email,10))/COUNT(*) s10, COUNT(DISTINCT LEFT(email,20))/COUNT(*) s20, COUNT(DISTINCT email)/COUNT(*) full FROM users;
```
InnoDB max index key prefix: 3072 B (DYNAMIC row format); 767 B with legacy COMPACT.

## 3.5 Functional indexes (MySQL 8.0.13+)
```sql
CREATE INDEX idx_lower_email ON users ((LOWER(email)));       -- NOTE the double parentheses
SELECT * FROM users WHERE LOWER(email) = 'a@x.com';           -- now sargable through the functional index
```
Implemented as a hidden virtual generated column with an index. Alternatives: explicit generated column (`... AS (LOWER(email)) VIRTUAL`, then index it) or use a case-insensitive collation (`utf8mb4_0900_ai_ci` is the 8.0 default, already case-insensitive, so `LOWER()` is often unnecessary in MySQL). `[PG]` expression indexes: `CREATE INDEX ON users (lower(email))`; also **partial indexes**: `CREATE INDEX ON orders (customer_id) WHERE status='OPEN'` (MySQL has no partial indexes). `[ORA]` function-based indexes.

## 3.6 Invisible indexes (safe drop rehearsal)
```sql
ALTER TABLE orders ALTER INDEX idx_old INVISIBLE;   -- optimizer ignores it, but it is still maintained on writes
-- watch for regressions for a week; if bad:
ALTER TABLE orders ALTER INDEX idx_old VISIBLE;     -- instant rollback (no rebuild)
-- if fine:
DROP INDEX idx_old ON orders;
```
`SET SESSION optimizer_switch='use_invisible_indexes=on'` lets a session test with them.

## 3.7 Cardinality, statistics, ANALYZE TABLE
The optimizer estimates row counts from **index statistics**: InnoDB samples `innodb_stats_persistent_sample_pages` (default 20) leaf pages to estimate `Cardinality` in `SHOW INDEX FROM t`. Stats are persisted (`mysql.innodb_index_stats`) and auto-recalculated when >10% of rows changed (`innodb_stats_auto_recalc`).
```sql
SHOW INDEX FROM orders;              -- Cardinality column = estimated distinct prefix values
ANALYZE TABLE orders;                -- re-sample stats (cheap, brief metadata lock)
ANALYZE TABLE orders UPDATE HISTOGRAM ON status, region WITH 64 BUCKETS;   -- column histograms for non-indexed skewed columns (8.0)
```
Symptoms of stale/wrong stats: plan flips after bulk load; `rows` in EXPLAIN far from reality (compare with EXPLAIN ANALYZE actual rows); fix = ANALYZE, histogram for skew, hint (`/*+ INDEX(o idx) */`, `FORCE INDEX`) as last resort. `[PG]` `ANALYZE` / autovacuum-analyze, `default_statistics_target`, `CREATE STATISTICS` for correlated columns. `[ORA]` `DBMS_STATS.GATHER_TABLE_STATS`, histograms, SQL plan baselines.

## 3.8 When indexes hurt
- Every INSERT/DELETE updates **every** index; UPDATE of an indexed column deletes + inserts entries in that index. 8 indexes ~ 9 B+ tree writes per row insert, plus undo/redo for each.
- Secondary index changes are buffered by the **change buffer** (for non-unique secondary indexes) so random I/O is deferred.
- More indexes -> more buffer pool pages, longer backups, slower `ALTER`, slower optimizer planning, more lock footprint (each index used by a locking read gets locked).
- Low-cardinality single-column indexes (boolean flags) are rarely used; a composite with a selective partner is better.
- Bulk load: load first (sorted by PK), create secondary indexes after.

## 3.9 Finding redundant / unused indexes
```sql
SELECT * FROM sys.schema_redundant_indexes\G      -- (a) vs (a,b): (a) is redundant. Also duplicates.
SELECT * FROM sys.schema_unused_indexes;           -- never used since server start (performance_schema); restart resets, so observe over a full business cycle
SELECT * FROM sys.schema_index_statistics ORDER BY rows_selected DESC;
```
Procedure: candidate -> make INVISIBLE -> observe -> DROP. Remember unique/PK-backing indexes and indexes that support **foreign keys** (InnoDB needs an index on FK columns; it creates one implicitly).

---

# 4. The optimizer, join algorithms, EXPLAIN

## 4.1 What the optimizer does
```
SQL text -> Parser -> Resolver (names, types) -> Rewrite (subquery->semijoin, view merge, constant folding)
        -> Cost-based optimizer: for each candidate access path & join order, estimate cost = I/O cost + CPU cost
           using row estimates (index stats, histograms) and cost constants (mysql.server_cost / engine_cost)
        -> Execution plan (iterator tree) -> Executor pulls rows
```
- **Cost-based**: picks the plan with the lowest *estimated* cost; wrong estimates -> wrong plan.
- **Join order**: for N tables N! orders; MySQL uses a greedy/pruned search (`optimizer_search_depth`). Heuristic: start from the table with the smallest filtered result, join to tables reachable through an index. Override with `STRAIGHT_JOIN` or `/*+ JOIN_ORDER(a,b) */` only with evidence.
- Access path per table: `const` / `eq_ref` / `ref` / `range` / `index` / `ALL`.

## 4.2 Join algorithms
| Algorithm | When | Cost intuition |
|---|---|---|
| **Nested loop (index NLJ)** | inner side has an index on join column (`eq_ref`/`ref`) | outer rows x log(N) seeks; best when outer is small |
| **Block nested loop (BNL)** | no index, MySQL < 8.0.18 | buffers outer rows, scans inner once per buffer |
| **Hash join** | equi-join without usable index. MySQL **8.0.18+** (and from 8.0.20 replaced BNL entirely, also used for inner non-equi, semi/anti/outer joins) | build hash table on smaller input (`join_buffer_size`, spills to disk in chunks), probe with the other: O(n+m). EXPLAIN Extra: `Using join buffer (hash join)` |
| **Sort-merge join** | `[PG]`, `[ORA]`; **MySQL has none** | sort both inputs on join key then merge; wins when inputs already sorted (index order) or for huge non-hashable joins |

```
Index nested loop:            Hash join:
for each row r in orders      1) build: hash table on customers.id (small side)
   (filter status='PAID')     2) probe: for each orders row, look up customer_id in hash table
   seek customers by PK       memory O(smaller side); no index needed
```
`[PG]` chooses among Nested Loop, Hash Join, Merge Join and can use parallel plans; `join_collapse_limit` and GEQO for many tables.

## 4.3 Reading EXPLAIN column by column
```sql
EXPLAIN SELECT ...;                 -- estimated plan, tabular
EXPLAIN FORMAT=TREE SELECT ...;     -- iterator tree (8.0.16+)
EXPLAIN ANALYZE SELECT ...;         -- EXECUTES the query (8.0.18+), adds actual time / rows / loops. Do not run on a destructive statement without a transaction you will roll back
```
| Column | Meaning |
|---|---|
| `id` | SELECT number; larger id runs first for subqueries; same id = joined together, top to bottom = join order |
| `select_type` | SIMPLE, PRIMARY, SUBQUERY, DERIVED, UNION, DEPENDENT SUBQUERY (correlated: bad sign) |
| `table` | table or alias (or `<derived2>`) |
| `partitions` | pruned partitions accessed |
| `type` | **access type**, best to worst below |
| `possible_keys` | candidates; `key` = what was chosen |
| `key_len` | bytes of the index actually used: tells how many composite columns |
| `ref` | what is compared to the key (`const`, another table's column, `func`) |
| `rows` | **estimated** rows examined for this step (per outer row!) |
| `filtered` | % of `rows` expected to survive the non-index conditions. rows x filtered% = estimated rows passed to next join step |
| `Extra` | the important hints, below |

**`type` ranking (best -> worst):** `system` > `const` (PK/unique lookup with constant) > `eq_ref` (unique/PK lookup per outer row in a join) > `ref` (non-unique index equality) > `fulltext` > `ref_or_null` > `index_merge` > `range` (`BETWEEN`, `<`, `>`, `IN`, `LIKE 'x%'`) > `index` (**full index scan**: reads whole index, cheaper than ALL only if covering) > `ALL` (**full table scan**).

**`Extra` values**
| Extra | Meaning | Good/bad |
|---|---|---|
| `Using index` | covering index, no table access | good |
| `Using index condition` | ICP: engine checks part of WHERE on index entries before fetching row | good |
| `Using where` | server filters rows after engine returns them (condition not fully served by index) | neutral; with `type=ALL` it means scan + filter |
| `Using filesort` | explicit sort step (not necessarily disk) | bad on big sets, avoid with matching index |
| `Using temporary` | temp table for GROUP BY / DISTINCT / UNION / derived | bad on big sets |
| `Using join buffer (hash join)` | join without usable index | check missing index |
| `Backward index scan` | reading index descending | fine |
| `Using index for skip scan` | 8.0.13+ skip scan | okay |
| `Select tables optimized away` | e.g. `MIN/MAX` on indexed col | great |
| `Impossible WHERE` | contradiction, no rows | check logic |

## 4.4 Annotated EXPLAIN examples [ILLUSTRATIVE, MySQL 8]
Schema: `customers(id PK, name, email UNIQUE, country)`, `orders(id PK, customer_id, status, total, created_at, KEY idx_cust_created(customer_id, created_at))`, 5M orders, 200k customers.

**Example 1 - PK lookup**
```sql
EXPLAIN SELECT * FROM orders WHERE id = 1001;
```
```
type=const  key=PRIMARY  key_len=8  ref=const  rows=1  filtered=100.00  Extra=NULL
```
Reading: one B+ tree descent; optimizer treats it as a constant at plan time.

**Example 2 - Full scan (missing index)**
```sql
EXPLAIN SELECT * FROM orders WHERE status = 'PAID' AND total > 500;
```
```
type=ALL  possible_keys=NULL  key=NULL  rows=4987000  filtered=3.33  Extra=Using where
```
Reading: no usable index; reads ~5M rows, expects 3.33% (~166k) to pass. Fix: `CREATE INDEX idx_status_total ON orders(status, total)` (equality then range) -> `type=range`, `Using index condition`.

**Example 3 - Join with eq_ref**
```sql
EXPLAIN SELECT c.name, o.total FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE o.created_at >= '2025-06-01' AND o.status='PAID';
```
```
id table type   key              ref                rows    filtered Extra
1  o     range  idx_created      NULL               120000  10.00    Using index condition; Using where
1  c     eq_ref PRIMARY          shop.o.customer_id 1       100.00   NULL
```
Reading: driving table `o` (top row) filters by range on `created_at` (index `idx_created` assumed), ~120k rows examined, 10% pass -> 12k rows x 1 PK lookup on `c` each. Total lookups ~ 12k, not 5M.

**Example 4 - filesort + temporary**
```sql
EXPLAIN SELECT status, COUNT(*) FROM orders GROUP BY status ORDER BY COUNT(*) DESC;
```
```
type=index  key=idx_status  rows=4987000  Extra=Using index; Using temporary; Using filesort
```
`Using index` (covering scan of idx_status, cheap) but grouping result of ~5 rows goes to a temp table and is sorted: harmless because the *result* is tiny; the cost is the 5M-entry index scan. The fix for dashboards is pre-aggregation, not an index.

**Example 5 - correlated subquery**
```sql
EXPLAIN SELECT * FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id=c.id AND o.total>1000);
```
MySQL 8 turns `EXISTS` into a **semijoin** (strategies: FirstMatch, LooseScan, Materialization, DuplicateWeedout): you will see `select_type=SIMPLE` for both rows and `Extra: FirstMatch(c)`. A `DEPENDENT SUBQUERY` in `select_type` means it was *not* transformed and runs once per outer row (5.x behaviour, or with `NOT IN`+NULL semantics / LIMIT / aggregates).

**Example 6 - EXPLAIN ANALYZE**
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id=42 ORDER BY created_at DESC LIMIT 10;
```
```
-> Limit: 10 row(s)  (cost=2.4 rows=10) (actual time=0.041..0.055 rows=10 loops=1)
    -> Index lookup on orders using idx_cust_created (customer_id=42) (reverse)
         (cost=2.4 rows=10) (actual time=0.040..0.052 rows=10 loops=1)
```
How to read: `cost` and `rows` are optimizer estimates; `actual time=first_row..last_row` (ms), `rows` = actual rows *per loop*, `loops` = how many times that node ran. **Total rows = rows x loops.** Warning signs: estimated rows 100 vs actual 2,000,000 (stale stats/skew), `loops` in the millions on an inner node (N+1 inside the query), a node whose time dominates.
`[PG]` `EXPLAIN (ANALYZE, BUFFERS)` shows `Seq Scan`, `Index Scan`, `Index Only Scan`, `Bitmap Heap Scan`, `Heap Fetches`, `Rows Removed by Filter`, `Buffers: shared hit/read` - buffers are the most reliable cost signal.

## 4.5 The 30-second EXPLAIN checklist
1. `type` = ALL / index on a big table? 2. `key` NULL though `possible_keys` not? 3. `rows` x nested loops explosion? 4. `filtered` tiny (index fetches many rows only to discard)? 5. `Using filesort` / `Using temporary` on big sets? 6. `key_len` shorter than expected (stopped at first range column)? 7. estimate vs EXPLAIN ANALYZE actual gap (stats)?

---

# 5. Why an index is not used

| Cause | Bad | Fix |
|---|---|---|
| Function/expression on column | `WHERE YEAR(created_at)=2025` | `created_at >= '2025-01-01' AND created_at < '2026-01-01'` |
| Arithmetic on column | `WHERE price*1.18 > 100` | `WHERE price > 100/1.18` |
| Implicit type conversion | `WHERE phone = 9876543210` (phone VARCHAR) -> converts every row to number | `WHERE phone = '9876543210'` |
| Leading wildcard | `LIKE '%son'` | `LIKE 'son%'`; for infix use FULLTEXT / trigram (`[PG]` pg_trgm) / reversed-string column for suffix |
| `OR` across different columns | `WHERE a=1 OR b=2` | `UNION ALL` of two indexed queries (add `AND a<>1` or use UNION) - or rely on `index_merge` if it appears |
| `NOT IN`, `<>`, `!=` | `WHERE status <> 'DONE'` | usually low-selectivity anyway; invert to `IN ('NEW','OPEN')` |
| `NOT IN (subquery)` with NULLs | semantics + poor plan | `NOT EXISTS` |
| Low selectivity | `WHERE is_active=1` (95% rows) | scan is correct; consider redesign |
| Collation/charset mismatch in join | `a.name (utf8mb4_0900_ai_ci) = b.name (latin1)` -> convert per row | make both columns same charset/collation |
| Leftmost prefix violated | `WHERE city=?` on `(country,city)` | reorder / add index |
| Stale stats / skew | optimizer picks scan | `ANALYZE TABLE`, histogram, hint |
| Index too wide / prefix only | selectivity loss | measure |
| Range on first column then equality | `WHERE a>1 AND b=2` on `(a,b)` | index `(b,a)` |

**Implicit conversion rules of thumb (MySQL):** comparing a `VARCHAR` column to a number converts the *column* to double for each row -> index unusable and wrong matches (`'1abc' = 1` is true). Comparing an `INT` column to a string constant converts the constant -> index still fine. Joins between `INT` and `VARCHAR`, or `utf8mb3` vs `utf8mb4` columns, silently kill index use on the converted side. `[ORA]` implicit `TO_NUMBER` on VARCHAR2 columns; `[PG]` refuses many implicit casts (errors instead of silently scanning) but still can't use the index if you cast the column.

## SARGability: Search ARGument ABLE
A predicate is SARGable if the engine can turn it into an index seek with the **bare column on one side**.
```sql
-- non-sargable                                   -- sargable
WHERE DATE(created_at) = '2025-03-01'             WHERE created_at >= '2025-03-01' AND created_at < '2025-03-02'
WHERE LOWER(email) = 'a@x.com'                    WHERE email = 'a@x.com'  (case-insensitive collation) or functional index
WHERE IFNULL(discount,0) > 10                     WHERE discount > 10                (NULL is not > 10 anyway)
WHERE id + 1 = 100                                WHERE id = 99
WHERE SUBSTRING(code,1,3) = 'ABC'                 WHERE code LIKE 'ABC%'
WHERE CONCAT(first,' ',last) = 'A B'              WHERE first='A' AND last='B'
WHERE created_at BETWEEN '2025-03-01' AND '2025-03-31'   -- BUG if column is DATETIME: misses 03-31 00:00:01+. Use half-open [start, next_start)
```
**Half-open ranges** (`>= start AND < next_start`) are the canonical safe form for timestamps.

---

# 6. Query patterns

## 6.1 N+1 at the SQL level
App code (JPA lazy loading or hand-written loop) issues 1 query for the parent list and N queries for children:
```
SELECT * FROM orders WHERE customer_id = 42;          -- 1 query -> 200 orders
SELECT * FROM order_items WHERE order_id = 1;         -- x200
SELECT * FROM order_items WHERE order_id = 2;
...
```
Cost model: 201 round trips x (network ~0.5-2 ms + parse + lock/latch + fetch). At 1 ms RTT that is 200 ms of pure latency though every query is "fast" in the slow log (each 0.1 ms, invisible in a slow-query log!). Detect: the **query count per request** in APM, `performance_schema.events_statements_summary_by_digest` with huge `COUNT_STAR` and tiny `AVG_TIMER_WAIT`, Hibernate `hibernate.generate_statistics`.

Fixes:
```sql
-- one JOIN (watch row multiplication: 1 order x k items = k rows)
SELECT o.id, o.total, i.product_id, i.qty
FROM orders o JOIN order_items i ON i.order_id = o.id WHERE o.customer_id = 42;
-- or two queries with IN (batch): 2 round trips, no multiplication
SELECT * FROM order_items WHERE order_id IN (1,2,...,200);   -- Hibernate @BatchSize / JOIN FETCH / EntityGraph
```
Index requirement: `order_items(order_id)`. Cartesian trap: joining two collections (`items` and `payments`) in one query multiplies rows (k x m); batch them separately.

## 6.2 Pagination
### OFFSET cost
```sql
SELECT * FROM t ORDER BY created, id LIMIT 20 OFFSET 900000;
```
The engine must **produce and discard 900,000 rows** (and with `SELECT *` via a secondary index, do 900,000 bookmark lookups first) before returning 20. Cost grows linearly with page number; deep pages by crawlers/exports melt the DB. Also unstable under concurrent inserts (rows shift between pages -> duplicates/skips).

**Measured [VALIDATED, SQLite in-memory, 1,000,000 rows x ~200-byte payload, index on (created,id)]:**
```
offset 10               ~ 0-1 ms
offset 900000           ~ 27-35 ms      (walks 900k index entries + row lookups, discarding them)
deferred join @ 900000  ~ 8-13 ms       (walks 900k *index only* entries, fetches 20 rows)
keyset after row 900000 ~ 0 ms          (index seek straight to the position)
```
Absolute times are tiny (in-memory, no network); the **shape** is the point - on a disk-bound MySQL the gap widens to seconds.

### Keyset (seek) pagination
Remember the last row's sort key; next page = "rows after it".
```sql
-- page 1
SELECT id, created, title FROM t ORDER BY created, id LIMIT 20;
-- next page, client passes last_created, last_id  (tie-breaker id makes ordering total & unique)
SELECT id, created, title FROM t
WHERE (created, id) > (?, ?)         -- row-constructor comparison, works in PG, SQLite, MySQL; MySQL's optimizer handles it as range since 5.7, verify with EXPLAIN
ORDER BY created, id LIMIT 20;
-- portable expanded form:
--   WHERE created > ? OR (created = ? AND id > ?)      (some optimizers, e.g. SQLite, fail to seek with the OR form: I measured a full index scan)
```
Needs an index on the sort key **including the tiebreaker**. Limits: no "jump to page 87", only next/prev (prev = reverse comparison + reverse order, then flip). Cursor token = base64 of (created,id). Best for infinite scroll, APIs, batch exports. `[ORA]` 12c+: `OFFSET n ROWS FETCH NEXT 20 ROWS ONLY`; older: `ROWNUM` wrapped subquery. Oracle has no row-constructor `>` on tuples in all contexts; use expanded form.

### Deferred join (late row lookup) - when you must keep OFFSET
```sql
SELECT t.* FROM t
JOIN (SELECT id FROM t ORDER BY created, id LIMIT 20 OFFSET 900000) x ON x.id = t.id
ORDER BY t.created, t.id;
```
The inner query scans a **covering index** `(created,id)` (no row fetches for discarded rows), then fetches only 20 full rows.

## 6.3 COUNT(*) cost
- `COUNT(*)` counts rows; `COUNT(1)` is identical in every mainstream engine; `COUNT(col)` skips NULLs (and may pick a different index); `COUNT(DISTINCT col)` needs dedupe.
- **InnoDB does not store the row count** (MVCC: different transactions see different counts), so `SELECT COUNT(*) FROM big` scans the *smallest* index (it can use any secondary index, which is narrower than the clustered one). ~100M rows: seconds to minutes. MyISAM stored an exact count (but no transactions). `[PG]` also scans (index-only scan if visibility map fresh). `[ORA]` scans too (bitmap/small index).
- Alternatives: estimates (`information_schema.TABLES.TABLE_ROWS`, `[PG]` `pg_class.reltuples`) for UI "about 1.2M results"; a **counter table** updated transactionally (hot row contention: shard the counter into 16 slots); Redis counters; cap the count: `SELECT COUNT(*) FROM (SELECT 1 FROM t WHERE ... LIMIT 1001) x` -> "1000+".
- Pagination UIs: do not run `COUNT(*)` for total pages on every request.

## 6.4 IN vs EXISTS vs JOIN
| Need | Preferred | Why |
|---|---|---|
| Customers that have at least one order | `EXISTS` (semi-join) | stops at first match, no duplicate rows |
| Same, using JOIN | `JOIN ... ` + `DISTINCT` / `GROUP BY` | duplicates if many orders: extra dedupe step, and wrong counts if you forget |
| Customers with no orders | `NOT EXISTS` or `LEFT JOIN ... IS NULL` (anti-join) | NULL-safe |
| `NOT IN (subquery)` | **avoid** | any NULL in the subquery makes result empty (see 7.1) |
| Need columns from the other table | `JOIN` | |

Modern optimizers (MySQL 8, PG, Oracle) rewrite `IN (subquery)` and `EXISTS` to the same semi-join plan; **differences are mostly about semantics, not speed**. MySQL 5.5 and earlier executed `IN (subquery)` as a dependent subquery per row: the old advice "EXISTS is faster" comes from then. Check with EXPLAIN rather than folklore.

```sql
-- semi-join (each customer at most once)
SELECT c.* FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
-- anti-join
SELECT c.* FROM customers c WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
SELECT c.* FROM customers c LEFT JOIN orders o ON o.customer_id = c.id WHERE o.id IS NULL;
```
Both anti-join forms are validated below (7.1) - they return customer C.

## 6.5 Correlated subquery rewrites
A correlated subquery references the outer row; naive plan = run once per outer row (O(n x m) without index, O(n log m) with).
```sql
-- correlated: employees above their department average
SELECT name, dept, salary FROM emp e WHERE salary > (SELECT AVG(salary) FROM emp WHERE dept = e.dept);
-- rewrite 1: join to pre-aggregated derived table (aggregate once per dept)
SELECT e.name, e.dept, e.salary
FROM emp e JOIN (SELECT dept, AVG(salary) a FROM emp GROUP BY dept) d ON d.dept = e.dept
WHERE e.salary > d.a;
-- rewrite 2: window function (single pass)
SELECT name, dept, salary FROM (SELECT name, dept, salary, AVG(salary) OVER (PARTITION BY dept) a FROM emp) t WHERE salary > a;
```
All three give (validated): `Asha ENG 300, Esha HR 120, Hari OPS 110, Indu OPS 130`. Scalar subquery in SELECT list (`SELECT c.id, (SELECT COUNT(*) FROM orders o WHERE o.customer_id=c.id) cnt`) = N executions; replace with `LEFT JOIN ... GROUP BY` or a window. Recent MySQL 8.0 releases can decorrelate some scalar subqueries; do not assume it, check EXPLAIN.

---

# 7. SQL semantics that interviews probe

## 7.1 NULL and three-valued logic
Every comparison yields TRUE, FALSE or **UNKNOWN**; `WHERE` keeps only TRUE.
```
NULL = NULL     -> UNKNOWN          x IS NULL    -> TRUE/FALSE only
1 IN (1, NULL)  -> TRUE             2 IN (1, NULL)     -> UNKNOWN   (not FALSE)
2 NOT IN (1, NULL) -> UNKNOWN  ==> row rejected!
```
[VALIDATED, SQLite]: `select null=null, null is null, 1 in (1,null), 2 in (1,null), 2 not in (1,null)` -> `NULL | 1 | 1 | NULL | NULL`.

**NOT IN trap** [VALIDATED]. `cust` = A(1), B(2), C(3); `ord.cid` = 1, 1, 2, **NULL**:
```sql
SELECT COUNT(*) FROM cust WHERE id NOT IN (SELECT cid FROM ord);                 -- 0   (!!)  C should qualify
SELECT COUNT(*) FROM cust c WHERE NOT EXISTS (SELECT 1 FROM ord o WHERE o.cid=c.id);  -- 1   (correct)
```
Why: `3 NOT IN (1,1,2,NULL)` = `3<>1 AND 3<>1 AND 3<>2 AND 3<>NULL` = TRUE AND ... AND UNKNOWN = UNKNOWN. Fix: `NOT EXISTS`, or `NOT IN (SELECT cid FROM ord WHERE cid IS NOT NULL)`.

Other NULL facts:
- `COUNT(*)` counts rows, `COUNT(col)` counts non-NULL: validated `select count(*), count(cid) from ord` -> `4 | 3`.
- `SUM/AVG/MIN/MAX` ignore NULLs; `SUM` of no rows (or all NULL) is NULL, not 0: use `COALESCE(SUM(x),0)`. `AVG(col)` divides by non-NULL count.
- `NULL` in `ORDER BY`: MySQL/SQLite sort NULLs first for ASC, `[PG]` last for ASC (`NULLS FIRST/LAST` supported in PG/Oracle, not MySQL: emulate with `ORDER BY col IS NULL, col`).
- `GROUP BY`/`DISTINCT` treat NULLs as one group. `UNIQUE` index: MySQL/PG allow multiple NULLs; SQL Server allows one.
- `[ORA]` **empty string `''` is NULL**; length('') is NULL. A classic Oracle-migration bug.
- `NULL` in string concat: `CONCAT('a',NULL)` = NULL (MySQL); `[PG]` `||` yields NULL but `concat()` ignores NULL; `[ORA]` `||` ignores NULL.
- `LEFT JOIN` then filter on right-table column in `WHERE` turns it into an INNER JOIN. Put the filter in `ON`.

## 7.2 GROUP BY / HAVING semantics
Logical processing order (not the order you write): `FROM/JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT (window functions evaluated here) -> DISTINCT -> ORDER BY -> LIMIT`.
- `WHERE` filters rows before grouping, cannot reference aggregates; `HAVING` filters groups after and can. Put non-aggregate conditions in `WHERE` (fewer rows grouped, can use index).
- Column aliases: usable in `ORDER BY` (all engines), `GROUP BY`/`HAVING` in MySQL (extension), not in `WHERE`.
- `ONLY_FULL_GROUP_BY` (MySQL 5.7+ default): every selected non-aggregated column must be functionally dependent on the GROUP BY columns; older MySQL returned an arbitrary row's value silently. `ANY_VALUE(col)` opts out explicitly.
- MySQL 8 no longer sorts `GROUP BY` results implicitly: add `ORDER BY`.
- `GROUP BY` uses index order if an index matches (loose/tight index scan) - otherwise a temp table (`Using temporary`).
- Validated: `select dept,count(*),avg(salary) from emp group by dept having count(*)>=3` -> `ENG 4 212.5 | HR 3 100.0 | OPS 3 106.67`.

## 7.3 Joins (with semi/anti)
```
INNER      A ∩ B rows matching predicate
LEFT       all A; B columns NULL where no match          (anti-join = LEFT + WHERE b.pk IS NULL)
RIGHT      mirror of LEFT (rarely used; swap tables)
FULL OUTER all A and B  ([PG]/[ORA] native; MySQL: LEFT UNION RIGHT)
CROSS      m x n (calendar x product grids, generating series)
SELF       table joined to itself with aliases (manager, adjacent rows)
SEMI       "A rows that have a match in B", each A once  (EXISTS / IN)   - no SQL keyword, planner concept
ANTI       "A rows with no match in B"                   (NOT EXISTS / LEFT..IS NULL)
```
MySQL FULL OUTER: `SELECT ... FROM a LEFT JOIN b ON ... UNION SELECT ... FROM a RIGHT JOIN b ON ...` (use `UNION ALL` plus `WHERE a.id IS NULL` on the second half to avoid dedupe cost).
`USING(col)` vs `ON`: `USING` merges the column in `SELECT *`. `NATURAL JOIN` joins on all same-named columns: schema changes silently change semantics; avoid.

## 7.4 UNION vs UNION ALL
`UNION` = concatenate + **DISTINCT** (sort/hash + temp table); `UNION ALL` = concatenate only. Use `UNION ALL` unless you need dedupe (validated: two identical 7-row selects: `UNION` -> 3 rows; `UNION ALL` -> 14). Column count/type must match; names come from the first select; `ORDER BY`/`LIMIT` apply to the whole union unless parenthesised. `INTERSECT`/`EXCEPT` exist in MySQL 8.0.31+, PG (both); Oracle spells EXCEPT as `MINUS`.

## 7.5 Window functions
Compute over a set of rows related to the current row **without collapsing** rows (unlike GROUP BY). Syntax: `func() OVER (PARTITION BY ... ORDER BY ... frame)`. MySQL 8.0+, PG 8.4+, Oracle 8i+, SQLite 3.25+.

**Ranking** [VALIDATED] on `emp` ENG dept salaries 300, 200, 200, 150:
```sql
SELECT name, salary,
  ROW_NUMBER() OVER w rn, RANK() OVER w rk, DENSE_RANK() OVER w dr
FROM emp WHERE dept='ENG' WINDOW w AS (ORDER BY salary DESC);
```
```
name   | salary | rn | rk | dr
Asha   | 300    | 1  | 1  | 1
Bala   | 200    | 2  | 2  | 2
Chitra | 200    | 3  | 2  | 2        <- tie: ROW_NUMBER arbitrary among ties (add tie-breaker!), RANK repeats
Dev    | 150    | 4  | 4  | 3        <- RANK skips (4), DENSE_RANK does not (3)
```
`ROW_NUMBER` unique sequence; `RANK` ties share rank then gap; `DENSE_RANK` ties share, no gap; `NTILE(n)` buckets; `PERCENT_RANK`, `CUME_DIST`.

**LAG / LEAD** [VALIDATED] `pay(id, amount)` = 100, 50, 70, 30:
```sql
SELECT id, amount, LAG(amount) OVER (ORDER BY id) prev, LEAD(amount) OVER (ORDER BY id) nxt,
       amount - LAG(amount) OVER (ORDER BY id) diff FROM pay;
```
```
id | amount | prev | nxt  | diff
1  | 100    | NULL | 50   | NULL
2  | 50     | 100  | 70   | -50
3  | 70     | 50   | 30   | 20
4  | 30     | 70   | NULL | -40
```
`LAG(col, n, default)` offset and default value.

**Aggregates with frames**
```sql
SUM(amount) OVER (ORDER BY id)                                           -- running total; default frame with ORDER BY = RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
AVG(amount) OVER (ORDER BY id ROWS BETWEEN 1 PRECEDING AND CURRENT ROW)  -- moving average of 2
SUM(x) OVER (PARTITION BY dept)                                          -- no ORDER BY: whole partition
```
Validated running total: `100, 150, 220, 250`; moving avg (2 rows): `100.0, 75.0, 60.0, 50.0`.
**ROWS vs RANGE:** `ROWS` counts physical rows; `RANGE` includes all **peer rows with equal ORDER BY value**. The default frame is RANGE, so with duplicate sort keys `SUM(x) OVER (ORDER BY d)` gives tied rows the same running total; if you want strictly row-by-row, write `ROWS UNBOUNDED PRECEDING` (also faster: RANGE needs peer tracking). `GROUPS` frames exist in PG/SQLite, not MySQL.
Window functions cannot appear in `WHERE`: wrap in a subquery/CTE (`WHERE rn = 1` pattern). `FIRST_VALUE`, `LAST_VALUE` (default frame ends at CURRENT ROW, so `LAST_VALUE` surprises: use `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`), `NTH_VALUE`.

## 7.6 CTEs and recursive CTEs
```sql
WITH paid AS (SELECT customer_id, SUM(total) s FROM orders WHERE status='PAID' GROUP BY customer_id)
SELECT c.name, p.s FROM customers c JOIN paid p ON p.customer_id = c.id;
```
CTE = named temporary result for readability; MySQL 8 / PG 12+ may inline or materialize (PG 12+: `MATERIALIZED` / `NOT MATERIALIZED` hints; PG <12 always materialized = optimization fence). Not automatically faster than a subquery.

**Recursive CTE - org hierarchy** [VALIDATED on emp table: Asha(CEO) -> Bala, Chitra, Esha, Hari, Indu; Bala -> Dev; Esha -> Farid, Gita; Indu -> Jai]:
```sql
WITH RECURSIVE t(id, name, lvl, path) AS (
  SELECT id, name, 0, name FROM emp WHERE manager_id IS NULL            -- anchor
  UNION ALL
  SELECT e.id, e.name, t.lvl + 1, t.path || ' > ' || e.name            -- recursive step (MySQL: CONCAT(t.path,' > ',e.name))
  FROM emp e JOIN t ON e.manager_id = t.id
) SELECT * FROM t ORDER BY path;
```
```
id | name   | lvl | path
1  | Asha   | 0   | Asha
2  | Bala   | 1   | Asha > Bala
4  | Dev    | 2   | Asha > Bala > Dev
3  | Chitra | 1   | Asha > Chitra
5  | Esha   | 1   | Asha > Esha
6  | Farid  | 2   | Asha > Esha > Farid
7  | Gita   | 2   | Asha > Esha > Gita
8  | Hari   | 1   | Asha > Hari
9  | Indu   | 1   | Asha > Indu
10 | Jai    | 2   | Asha > Indu > Jai
```
Execution: anchor produces the seed set; each iteration joins the *last iteration's new rows* to the table until no new rows; `UNION` (not ALL) dedupes and can stop cycles. Cycle guard: depth limit `WHERE t.lvl < 20`, MySQL `cte_max_recursion_depth` (default 1000), PG `CYCLE` clause (14+). MySQL syntax needs `RECURSIVE` keyword (Oracle 11g2+ doesn't; PG and SQLite do). `[ORA]` also has `CONNECT BY PRIOR ... START WITH`. Alternatives for deep hierarchies read often: closure table, materialized path (`path LIKE '1/2/%'`), nested sets.

---

# 8. Transactions: ACID, MVCC, isolation levels

## 8.1 ACID mapped to mechanisms
| Property | Meaning | InnoDB mechanism |
|---|---|---|
| **A**tomicity | all or nothing | **undo log** rolls back partial work |
| **C**onsistency | constraints/invariants hold before and after | constraints (PK/FK/CHECK/UNIQUE) + application logic; "C" is a contract, not something the engine gives for free |
| **I**solation | concurrent tx behave as if serial (to a degree) | **MVCC** (reads) + **locks** (writes) |
| **D**urability | committed data survives crash | **redo log** fsync (+ binlog) |

## 8.2 MVCC step by step (InnoDB)
**Ingredients**
- Every clustered-index row has hidden columns: `DB_TRX_ID` (6 B, id of the last transaction that inserted/updated it), `DB_ROLL_PTR` (7 B, pointer to the **undo record** holding the previous version), and `DB_ROW_ID` (6 B, only when there is no PK).
- **Undo log** (in undo tablespaces) stores old versions -> forms a **version chain** per row.
- A **read view** (snapshot) = `{creator_trx_id, m_ids = ids of transactions active (uncommitted) at creation, up_limit_id = min(m_ids), low_limit_id = next trx id to be assigned}`.

**Visibility rule for a row version with trx id `X`**
```
X == creator_trx_id           -> visible (my own change)
X <  up_limit_id              -> visible (committed before every active tx started)
X >= low_limit_id             -> NOT visible (started after my snapshot)
X in m_ids                    -> NOT visible (was uncommitted when snapshot taken)
otherwise                     -> visible (committed before snapshot)
if not visible: follow DB_ROLL_PTR to the older version and re-test
```
**Trace** (row `id=1, balance=100`, last written by committed trx 90). Session A = reader, Session B = trx 105.
```
1. A: START TRANSACTION; SELECT balance FROM acct WHERE id=1;
   -> read view created lazily at first consistent read (RR): m_ids={105 (B, active)}, up_limit=105, low_limit=106
      version: (balance=100, trx 90)   90 < 105 -> visible -> A sees 100
2. B: UPDATE acct SET balance=150 WHERE id=1;   (B takes an X record lock)
      clustered row becomes (balance=150, DB_TRX_ID=105, DB_ROLL_PTR --> undo)   undo: (balance=100, trx 90)
      redo records written; not yet committed
3. A: SELECT balance ...   row head trx 105 in m_ids -> invisible -> follow roll ptr -> (100, trx 90) visible -> 100   (no blocking!)
4. B: COMMIT.
5. A: SELECT balance ... (REPEATABLE READ) same read view: 105 still in m_ids -> invisible -> reads 100  (repeatable)
   Under READ COMMITTED: a NEW read view is created per statement: 105 no longer active and < low_limit -> visible -> reads 150
6. A: COMMIT. Purge thread: undo for trx 105's old version no longer needed by any read view -> freed.
```
```
Version chain (newest first):
 clustered leaf: [id=1 | bal=150 | trx=105 | roll_ptr] --> undo: [bal=100 | trx=90 | roll_ptr] --> undo: [bal=80 | trx=70 | ...]
```
- **DELETE** = sets a *delete-mark*; physically removed by the **purge** thread once no read view can see the row. **UPDATE of PK** = delete-mark + insert.
- Long-running transactions (or forgotten `START TRANSACTION`, or a stuck consistent snapshot in a reporting tool) block purge: **history list length** grows (`SHOW ENGINE INNODB STATUS`, `information_schema.INNODB_METRICS trx_rseg_history_len`), undo tablespace bloats, reads have to walk long chains.
- Readers never block writers and writers never block readers **for consistent (plain) reads**. Writers still block writers.
- `[PG]` MVCC keeps old row versions *in the table heap* (tuple headers `xmin`/`xmax`); updates create a new heap tuple (HOT updates avoid touching indexes when possible); **VACUUM** reclaims dead tuples; bloat and transaction-id wraparound are its unique failure modes. `[ORA]` old versions live in undo segments and the read is made consistent by SCN; error `ORA-01555 snapshot too old` if undo overwritten.

## 8.3 Isolation levels (SQL standard vs reality)
| Level | Dirty read | Non-repeatable read | Phantom | Notes |
|---|---|---|---|---|
| READ UNCOMMITTED | possible | possible | possible | reads latest version regardless of commit (rarely used) |
| READ COMMITTED | no | possible | possible | new snapshot per statement. **Default in PG and Oracle** |
| REPEATABLE READ | no | no | mostly no in InnoDB (see 8.5) | one snapshot per transaction. **Default in MySQL/InnoDB** |
| SERIALIZABLE | no | no | no | InnoDB: plain SELECT becomes `SELECT ... FOR SHARE` (when autocommit off) -> lots of blocking; `[PG]` SSI (predicate locks, aborts with serialization failure `40001`: retry!) ; `[ORA]` "serializable" is really snapshot isolation |

Anomalies the standard table forgets: **lost update** and **write skew**. Snapshot isolation (`[PG]` REPEATABLE READ, `[ORA]` SERIALIZABLE) allows write skew but PG detects lost update (error "could not serialize access due to concurrent update"). InnoDB REPEATABLE READ does **not** detect an application-level lost update.

Set: `SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;` (next tx: `SET TRANSACTION ...`), global default `transaction_isolation`. Spring: `@Transactional(isolation = Isolation.READ_COMMITTED)` (see `Spring_Transactional.md`).

## 8.4 Two-session timelines
Schema for all: `acct(id PK, balance INT)`; rows `(1, 100)`; `doctor(id PK, on_call TINYINT)`.
Time flows downward. `S1`/`S2` are two connections.

### a) Dirty read (READ UNCOMMITTED only)
```
t  S1                                          S2
1  SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED; START TRANSACTION;
2                                              START TRANSACTION; UPDATE acct SET balance=0 WHERE id=1;   (uncommitted)
3  SELECT balance FROM acct WHERE id=1;   --> 0   (DIRTY: S2 has not committed)
4                                              ROLLBACK;
5  S1 acted on a value that never existed.
```
Under RC/RR at t3 S1 reads 100. **Wrong answer to avoid:** "InnoDB RC has dirty reads": no, RC only reads committed data.

### b) Non-repeatable read (READ COMMITTED)
```
t  S1 (RC)                                     S2
1  START TRANSACTION;
2  SELECT balance FROM acct WHERE id=1;  --> 100
3                                              UPDATE acct SET balance=150 WHERE id=1; COMMIT;
4  SELECT balance FROM acct WHERE id=1;  --> 150   (same query, different result inside one tx)
```
Under REPEATABLE READ S1 reads 100 at t4. Impact: reports that read the same table twice within one transaction get inconsistent totals; RC is usually fine for OLTP where each statement is self-contained.

### c) Phantom read
```
t  S1                                          S2
1  START TRANSACTION;
2  SELECT COUNT(*) FROM acct WHERE balance>=100;  --> 1
3                                              INSERT INTO acct VALUES (2, 500); COMMIT;
4  SELECT COUNT(*) FROM acct WHERE balance>=100;
      RC: --> 2  (phantom row appeared)
      InnoDB RR (plain SELECT): --> 1  (same snapshot: no phantom)
5  UPDATE acct SET balance = balance+1 WHERE balance>=100;   -- current read: touches BOTH rows (affected rows 2)
6  SELECT COUNT(*) FROM acct WHERE balance>=100;  --> 2   (row 2 now carries S1's own trx_id -> visible: phantom surfaces in RR!)
```
This is the known InnoDB caveat: **snapshot reads see no phantoms, but a write in the same transaction can "adopt" rows inserted by others**. Locking reads (`FOR UPDATE`) also see the newest rows. To truly block inserts into the range at t3, S1 must use a locking read at t2 (next-key locks, section 9).

### d) Lost update (read-modify-write in the application)
```
t  S1                                           S2
1  START TRANSACTION;                            START TRANSACTION;
2  SELECT balance FROM acct WHERE id=1; -->100   SELECT balance FROM acct WHERE id=1; --> 100
3  app computes 100+50 = 150
4                                                app computes 100+30 = 130
5  UPDATE acct SET balance=150 WHERE id=1;       (S2 blocks on S1's row lock)
6  COMMIT;                                       (S2 wakes up)  UPDATE acct SET balance=130 WHERE id=1; COMMIT;
   Final = 130. S1's +50 lost. (InnoDB RR/RC and Oracle default: no error.)   [PG RR: S2's UPDATE fails with 40001 -> retry]
```
Fixes, cheapest first:
1. **Atomic update:** `UPDATE acct SET balance = balance + 50 WHERE id=1;` (uses the current value under the row lock; correct in every isolation level).
2. **Pessimistic:** `SELECT ... FOR UPDATE` at t2, so S2 blocks at its SELECT and reads 150 after S1 commits.
3. **Optimistic:** `version` column: `UPDATE acct SET balance=?, version=version+1 WHERE id=1 AND version=?` and check affected rows = 1 (JPA `@Version`).
4. `SERIALIZABLE` / PG retry loop.

### e) Write skew (invariant across rows, snapshot isolation)
Rule: at least one doctor must stay on call. Rows: Asha(on_call=1), Bala(on_call=1).
```
t  S1 (RR)                                             S2 (RR)
1  START TRANSACTION;                                   START TRANSACTION;
2  SELECT COUNT(*) FROM doctor WHERE on_call=1; -->2    SELECT COUNT(*) FROM doctor WHERE on_call=1; -->2
3  (2 >= 2, safe to leave)                              (2 >= 2, safe to leave)
4  UPDATE doctor SET on_call=0 WHERE name='Asha';       UPDATE doctor SET on_call=0 WHERE name='Bala';   (different rows: no lock conflict!)
5  COMMIT;                                              COMMIT;
   Result: 0 doctors on call. Invariant broken; each tx individually valid, no row was written twice.
```
Fixes: lock the *set that the decision reads*: `SELECT ... WHERE on_call=1 FOR UPDATE` (S2 blocks at t2 until S1 commits, then its current read sees count 1 and it refuses); or a **materialized conflict** row (single `on_call_summary` row updated by both -> row lock conflict); or `SERIALIZABLE` (InnoDB: shared locks -> deadlock, retry; PG SSI: one aborts); or a DB constraint/trigger where expressible.

## 8.5 InnoDB REPEATABLE READ specifics
- **Consistent (snapshot) read**: plain `SELECT`. Snapshot created at the **first consistent read** in the transaction (not at `START TRANSACTION`; use `START TRANSACTION WITH CONSISTENT SNAPSHOT` to pin it immediately). Nobody's later commits are visible.
- **Locking / current read**: `SELECT ... FOR UPDATE`, `FOR SHARE`, `UPDATE`, `DELETE`, `INSERT ... SELECT` source. They read the **latest committed** version and take locks; they ignore your snapshot. A transaction can therefore see two "versions of reality" (5-6 in 8.4c).
- In RR, locking reads take **next-key locks** (record + gap before it) on scanned index ranges to stop other sessions inserting into the range, which is how InnoDB prevents phantoms for locking statements. In RC, only **record locks** (gap locks disabled except for FK/duplicate checks), which lowers deadlocks/contention but permits phantoms. Many high-throughput shops run MySQL at RC + row-based binlog for that reason.
- MySQL replication: statement-based binlog required RR/SERIALIZABLE for safety; with `binlog_format=ROW` (default) RC is safe.

---

# 9. InnoDB locking

## 9.1 Lock types
| Lock | What it protects | Notes |
|---|---|---|
| **Shared (S)** / **Exclusive (X)** | row read-for-share / write | S-S compatible, everything else conflicts |
| **Record lock** | one index record | on the *index entry*; without a usable index InnoDB locks via the clustered index and effectively every scanned record |
| **Gap lock** | the open interval between two index records (or before first / after last) | purpose: block inserts into the gap. Gap locks do **not conflict with each other** (S gap and X gap are compatible) |
| **Next-key lock** | record + gap before it: `(prev, rec]` | default in RR for index scans |
| **Insert intention lock** | a gap lock flavour taken by INSERT before inserting | conflicts with gap locks held by others (source of insert deadlocks); insert intention locks do not conflict with each other |
| **Intention locks IS / IX** | table-level "I plan to lock rows in this table" | allow cheap conflict check with table locks; never block row operations |
| **Table lock** | `LOCK TABLES`, DDL metadata locks (**MDL**), AUTO-INC lock | MDL is not an InnoDB row lock: a pending `ALTER` queues behind long transactions and blocks new queries (see war story 15.4) |

## 9.2 Next-key / gap ranges with an example
Table `t(id PK, k INT)` with `id` values **10, 20, 30** (unique PK index). The key space is split into intervals:
```
   (-inf,10]   (10,20]   (20,30]   (30,+inf)
  ---|-------|---|------|---|------|--------
     10          20          30          supremum
```
| Statement in RR | Locks taken |
|---|---|
| `WHERE id=20 FOR UPDATE` (unique, exists) | record lock on 20 only (no gap) |
| `WHERE id=15 FOR UPDATE` (unique, not found) | gap lock on (10,20) -> inserts of 11..19 block |
| `WHERE id BETWEEN 15 AND 25 FOR UPDATE` | next-key locks covering (10,20] and (20,30] (the scan must look at 30 to know it is done; exact treatment of the boundary record varies slightly by version), so inserting id 12, 25 or updating 20 blocks; inserting 35 does not |
| `WHERE id > 30 FOR UPDATE` | next-key (30,+inf) incl. supremum: blocks all inserts of ids > 30 |
| non-unique `WHERE k=5 FOR UPDATE` with k entries 3,5,9 | next-key on the k=5 entries + gap lock (5,9) + record locks on their PKs |
| `WHERE unindexed_col=5 FOR UPDATE` | **every scanned row and gap**: behaves like a table lock. Classic production incident: missing index makes a "row lock" statement freeze a table |
In RC: only record locks on matching rows (plus rows examined, released early for non-matching rows via "semi-consistent read" on UPDATE).

Why phantoms are mostly prevented: for range **locking** reads, the gap locks physically prevent other sessions from inserting into the range; for **plain** reads the snapshot hides new rows. The leak is mixing the two in one transaction (8.4c).

## 9.3 FOR UPDATE / FOR SHARE / NOWAIT / SKIP LOCKED
```sql
SELECT ... FOR UPDATE;                 -- X locks: "I will modify these rows"
SELECT ... FOR SHARE;                  -- S locks (8.0; older syntax LOCK IN SHARE MODE): "these rows must not change until I commit"
SELECT ... FOR UPDATE NOWAIT;          -- fail immediately with error 3572 instead of waiting   (MySQL 8.0.1+, PG, Oracle)
SELECT ... FOR UPDATE SKIP LOCKED;     -- silently skip rows others have locked (MySQL 8.0.1+, PG 9.5+, Oracle)
SELECT ... FOR UPDATE OF t1 ...;       -- lock only rows of a joined table (8.0)
```
**Job-queue pattern (multiple workers, no double pick-up):**
```sql
START TRANSACTION;
SELECT id, payload FROM jobs WHERE status='NEW' ORDER BY id LIMIT 10 FOR UPDATE SKIP LOCKED;   -- index (status,id)
UPDATE jobs SET status='RUNNING', worker='w7' WHERE id IN (...those ids...);
COMMIT;                                  -- keep this transaction tiny; do the slow work outside
```
`SKIP LOCKED` returns an inconsistent view by design: fine for queues, wrong for reports. Prefer optimistic `UPDATE ... SET status='RUNNING' WHERE id=? AND status='NEW'` (check affected rows) when contention is low.

## 9.4 Deadlock: example and detection
**Classic ordering deadlock**
```
t  S1                                               S2
1  START TRANSACTION;                                START TRANSACTION;
2  UPDATE acct SET balance=balance-10 WHERE id=1;   (S1 holds X on id=1)
3                                                    UPDATE acct SET balance=balance-20 WHERE id=2;   (S2 holds X on id=2)
4  UPDATE acct SET balance=balance+10 WHERE id=2;   -> waits for S2
5                                                    UPDATE acct SET balance=balance+20 WHERE id=1;   -> waits for S1  => CYCLE
6  InnoDB detects the cycle in the wait-for graph at t5 and rolls back one:
   ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```
**Detection:** InnoDB maintains a wait-for graph; with `innodb_deadlock_detect=ON` (default) a cycle is found immediately when a lock wait is about to start. **Victim** = the transaction that is cheapest to roll back: fewer rows changed / smaller "weight" (undo records + locks), *not* simply the youngest. The **whole transaction** of the victim is rolled back (unlike lock wait timeout below). Applications must catch 1213 and **retry the whole transaction**. On extremely hot rows detection itself is costly: `innodb_deadlock_detect=OFF` + small `innodb_lock_wait_timeout` is an option for very high concurrency.
`[PG]` detects via `deadlock_timeout` (1 s) then aborts the transaction that noticed; error `40P01`. `[ORA]` ORA-00060, rolls back only the *statement*.

**Gap-lock insert deadlock (RR, no row exists):**
```
t  S1                                                     S2
1  START TRANSACTION;                                      START TRANSACTION;
2  SELECT * FROM t WHERE id=15 FOR UPDATE;  (gap lock (10,20)) 
3                                                          SELECT * FROM t WHERE id=15 FOR UPDATE;  (gap locks are compatible: also granted)
4  INSERT INTO t VALUES (15,'a');  -> needs insert-intention lock, blocked by S2's gap lock
5                                                          INSERT INTO t VALUES (15,'b');  -> blocked by S1's gap lock => deadlock
```
Fix: don't check-then-insert with locking reads; use `INSERT ... ON DUPLICATE KEY UPDATE` / `INSERT IGNORE` / unique constraint + catch duplicate-key error, or run RC.

**Reading SHOW ENGINE INNODB STATUS \G** [ILLUSTRATIVE excerpt of the `LATEST DETECTED DEADLOCK` section]
```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2025-06-01 10:15:22 140217 
*** (1) TRANSACTION:
TRANSACTION 4711, ACTIVE 3 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1
UPDATE acct SET balance=balance+10 WHERE id=2
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 58 page no 4 n bits 72 index PRIMARY of table `shop`.`acct` trx id 4711 lock_mode X locks rec but not gap
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS ... index PRIMARY ... trx id 4711 lock_mode X locks rec but not gap waiting
*** (2) TRANSACTION:
TRANSACTION 4712, ACTIVE 2 sec starting index read ...
UPDATE acct SET balance=balance+20 WHERE id=1
*** (2) HOLDS THE LOCK(S): ... id=2 ...
*** (2) WAITING FOR THIS LOCK TO BE GRANTED: ... id=1 ...
*** WE ROLL BACK TRANSACTION (2)
```
How to read: for each of the two transactions note the **statement**, the **lock it holds** and the **lock it waits for** (`index`, `lock_mode X`, `locks rec but not gap` = plain record lock, `locks gap before rec` = gap lock, no qualifier = next-key), and the last line naming the victim. Other sections: `TRANSACTIONS` (history list length, active tx), `SEMAPHORES` (mutex contention), `FILE I/O`, `BUFFER POOL AND MEMORY` (hit rate), `ROW OPERATIONS`. To log every deadlock: `SET GLOBAL innodb_print_all_deadlocks=ON`. Live lock analysis (8.0):
```sql
SELECT * FROM performance_schema.data_locks;            -- every lock held/requested (LOCK_TYPE, LOCK_MODE 'X,REC_NOT_GAP', LOCK_DATA)
SELECT * FROM performance_schema.data_lock_waits;       -- who waits for whom
SELECT * FROM sys.innodb_lock_waits\G                   -- readable wait pairs with blocking pid and the KILL statement
```

## 9.5 Lock wait timeout
`innodb_lock_wait_timeout` default **50 s**. A statement waiting longer fails with `ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction`. By default only the **statement** is rolled back and the transaction stays open, still holding earlier locks (`innodb_rollback_on_timeout=OFF`): a common source of lingering locks; the app must roll back explicitly. Not a deadlock: usually a long transaction (idle in app, awaiting an HTTP call, or a forgotten commit) is holding a lock. Find the blocker via `sys.innodb_lock_waits`, `information_schema.INNODB_TRX` (`trx_started`, `trx_mysql_thread_id`), `SHOW PROCESSLIST` (Sleep with old transaction).

## 9.6 Designing to avoid deadlocks and lock pain
1. **Consistent lock ordering**: always touch rows/tables in the same order (sort ids before locking: transfer between accounts locks `min(id)` then `max(id)`).
2. **Short transactions**: no remote calls, no user think-time, no big loops inside; batch work in chunks of ~1000 rows.
3. **Index the WHERE columns of every UPDATE/DELETE/locking SELECT** so the locked range is minimal (see missing-index-table-lock above).
4. Prefer **atomic single-statement updates** over read-then-write; `UPDATE ... WHERE stock >= n` and check affected rows.
5. Use RC when you do not need gap locks. Avoid `SELECT ... FOR UPDATE` on non-unique ranges under RR.
6. Lock the **parent** row first, then children; keep FK operations consistent.
7. Retry on 1213 / 1205 / PG 40001, 40P01 with jittered backoff (idempotent transaction bodies). In Spring: retry aspect/`@Retryable` around the *transaction boundary*, not inside it.
8. Split hot rows (counter shards), avoid hotspot `UPDATE counter SET n=n+1` on one row at high TPS.
9. Bulk `DELETE`/`UPDATE` in chunks (`LIMIT 1000` in a loop) to avoid lock table blow-up and replica lag.

---

# 10. Durability: WAL, redo, undo, binlog, crash recovery

## 10.1 The four logs
| Log | Layer | Content | Used for |
|---|---|---|---|
| **Redo log** (`#innodb_redo/` files, 8.0.30+ `innodb_redo_log_capacity`) | InnoDB | physical/logical page changes ("page X offset Y bytes") | crash recovery (roll forward). Sequential append -> fast |
| **Undo log** | InnoDB (undo tablespaces) | previous row versions | rollback, MVCC reads, crash recovery (roll back uncommitted) |
| **Binary log (binlog)** | MySQL server layer (engine independent) | logical changes (ROW format: before/after images) | replication, point-in-time recovery, CDC (Debezium reads binlog) |
| **Doublewrite buffer** | InnoDB | copy of flushed pages | protects against **torn page** writes (16 KB page vs 4 KB disk sector atomicity) |

**Write-ahead logging (WAL):** a change is first made to a page **in the buffer pool** (page becomes dirty) and described in the redo log **before** it is written to the data file. Data pages are flushed lazily in the background (checkpointing). At commit only the **redo log** must be durably on disk - a sequential write instead of many random page writes.

## 10.2 Commit path (with binlog) - two-phase commit
```
1. Transaction changes pages in buffer pool; writes undo + redo records to log buffer
2. COMMIT: redo log "PREPARE" record written + fsync (per innodb_flush_log_at_trx_commit)
3. binlog event written + fsync (per sync_binlog)          <- the commit point for replication/recovery
4. redo log "COMMIT" mark written
5. Locks released, client gets OK. Dirty pages flushed later.
```
Group commit batches many transactions per fsync (huge throughput win).

## 10.3 Crash recovery, conceptually
1. Start, find last **checkpoint LSN** in redo log.
2. **Redo phase**: replay redo records after the checkpoint to bring pages to the crash-time state (idempotent, page LSN compare). Torn pages restored from doublewrite buffer first.
3. Find transactions still active/prepared. For **prepared** ones consult the **binlog**: if the transaction's event is in the binlog -> commit it; else roll back. This keeps engine and binlog (hence replicas) consistent.
4. **Undo phase**: roll back uncommitted transactions using undo (done in background after the server accepts connections).
5. Purge of delete-marked rows continues later.

## 10.4 Durability knobs
| Setting | Values | Effect |
|---|---|---|
| `innodb_flush_log_at_trx_commit` | **1** (default): write + fsync redo at every commit - full ACID durability. **2**: write to OS cache at commit, fsync ~1/s: survives mysqld crash, may lose ~1 s on OS/power crash. **0**: write+fsync ~1/s: may lose 1 s even on mysqld crash | 1 for money, 2 acceptable for logs/analytics replicas |
| `sync_binlog` | **1** (default in 8.0): fsync binlog each commit group. 0: OS decides | `1` + `flush_log_at_trx_commit=1` = "double 1", no committed tx lost, replica-safe |
| `innodb_flush_method` | `O_DIRECT` typical on Linux | bypass OS page cache for data files |
| `innodb_doublewrite` | ON | keep ON unless storage guarantees atomic 16 KB writes |
| `binlog_format` | ROW (default), STATEMENT, MIXED | ROW is deterministic; larger logs |
`[PG]` WAL, `synchronous_commit` (on/remote_write/local/off), `wal_level`, checkpoints, `full_page_writes`. `[ORA]` redo log groups + archived redo, `COMMIT WRITE BATCH NOWAIT`.

---

# 11. Replication, partitioning vs sharding, pooling

## 11.1 Replication
```
Primary: writes -> redo -> binlog ----network----> Replica: I/O thread -> relay log -> SQL/applier threads (parallel) -> data
```
- **Asynchronous (default)**: primary commits and returns without waiting for replicas. Fast; on primary crash, last transactions may be **lost** on promotion (data loss window = lag).
- **Semi-synchronous**: primary waits until at least one replica has *received and written to relay log* (not applied). Default `AFTER_SYNC` (lossless): wait before engine commit, so no client ever saw a commit that replicas lack. Cost: +1 network RTT per commit; falls back to async after `rpl_semi_sync_*_timeout` (default 10 s).
- **Group Replication / InnoDB Cluster / Galera**: certification-based, virtually synchronous; higher write latency, conflicts abort transactions.
- **GTID** (`gtid_mode=ON`): global transaction ids simplify failover and auto-positioning.
- Binlog formats: prefer ROW.

**Replica lag** = time between commit on primary and apply on replica (`SHOW REPLICA STATUS` -> `Seconds_Behind_Source`, 8.0.22+ naming; older `SHOW SLAVE STATUS`/`Seconds_Behind_Master`; more reliable: heartbeat table as in pt-heartbeat). Causes: single-threaded apply (fix: `replica_parallel_workers` > 1 with `replica_parallel_type=LOGICAL_CLOCK`), huge transactions (one 5-minute DELETE = 5 minutes of lag), long DDL, replica hardware weaker, replica serving heavy reads, network.

**Read-your-writes problem:** user saves profile (primary), redirect reads from a lagging replica -> sees old data. Solutions:
1. Read from primary for N seconds after a write by that user (or for that entity) - sticky routing by session flag.
2. Carry a **GTID/LSN token**: after write, remember `@@gtid_executed`; on read choose a replica that has applied it (`WAIT_FOR_EXECUTED_GTID_SET(gtid, timeout)`); `[PG]` compare `pg_last_wal_replay_lsn()`.
3. Critical reads (balance, inventory checks before payment) always on primary.
4. Return the written data in the write response / update client cache.
Spring: `AbstractRoutingDataSource` keyed by `@Transactional(readOnly=true)`; beware that read-only routing is decided when the connection is acquired.

**Failover:** detect (orchestrator/MHA/Patroni/RDS) -> choose the replica with highest GTID set -> promote -> repoint others -> repoint apps (DNS/proxy/VIP) -> fence the old primary (STONITH) to avoid **split brain** (two primaries accepting writes). Async: some transactions exist only on the dead primary (reconcile from its binlog later). Applications must handle connection resets, idempotent retries, and possibly read-only errors (`super_read_only`).

## 11.2 Partitioning vs sharding
| | Partitioning | Sharding |
|---|---|---|
| Scope | one table split into pieces **inside one database server** | data split across **multiple servers** |
| Transparent to app | yes (same table name) | no: routing layer / shard key in the app or a proxy (Vitess, ShardingSphere) |
| Goals | maintenance (drop old partition instantly), pruning, smaller indexes | scale writes/storage beyond one machine |
| Limits | one server's CPU/IO | cross-shard joins/transactions (2PC/saga), rebalancing, ops complexity |

**Partition types (MySQL)** `RANGE`, `LIST`, `HASH`, `KEY`, plus subpartitioning.
```sql
CREATE TABLE events (
  id BIGINT NOT NULL, created_at DATE NOT NULL, payload JSON,
  PRIMARY KEY (id, created_at)                        -- partition key MUST be part of every unique key incl. PK
) PARTITION BY RANGE COLUMNS (created_at) (
  PARTITION p2025_01 VALUES LESS THAN ('2025-02-01'),
  PARTITION p2025_02 VALUES LESS THAN ('2025-03-01'),
  PARTITION pmax     VALUES LESS THAN (MAXVALUE)
);
ALTER TABLE events DROP PARTITION p2025_01;            -- instant retention purge (vs DELETE of 100M rows)
EXPLAIN SELECT * FROM events WHERE created_at >= '2025-02-10' AND created_at < '2025-02-20';   -- partitions: p2025_02  (partition pruning)
```
MySQL InnoDB restrictions: **no foreign keys** on partitioned tables, unique keys must include the partition column, pruning only when the WHERE has the partition column in a sargable form; a query without it scans **all partitions** (can be slower than unpartitioned: N index descents). Partitioning is *not* a general speed-up.
`[PG]` declarative partitioning (RANGE/LIST/HASH), partition-wise joins, `DETACH PARTITION`. `[ORA]` mature partitioning (interval, reference, local vs global indexes).

**Sharding**
- **Shard key** selection criteria: high cardinality, even distribution, present in most queries (so a query hits one shard), rarely updated, aligns with transaction boundaries (all rows of one tenant/customer on one shard).
- Strategies: **hash/mod** (even, resharding hard -> use consistent hashing or many logical shards mapped to few servers), **range** (easy range scans, hot tail for monotonic keys like time/auto-id), **directory/lookup table** (flexible, extra hop and SPOF), **geo/tenant**.
- **Hot shards**: celebrity user, one big tenant, time-based key (all writes on the latest shard). Mitigate: compound key/salting, split big tenants, separate tier for whales.
- Costs: cross-shard queries scatter-gather, distributed transactions (avoid: design so a business transaction is single-shard; else sagas/outbox), global unique ids (Snowflake/ULID/ticket server), resharding.
- Try in order: indexes/query fixes -> caching -> read replicas -> vertical scale -> partitioning -> functional split (separate services/DBs) -> sharding.

## 11.3 Connection pooling and max_connections
- A connection = server thread (MySQL, ~256 KB-several MB + per-query buffers) or process (PG, ~5-10 MB). `max_connections` default **151** in MySQL, 100 in PG. Too many active connections = context switching and lock contention; throughput *drops* past the sweet spot.
- **HikariCP sizing**: start with `connections ~ (core_count * 2) + effective_spindle_count` on the DB server side (PG wiki formula), not app-side thread count. Or **Little's law**: `pool = throughput x average connection hold time` e.g. 500 tx/s x 0.02 s = **10** connections. A pool of 10 for a service with 200 request threads is normal: excess requests queue in the pool (`connectionTimeout` 30 s) - that queue is the backpressure.
- Total across instances: `instances x maximumPoolSize <= max_connections - headroom (admin, replicas, monitoring)`. 20 pods x 20 = 400 > 151 -> `Too many connections`.
- Pool exhaustion causes: long transactions, connections held while calling external APIs, leaks (`leakDetectionThreshold`), slow queries, `@Transactional` around HTTP calls, open-in-view holding connections during view rendering.
- Set `maxLifetime` below DB/proxy `wait_timeout` (idle kill), use `validation` via JDBC4 `isValid`. External poolers: ProxySQL (MySQL), PgBouncer (PG; transaction pooling breaks session state, prepared statements, advisory locks), RDS Proxy.
- Server-side prepared statements and `useServerPrepStmts`/`cachePrepStmts` for MySQL Connector/J; batch inserts via `rewriteBatchedStatements=true`.

---

# 12. Schema design, integrity, online change, backup

## 12.1 Normalization
| Form | Rule | Violation example | Fix |
|---|---|---|---|
| **1NF** | atomic values, no repeating groups | `phones = '123,456'` | child table `phone(person_id, number)` |
| **2NF** | 1NF + no partial dependency on part of a composite key | `order_item(order_id, product_id, product_name)`: `product_name` depends on `product_id` only | move to `product` |
| **3NF** | 2NF + no transitive dependency (non-key -> non-key) | `employee(id, dept_id, dept_name)` | `department(id,name)` |
| **BCNF** | every determinant is a candidate key | `class(student, teacher, subject)` with teacher->subject | split (teacher,subject) and (student,teacher) |
Mnemonic: "every non-key attribute depends on **the key, the whole key, and nothing but the key**".

**Denormalize deliberately** for read paths: store `orders.customer_name` snapshot (also *correct* for history: price/name at purchase time), pre-computed counters, summary tables, JSON blobs for opaque documents. Cost: update anomalies, need for sync (transactions, triggers, async projections). Rule: normalize the write model, denormalize projections for reads (CQRS-lite).

## 12.2 Keys
- **Surrogate key** (`BIGINT AUTO_INCREMENT`, snowflake, UUIDv7): stable, small, no business meaning. **Natural key** (email, ISBN, country code): meaningful but can change/be reused (email changes break every FK). Best practice: surrogate PK + `UNIQUE` constraint on the natural key.
- Composite PK for pure link tables `(student_id, course_id)`: the PK index is also the join index.
- FK columns need an index (InnoDB adds one if absent); FK checks take locks on parent rows.

## 12.3 Data types that bite
| Topic | Guidance |
|---|---|
| **Money** | `DECIMAL(19,4)` or integer minor units (cents) + currency code. Never `FLOAT/DOUBLE`: `0.1+0.2 <> 0.3`. Java side `BigDecimal` (never `double`) with explicit `RoundingMode`. |
| **VARCHAR sizing** | length is the max chars, storage is actual + 1-2 B length; but temp tables/sorts (`MEMORY`, filesort) may allocate the **declared max**, and index key limit (3072 B = 768 utf8mb4 chars) - do not declare `VARCHAR(4000)` "just in case" for indexed columns. `utf8mb4` = up to 4 B/char (real UTF-8; MySQL's `utf8` = `utf8mb3`, cannot store emoji). Use `CHAR` only for fixed-length codes. |
| **Timestamps** | `DATETIME` = wall-clock, no zone, range 1000-9999. `TIMESTAMP` = UTC epoch internally, converted to session `time_zone`, range **1970-2038** (4 B; 2038 problem). Prefer storing **UTC** in `DATETIME(3)`/`TIMESTAMP(3)` and converting at the edges; `[PG]` `timestamptz` (stored UTC). Java: `Instant`/`OffsetDateTime`; avoid `java.util.Date` + server default zones. Fractional seconds: `DATETIME(3)`. DST: never store local time without offset for events. |
| **ENUM** | stored as 1-2 B index, fast, but **adding a value requires ALTER** (instant only if appended at end), order = definition order (sorting surprises), invalid value -> '' in non-strict mode, portability poor. Prefer lookup table + FK, or `VARCHAR` + `CHECK` (MySQL 8.0.16+ enforces CHECK). |
| **BOOLEAN** | `TINYINT(1)` in MySQL (alias `BOOLEAN`); `[PG]` real boolean; `[ORA]` no boolean column (use `NUMBER(1)` or `CHAR(1)`). |
| **JSON columns** | MySQL `JSON` (binary storage, validated), PG `jsonb` (indexable with GIN). Good for sparse/variable attributes, event payloads. Bad for anything you filter/join/aggregate on constantly: no per-attribute stats, big documents update whole value (MySQL partial JSON update exists for `JSON_SET/REPLACE/REMOVE` in some cases). Index via **generated column**: `ALTER TABLE t ADD c_city VARCHAR(50) AS (payload->>'$.city') VIRTUAL, ADD INDEX(c_city);` or a functional index; multi-valued indexes (8.0.17) for arrays. |
| **TEXT/BLOB** | off-page storage; avoid in hot tables; `SELECT *` drags them. Store files in object storage, keep URL. |
| **Integers** | `INT` max 2,147,483,647 - auto-increment overflow on busy tables is a classic outage; use `BIGINT` (or `INT UNSIGNED` = 4.29 B) for event/log tables. Display width `INT(11)` is meaningless in 8.0. |
| **Character collation** | `utf8mb4_0900_ai_ci` (8.0 default, accent- and case-insensitive) vs `utf8mb4_bin` (exact). Affects uniqueness ('a' = 'A' under _ci!) and join index use. |

## 12.4 Constraints and integrity
- `PRIMARY KEY`, `UNIQUE` (also an index), `NOT NULL`, `FOREIGN KEY` (`ON DELETE CASCADE / SET NULL / RESTRICT`), `CHECK` (MySQL 8.0.16+, before that parsed and ignored!), `DEFAULT`.
- Let the database enforce invariants that must never break (uniqueness, referential integrity): application checks race (check-then-insert = duplicate under concurrency; the unique index is the only real guard). Catch `DuplicateKeyException`/`DataIntegrityViolationException`.
- FKs: cost on write (lookups/locks) and on bulk load, complicate sharding/online DDL; many high-scale shops drop FKs and rely on application + periodic orphan audits; for a normal 5-year-experience service keep them.
- Soft delete (`deleted_at`) interacts with UNIQUE (need `UNIQUE(email, deleted_at)` tricks or partial index in PG) and with every query needing the filter - consider archive tables.

## 12.5 Online schema change and backward-compatible migrations
**Problem:** `ALTER TABLE` on a 500M-row table. Options in MySQL 8:
| Method | How | Notes |
|---|---|---|
| `ALGORITHM=INSTANT` | metadata-only change (8.0.12+: add column at end, 8.0.29+: any position; drop column 8.0.29+; rename column, change default) | milliseconds; check with `ALTER TABLE ... , ALGORITHM=INSTANT` - errors if unsupported instead of silently copying |
| `ALGORITHM=INPLACE, LOCK=NONE` | rebuild/alter in place while allowing concurrent DML (add index, change nullability...) | writes logged in a row log and applied at the end; still needs short exclusive **metadata lock** at start and end |
| `ALGORITHM=COPY` | create new table, copy, swap | blocks writes, needs 2x disk |
| **pt-online-schema-change** (Percona) | creates shadow table, copies in chunks, **triggers** keep it in sync, atomic rename | triggers add overhead; problems with existing triggers/FKs |
| **gh-ost** (GitHub) | shadow table filled by chunk copy + applying changes read from **binlog** (triggerless), can pause/throttle, test on replica | preferred for busy primaries |
Always specify: `ALTER TABLE t ADD COLUMN c INT NULL, ALGORITHM=INSTANT, LOCK=NONE;`
`[PG]` `ADD COLUMN ... DEFAULT const` is instant since PG 11; `CREATE INDEX CONCURRENTLY`; add FK/CHECK as `NOT VALID` then `VALIDATE CONSTRAINT`; `ALTER COLUMN TYPE` rewrites the table. `[ORA]` online DDL (`ONLINE`), `DBMS_REDEFINITION`.

**Metadata-lock hazard:** an `ALTER` waits for an MDL while an old transaction holds one; while it waits, **all new queries on that table queue behind the ALTER** -> outage. Mitigate: `SET SESSION lock_wait_timeout=5` before DDL, run when no long transactions (`information_schema.INNODB_TRX`), retry.

**Expand/contract (parallel change) for zero-downtime deploys** - each step is a separate deploy and is backward compatible with the previous app version:
```
1. EXPAND    add new nullable column/table/index (instant/online).   Old app ignores it.
2. DUAL-WRITE deploy app writing both old and new columns; reads still old.
3. BACKFILL  copy existing data in small chunks (id ranges, sleep between, monitor lag).
4. VERIFY    compare old vs new (checksums/counts), switch reads to new behind a flag.
5. CONTRACT  stop writing old, deploy, later DROP old column/table (only after rollback window).
```
Rename a column = add new + backfill + switch + drop old (never a direct `RENAME` while two app versions run). Never add `NOT NULL` without default in one step on big tables. Migration tools: Flyway/Liquibase; keep migrations idempotent and separated from application boot on multi-instance deployments (lock table protects, but long migrations block startup).

## 12.6 Backup, PITR
- **Logical**: `mysqldump --single-transaction` (consistent InnoDB snapshot without locking; slow to restore), `mydumper`. **Physical**: Percona XtraBackup / MySQL Enterprise Backup / LVM/EBS snapshots (fast restore), `[PG]` `pg_basebackup` + WAL archiving, `[ORA]` RMAN.
- **Point-in-time recovery** = restore last full backup + **replay binlog** up to just before the disaster: `mysqlbinlog --start-datetime=... --stop-datetime=... binlog.000123 | mysql` (or by GTID/position). Requires binlog retained (`binlog_expire_logs_seconds`) and shipped off-host. `[PG]` archive WAL + `recovery_target_time`.
- RPO (data loss tolerated) = backup interval + binlog shipping lag; RTO = restore time. **A backup not test-restored is not a backup.** Replicas are not backups (a `DROP TABLE` replicates). Delayed replica (`CHANGE REPLICATION SOURCE TO SOURCE_DELAY=3600`) gives a cheap safety net.

---

# 13. Observability

## 13.1 Slow query log
```sql
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;                    -- seconds; 0 = log everything (short bursts only)
SET GLOBAL log_queries_not_using_indexes = ON;       -- noisy; use with log_throttle_queries_not_using_indexes
SET GLOBAL log_slow_extra = ON;                      -- 8.0.14+: extra fields
```
Entry key fields: `Query_time`, `Lock_time`, `Rows_sent`, `Rows_examined`. **Rows_examined / Rows_sent >> 1** = the index is not selective enough / missing. Aggregate with `pt-query-digest` (groups by fingerprint, ranks by total time - optimise total time, not the single slowest query), or `mysqldumpslow`.

## 13.2 performance_schema and sys
```sql
-- top statements by total time (digest = normalized query)
SELECT DIGEST_TEXT, COUNT_STAR, ROUND(SUM_TIMER_WAIT/1e12,1) total_s, ROUND(AVG_TIMER_WAIT/1e9,2) avg_ms,
       SUM_ROWS_EXAMINED, SUM_ROWS_SENT, SUM_NO_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
-- friendlier:
SELECT * FROM sys.statement_analysis LIMIT 10;               -- same, formatted
SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;
SELECT * FROM sys.statements_with_temp_tables LIMIT 10;
SELECT * FROM sys.schema_unused_indexes;   SELECT * FROM sys.schema_redundant_indexes;
SELECT * FROM sys.innodb_lock_waits;       SELECT * FROM sys.processlist;
SELECT * FROM sys.io_global_by_file_by_bytes LIMIT 10;
```
Also: `SHOW PROCESSLIST` / `information_schema.PROCESSLIST` (state, time), `SHOW GLOBAL STATUS` (`Threads_running`, `Innodb_buffer_pool_reads` vs `_read_requests` -> hit ratio, `Created_tmp_disk_tables`, `Handler_read_rnd_next` high = table scans), `SHOW ENGINE INNODB STATUS`. `[PG]` `pg_stat_statements`, `pg_stat_activity`, `auto_explain`, `pg_locks`. `[ORA]` AWR/ASH, `V$SQL`.

## 13.3 Systematic slow-query workflow
```
symptom (latency p99 up / CPU up / pool exhausted)
 -> Is it the DB? (APM span, Threads_running, CPU, IOPS, lock waits) 
 -> top digests by total time (perf schema / pt-query-digest)
 -> for the top offender: EXPLAIN ANALYZE, compare estimated vs actual rows
 -> classify: missing/wrong index | non-sargable | too many rows returned | N+1 | lock wait | stats | big sort/temp | plan flip
 -> fix smallest-blast-radius first (index via INVISIBLE rehearsal / query rewrite / cache / pagination)
 -> verify in staging with production-size data, deploy, watch digest metrics before/after
```

---

# 14. Caching, OLTP vs OLAP, NoSQL

## 14.1 Cache (Redis) interplay
- **Cache-aside** (most common): read cache -> miss -> read DB -> populate with TTL. Write: update DB, then **delete** the cache key (delete rather than update avoids race where two writers set cache out of order).
- Failure modes: **stampede** (hot key expires, 1000 requests hit the DB: use single-flight/lock, jittered TTL, early refresh), **penetration** (non-existent keys: cache negative results, Bloom filter), **avalanche** (many keys expire together), **stale reads** after write (short TTL, event-driven invalidation via binlog/Debezium), **dual-write inconsistency** (DB commits, cache delete fails: use retry/outbox, TTL as backstop).
- The DB remains source of truth; cache is an optimisation you must be able to lose. Do not cache what you can fix with an index first.
- Write-through / write-behind exist but move durability risk into the cache.
- Replica lag and cache interact: a stale replica read can **re-populate the cache with old data** after invalidation. Populate from primary or version-stamp values.

## 14.2 OLTP vs OLAP
| | OLTP | OLAP |
|---|---|---|
| Workload | many short transactions, point reads/writes | few big scans/aggregations over history |
| Model | normalized, row store, B+ tree | star/snowflake schema (facts + dimensions), **column store** (ClickHouse, Redshift, BigQuery, Snowflake) |
| Optimises | latency, concurrency, ACID | throughput, compression, vectorized scans |
| Index | many selective B-trees | few/none; partition pruning, zone maps |
Run analytics on a replica or a warehouse fed by CDC/ETL, not on the primary. Heavy reports on OLTP primary = long snapshots (undo bloat), buffer pool churn, lock waits.

## 14.3 NoSQL decision guide with CAP notes
**CAP:** under a network **P**artition a system must choose **C**onsistency (refuse/err some requests) or **A**vailability (answer, maybe stale). Without a partition you trade latency vs consistency (PACELC). Single-node MySQL is "CA" trivially; replicated systems choose per configuration (semi-sync = leaning C, async replica reads = leaning A).
| Need | Pick | Why / caution |
|---|---|---|
| Relational, transactions, ad-hoc queries, reporting | **MySQL/PostgreSQL** | default choice; scales far (10 TB, 10k+ QPS with replicas/caching) |
| Sub-ms key lookups, counters, sessions, rate limits, leaderboards, locks | **Redis** | memory-bound, persistence options (RDB/AOF) weaker than an RDBMS |
| Flexible documents, per-document atomicity, evolving schema | **MongoDB** | multi-document transactions exist but cost; model for query patterns, embed vs reference |
| Massive write throughput, time series/event logs, predictable key-based queries, multi-DC, tunable consistency | **Cassandra/ScyllaDB/DynamoDB** | query-first modelling, no joins, eventual consistency by default (quorum reads/writes give `R+W>N` strong-ish), denormalize per query |
| Full-text search, relevance, log analytics | **Elasticsearch/OpenSearch** | search index, not a source of truth |
| Graph traversals (fraud rings, recommendations) | **Neo4j** | when recursive joins get deep |
| Analytics over billions of rows | **ClickHouse/BigQuery/Redshift** | columnar OLAP |
| Blob/file | Object storage (S3) | store keys in DB |
Decision heuristics: start relational; add a specialised store when a *measured* requirement (scale of writes, latency, data shape, search) cannot be met; each extra store = new consistency, backup, ops, and sync problems. "NoSQL scales, SQL does not" is a myth: the real axis is *what queries and consistency you need*.
BASE (basically available, soft state, eventually consistent) vs ACID. Eventual consistency implications: read-your-writes, monotonic reads, conflict resolution (LWW, vector clocks, CRDTs).

---

# 15. Production war stories

## 15.1 Slow query -> EXPLAIN evidence -> fix
**Symptom:** order list page p99 = 8 s after a data growth to 50M orders. DB CPU 90%.
1. Top digest (`sys.statement_analysis`): `SELECT * FROM orders WHERE customer_id = ? AND status = ? ORDER BY created_at DESC LIMIT 20` avg 1.4 s, `rows_examined` avg 380,000, `rows_sent` 20.
2. `EXPLAIN` [ILLUSTRATIVE]:
```
type=ref key=idx_customer_id key_len=8 rows=381000 filtered=10.00 Extra=Using where; Using filesort
```
3. Diagnosis: whale customer has 380k orders; index `(customer_id)` finds them all, filters `status`, sorts them (filesort) then takes 20.
4. Fix: `CREATE INDEX idx_cust_status_created ON orders (customer_id, status, created_at)` (online, `ALGORITHM=INPLACE, LOCK=NONE`), `DROP INDEX idx_customer_id` (redundant prefix) after INVISIBLE rehearsal.
5. After: `type=ref key=idx_cust_status_created rows=20 Extra=Backward index scan`, `rows_examined` 20, avg 0.6 ms. Lesson: **rows_examined vs rows_sent** points straight at the missing index; equality columns then sort column.

## 15.2 Deadlock storm after a deploy
**Symptom:** error 1213 spikes to 200/min after a release that added "update inventory for all items in an order" in a loop.
- `SHOW ENGINE INNODB STATUS` showed both transactions updating rows of `inventory` (`id` 7,9 vs 9,7): each order looped over items in the order the customer added them.
- Fix: sort items by `inventory.id` before the loop (consistent lock order); single statement `UPDATE ... WHERE id IN (...)` does not guarantee order, so keep `ORDER BY id` in the earlier `SELECT ... FOR UPDATE`; add retry (3 attempts with jitter) around the transaction; reduced transaction span by moving payment call outside the DB transaction. Deadlocks -> ~0/day, and a retry metric to alert on regressions.

## 15.3 Replica lag and stale reads
**Symptom:** "I paid but it shows unpaid" for ~1 min every night at 02:00.
- Evidence: `Seconds_Behind_Source` rises to 70 s at 02:00; a cleanup job ran `DELETE FROM audit WHERE created < NOW() - INTERVAL 90 DAY` (12M rows, one transaction). Replica applies that single transaction for ~70 s in one applier thread, blocking everything behind it.
- Fixes: delete in chunks of 5,000 with sleep + `replica_parallel_workers=8` (`LOGICAL_CLOCK`) + partition `audit` by month and `DROP PARTITION`; application: read-your-writes routing (payment status read from primary for 30 s after payment event).

## 15.4 The ALTER that took the site down
**Symptom:** `ALTER TABLE orders ADD COLUMN gift TINYINT` started 14:00 (expected instant); by 14:02 every query on `orders` hung, pool exhausted, 5xx.
- Cause: a BI analyst's session had `START TRANSACTION; SELECT ... FROM orders` open for 40 min (idle) -> holds a shared **MDL**. The ALTER requested an exclusive MDL and waited; **all later queries queued behind the pending ALTER's MDL request**.
- Evidence: `SHOW PROCESSLIST` state `Waiting for table metadata lock` on dozens of threads; `sys.schema_table_lock_waits` / `performance_schema.metadata_locks` showed the blocker thread. `KILL <blocker id>` -> ALTER proceeded (INSTANT, 10 ms), queue drained.
- Prevention: `SET lock_wait_timeout=5` on the DDL session + retry loop; scan `INNODB_TRX` for old transactions before DDL; kill idle transactions via `wait_timeout` (MySQL; PG has `idle_in_transaction_session_timeout`); use gh-ost; run DDL in low traffic.

## 15.5 Auto-increment overflow
`INT` PK on an events table hits 2,147,483,647 -> `ERROR 1062 Duplicate entry '2147483647'` for all inserts. Emergency `ALTER ... MODIFY id BIGINT` takes hours (COPY rebuild of a 2 TB table) -> gh-ost migration. Prevention: `BIGINT` from day one, alert at 70% of the max (`information_schema.TABLES.AUTO_INCREMENT`).

## 15.6 "Missing index" that was a type mismatch
`WHERE user_ref = 12345` on `user_ref VARCHAR(20)` with a perfectly good index: EXPLAIN `type=ALL`. MySQL cast every row to number. Fix: bind the parameter as a String (`setString`), not a `long`. Java angle: a Hibernate entity field typed `Long` while the column is `VARCHAR` caused it.

---

# 16. Forty classic SQL interview queries (all run on SQLite 3.45 [VALIDATED]; MySQL 8 notes where dialect differs)

**Sample data used throughout**
```
emp(id, name, dept, salary, manager_id)
 1 Asha   ENG 300 NULL      6 Farid  HR  90  5
 2 Bala   ENG 200 1         7 Gita   HR  90  5
 3 Chitra ENG 200 1         8 Hari   OPS 110 1
 4 Dev    ENG 150 2         9 Indu   OPS 130 1
 5 Esha   HR  120 1        10 Jai    OPS 80  9
```
Other tables are described inline. Portability legend: works in MySQL 8 / PG / Oracle unless noted. (In MySQL 5.7 there are no window functions or CTEs.)

## A. Ranking and top-N
**Q1. Second highest salary**
```sql
SELECT MAX(salary) FROM emp WHERE salary < (SELECT MAX(salary) FROM emp);          -- 200
```
Alternatives: `ORDER BY salary DESC LIMIT 1 OFFSET 1` (**wrong with duplicates** at the top: 300,300,200 would return 300; use `SELECT DISTINCT salary`), returns no row if absent (wrap in `SELECT (...) AS second` to get NULL).

**Q2. Nth highest salary (N=2), ties handled**
```sql
SELECT DISTINCT salary FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) rk FROM emp) t WHERE rk = 2;   -- 200
```
Use `DENSE_RANK` (distinct values), not `RANK` (gaps) or `ROW_NUMBER` (ignores ties).

**Q3. Top 2 earners per department**
```sql
SELECT dept, name, salary, rn FROM (
  SELECT dept, name, salary, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC, id) rn FROM emp) t
WHERE rn <= 2 ORDER BY dept, rn;
```
```
ENG Asha 300 1 | ENG Bala 200 2 | HR Esha 120 1 | HR Farid 90 2 | OPS Indu 130 1 | OPS Hari 110 2
```
Use `RANK`/`DENSE_RANK` instead of `ROW_NUMBER` if ties must all be included. Add a tie-breaker (`id`) for determinism.

**Q4. ROW_NUMBER vs RANK vs DENSE_RANK**: see 7.5 (300,200,200,150 -> rn 1,2,3,4; rank 1,2,2,4; dense 1,2,2,3).

**Q5. Highest-paid employee(s) per department (ties included)**
```sql
SELECT dept, name FROM (SELECT *, RANK() OVER (PARTITION BY dept ORDER BY salary DESC) r FROM emp) t WHERE r = 1;   -- ENG Asha, HR Esha, OPS Indu
-- pre-window era: SELECT * FROM emp e WHERE salary = (SELECT MAX(salary) FROM emp WHERE dept = e.dept);
```
**Q6. Second highest salary per department**
```sql
SELECT dept, salary FROM (SELECT dept, salary, DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) r FROM emp) t
WHERE r = 2 GROUP BY dept, salary;            -- ENG 200, HR 90, OPS 110
```
**Q7. Department with highest total salary**
```sql
SELECT dept, SUM(salary) t FROM emp GROUP BY dept ORDER BY t DESC LIMIT 1;    -- ENG 850   ([ORA] FETCH FIRST 1 ROW ONLY; ties: use RANK())
```
**Q8. Employees paid above their department average**
```sql
SELECT name, dept, salary FROM (SELECT *, AVG(salary) OVER (PARTITION BY dept) a FROM emp) t WHERE salary > a;
-- Asha ENG 300 | Esha HR 120 | Hari OPS 110 | Indu OPS 130
```
**Q9. Departments with at least 3 employees**
```sql
SELECT dept, COUNT(*) n, AVG(salary) a FROM emp GROUP BY dept HAVING COUNT(*) >= 3;   -- ENG 4 212.5 | HR 3 100.0 | OPS 3 106.67
```
## B. Hierarchies and self joins
**Q10. Employees earning more than their manager**
```sql
SELECT e.name, e.salary, m.name AS mgr, m.salary AS mgr_salary
FROM emp e JOIN emp m ON e.manager_id = m.id WHERE e.salary > m.salary;      -- (0 rows on this data set; syntax validated)
```
**Q11. Employees with no direct reports (leaf nodes)**
```sql
SELECT name FROM emp e WHERE NOT EXISTS (SELECT 1 FROM emp x WHERE x.manager_id = e.id);   -- Chitra, Dev, Farid, Gita, Hari, Jai
```
**Q12. Number of direct reports per manager**
```sql
SELECT m.name, COUNT(e.id) reports FROM emp m LEFT JOIN emp e ON e.manager_id = m.id
GROUP BY m.id, m.name HAVING COUNT(e.id) > 0 ORDER BY reports DESC;      -- Asha 5, Esha 2, Bala 1, Indu 1
```
**Q13. Full org hierarchy with level and path (recursive CTE)** - see 7.6 (validated 10-row output). To count *all* (transitive) reports: recursive CTE from each manager, or join the CTE output on path prefix.

## C. Duplicates
Data `users(id,email)`: (1,a),(2,b),(3,a),(4,c),(5,b),(6,a).
**Q14. Find duplicate emails**
```sql
SELECT email, COUNT(*) c FROM users GROUP BY email HAVING COUNT(*) > 1;      -- a 3, b 2
```
**Q15. Delete duplicates, keep the lowest id**
```sql
DELETE FROM users WHERE id NOT IN (SELECT MIN(id) FROM users GROUP BY email);   -- keeps 1,2,4 (validated)
```
**MySQL gotcha:** the statement above is validated in SQLite, but MySQL raises error 1093 "You can't specify target table 'users' for update in FROM clause" for it; wrap in a derived table `NOT IN (SELECT id FROM (SELECT MIN(id) id FROM users GROUP BY email) x)`, or use a multi-table delete:
```sql
DELETE u FROM users u JOIN users k ON u.email = k.email AND u.id > k.id;        -- MySQL only syntax
-- window version (PG/MySQL 8): find ids to delete first
SELECT id FROM (SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) rn FROM users) t WHERE rn > 1;   -- 3, 6, 5 (validated)
```
`[PG]` `DELETE FROM users u USING users k WHERE u.email=k.email AND u.id>k.id;`. On a big table do it in batches and add a `UNIQUE(email)` afterwards so it cannot recur.

## D. Running, moving, comparison
Data `pay(id, amount)`: (1,100),(2,50),(3,70),(4,30).
**Q16. Running total**
```sql
SELECT id, amount, SUM(amount) OVER (ORDER BY id) run FROM pay;        -- 100, 150, 220, 250
```
**Q17. Moving average of current + previous row**
```sql
SELECT id, amount, AVG(amount) OVER (ORDER BY id ROWS BETWEEN 1 PRECEDING AND CURRENT ROW) ma FROM pay;   -- 100, 75, 60, 50
```
**Q18. Difference from previous row (LAG)** - see 7.5 (`NULL, -50, 20, -40`).
Data `px(d,p)`: (1,7),(2,1),(3,5),(4,3),(5,6),(6,4).
**Q19. Rolling 3-row sum**
```sql
SELECT d, p, SUM(p) OVER (ORDER BY d ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) r3 FROM px;   -- 7, 8, 13, 9, 14, 13
```
Data `yr(y,rev)`: 2021:100, 2022:120, 2023:90, 2024:150.
**Q20. Year-over-year growth %**
```sql
SELECT y, rev, rev - LAG(rev) OVER (ORDER BY y) diff,
       ROUND(100.0 * (rev - LAG(rev) OVER (ORDER BY y)) / LAG(rev) OVER (ORDER BY y), 1) pct FROM yr;
-- 2022: +20 (20.0%), 2023: -30 (-25.0%), 2024: +60 (66.7%)
```
**Q21. Month-over-month revenue** (`ord(id,cid,total,st,d)`: (1,1,10,PAID,01-01),(2,1,20,NEW,01-01),(3,2,5,PAID,01-02),(4,3,7,PAID,02-01),(5,1,40,PAID,02-03))
```sql
SELECT m, rev, rev - LAG(rev) OVER (ORDER BY m) d FROM (
  SELECT SUBSTR(d,1,7) m, SUM(total) rev FROM ord WHERE st='PAID' GROUP BY 1) t;      -- 2024-01: 15 | 2024-02: 47 (+32)
-- MySQL: DATE_FORMAT(d,'%Y-%m'); PG: to_char(d,'YYYY-MM') or date_trunc('month', d)
```
**Q22. Max profit from one buy then one sell** (prices above)
```sql
SELECT MAX(p - mn) profit FROM (SELECT p, MIN(p) OVER (ORDER BY d ROWS UNBOUNDED PRECEDING) mn FROM px) t;   -- 5 (buy 1 on d2, sell 6 on d5)
```
## E. Gaps, islands, streaks
Data `ids(id)`: 1,2,3,4,7,8,10,11,12,15.
**Q23. Gaps in a sequence**
```sql
SELECT id + 1 AS gap_start, nxt - 1 AS gap_end FROM (SELECT id, LEAD(id) OVER (ORDER BY id) nxt FROM ids) t WHERE nxt - id > 1;
-- 5-6 | 9-9 | 13-14
```
**Q24. Islands (consecutive runs) - the "row number difference" trick**
```sql
SELECT MIN(id) a, MAX(id) b FROM (SELECT id, id - ROW_NUMBER() OVER (ORDER BY id) g FROM ids) t GROUP BY g;
-- 1-4 | 7-8 | 10-12 | 15-15
```
Within an island `id - row_number` is constant, hence the group key. For dates use `DATE_SUB(d, INTERVAL rn DAY)` (MySQL) / `d - rn` (PG date) / SQLite `julianday(d) - rn`.
Data `logins(uid,d)`: uid1: 01-01,01-02,01-03,01-05,01-06; uid2: 01-01,01-03,01-04,01-04 (duplicate).
**Q25. Users with a login streak of >= 3 consecutive days**
```sql
SELECT uid, MIN(d) s, MAX(d) e, COUNT(*) n FROM (
  SELECT uid, d, julianday(d) - ROW_NUMBER() OVER (PARTITION BY uid ORDER BY d) g       -- MySQL: DATE_SUB(d, INTERVAL ROW_NUMBER() OVER (...) DAY)
  FROM (SELECT DISTINCT uid, d FROM logins) x) t                                        -- DISTINCT first: duplicates break the trick
GROUP BY uid, g HAVING COUNT(*) >= 3;                                                    -- 1 | 2024-01-01 | 2024-01-03 | 3
```
Variation, **longest streak per user**: wrap the grouped result and `MAX(n)` per uid. Alternative without windows: self-join on `d + 1`, three times (fragile).
**Q26. Missing numbers in 1..max via recursive CTE** (`nums`: 1,2,4,5,8)
```sql
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM s WHERE n < (SELECT MAX(n) FROM nums))
SELECT n FROM s WHERE n NOT IN (SELECT n FROM nums);      -- 3, 6, 7    (use NOT EXISTS if nums.n can be NULL)
```
## F. Reshaping and statistics
Data `sales(yr,q,amt)`: 2023: 10,20,30,40; 2024: 15,25,35,45.
**Q27. Pivot rows to columns with CASE (portable)**
```sql
SELECT yr, SUM(CASE WHEN q=1 THEN amt END) q1, SUM(CASE WHEN q=2 THEN amt END) q2,
           SUM(CASE WHEN q=3 THEN amt END) q3, SUM(CASE WHEN q=4 THEN amt END) q4 FROM sales GROUP BY yr;
-- 2023: 10 20 30 40 | 2024: 15 25 35 45
```
Conditional counts per status per day: `SUM(CASE WHEN st='PAID' THEN 1 ELSE 0 END)`; `[PG]` `COUNT(*) FILTER (WHERE st='PAID')`; `[ORA]` `PIVOT` clause. Dynamic column lists need dynamic SQL. Unpivot: `UNION ALL` of selects or `CROSS JOIN LATERAL/VALUES`.
**Q28. Median salary** (10 employees -> average of 5th and 6th)
```sql
SELECT AVG(salary) med FROM (SELECT salary, ROW_NUMBER() OVER (ORDER BY salary) rn, COUNT(*) OVER () n FROM emp) t
WHERE rn IN ((n + 1) / 2, (n + 2) / 2);                    -- 125.0   (integer division: even n -> two middle rows; odd n -> same row twice)
```
`[PG]`/`[ORA]` `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)`; MySQL has no built-in median. Percentile per group: add `PARTITION BY`.
**Q29. Cumulative percentage of total (Pareto)**
```sql
SELECT name, salary, ROUND(100.0 * SUM(salary) OVER (ORDER BY salary DESC, id) / SUM(salary) OVER (), 1) cum FROM emp;
-- Asha 20.4, Bala 34.0, Chitra 47.6, Dev 57.8, Indu 66.7, Esha 74.8, Hari 82.3, Farid 88.4, Gita 94.6, Jai 100.0
```
**Q30. Revenue share per customer** (paid+new orders above, NULL customer excluded)
```sql
SELECT cid, SUM(total) t, ROUND(100.0 * SUM(total) / SUM(SUM(total)) OVER (), 1) pct FROM ord WHERE cid IS NOT NULL GROUP BY cid;
-- (window over aggregate: SUM(SUM(total)) OVER ())
```
## G. Sessions, retention, cohorts
Data `act(uid, ts)`: uid1 at 10:00, 10:20, 12:00, 12:10 on 2024-01-01; uid2 at 09:00.
**Q31. Sessionization: new session when gap > 30 minutes**
```sql
SELECT uid, ts, SUM(new_s) OVER (PARTITION BY uid ORDER BY ts) sess FROM (
  SELECT uid, ts, CASE WHEN LAG(ts) OVER (PARTITION BY uid ORDER BY ts) IS NULL
                         OR (julianday(ts) - julianday(LAG(ts) OVER (PARTITION BY uid ORDER BY ts))) * 1440 > 30
                       THEN 1 ELSE 0 END new_s FROM act) t;
-- uid1: 10:00->1, 10:20->1, 12:00->2, 12:10->2 ; uid2: 09:00->1
-- MySQL: TIMESTAMPDIFF(MINUTE, LAG(ts) OVER (...), ts) > 30 ;  PG: ts - LAG(ts) OVER (...) > INTERVAL '30 minutes'
```
Pattern: flag boundaries with LAG, then running SUM of flags = session id. Then `GROUP BY uid, sess` for session length/count.
Data `signup(uid,sd)`: (1,01-01),(2,01-01),(3,01-02); `ev(uid,d)`: 1:01-01,01-02; 2:01-01,01-03; 3:01-02,01-03.
**Q32. Day-1 retention by signup cohort**
```sql
SELECT s.sd, COUNT(DISTINCT s.uid) cohort, COUNT(DISTINCT e.uid) d1,
       ROUND(100.0 * COUNT(DISTINCT e.uid) / COUNT(DISTINCT s.uid), 1) pct
FROM signup s LEFT JOIN ev e ON e.uid = s.uid AND e.d = date(s.sd, '+1 day')     -- MySQL: DATE_ADD(s.sd, INTERVAL 1 DAY)
GROUP BY s.sd;                                                                    -- 01-01: 2,1,50.0 | 01-02: 1,1,100.0
```
Generalize: replace `+1 day` with N days, or compute `DATEDIFF(e.d, s.sd)` and pivot day 0/1/7/30. Put the day filter in `ON` (LEFT JOIN semantics).

## H. Anti-joins, set logic, misc
**Q33. Customers who never ordered**
```sql
SELECT c.* FROM cust c LEFT JOIN ord o ON o.cid = c.id WHERE o.id IS NULL;      -- C   (cust A,B,C; orders reference customers 1,1,2 and one NULL: validated in 7.1 data)
SELECT c.* FROM cust c WHERE NOT EXISTS (SELECT 1 FROM ord o WHERE o.cid = c.id);
```
**Q34. The NOT IN + NULL trap** - see 7.1 (0 vs 1).
Data `purch(uid,prod)`: (1,A),(1,B),(2,A),(3,B),(3,B),(4,A),(4,B),(4,C).
**Q35. Users who bought both A and B**
```sql
SELECT uid FROM purch WHERE prod IN ('A','B') GROUP BY uid HAVING COUNT(DISTINCT prod) = 2;    -- 1, 4   (user 3 bought B twice: COUNT(*) would wrongly include)
```
**Q36. Latest row per key**
```sql
SELECT * FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY cid ORDER BY d DESC, id DESC) rn FROM ord WHERE cid IS NOT NULL) t WHERE rn = 1;
-- customer 1 -> id 5 (02-03), 2 -> id 3, 3 -> id 4
```
Pre-window alternative: join to `(SELECT cid, MAX(d) FROM ord GROUP BY cid)` (breaks on ties). `[PG]` `SELECT DISTINCT ON (cid) * FROM ord ORDER BY cid, d DESC`.
**Q37. First order per customer**: same as Q36 with `ORDER BY d, id` (validated: id 1, 3, 4).
Data `seat(id, student)`: 1 A, 2 B, 3 C, 4 D, 5 E.
**Q38. Swap every two adjacent seats' students (leave last odd one)**
```sql
SELECT CASE WHEN id % 2 = 1 AND id = (SELECT MAX(id) FROM seat) THEN id
            WHEN id % 2 = 1 THEN id + 1 ELSE id - 1 END AS id, student
FROM seat ORDER BY 1;                                      -- 1 B, 2 A, 3 D, 4 C, 5 E
```
**Q39. Salary bands**
```sql
SELECT CASE WHEN salary < 100 THEN '<100' WHEN salary < 200 THEN '100-199' ELSE '200+' END band, COUNT(*) n
FROM emp GROUP BY 1 ORDER BY 1;                            -- 100-199: 4, 200+: 3, <100: 3   (GROUP BY ordinal: MySQL/PG ok, [ORA] repeat the expression)
```
**Q40. Employees earning the same salary (group them)**
```sql
SELECT salary, GROUP_CONCAT(name) names FROM emp GROUP BY salary HAVING COUNT(*) > 1;   -- 90 Farid,Gita | 200 Bala,Chitra
-- PG: string_agg(name, ','); Oracle: LISTAGG(name, ',') WITHIN GROUP (ORDER BY name)
```
**Bonus one-liners**
- Swap values: `UPDATE t SET sex = CASE sex WHEN 'M' THEN 'F' ELSE 'M' END;`
- Top-N overall with ties: `SELECT * FROM emp WHERE salary >= (SELECT DISTINCT salary FROM emp ORDER BY salary DESC LIMIT 1 OFFSET 2)`.
- Delete rows older than N in batches: `DELETE FROM logs WHERE created < ? ORDER BY id LIMIT 5000;` loop until 0 affected.
- Users inactive 30 days: `GROUP BY uid HAVING MAX(ts) < :asof - INTERVAL 30 DAY` (validated variant returns user 2).
- Employees earning above the company average: `WHERE salary > (SELECT AVG(salary) FROM emp)` (4 rows).

---

# 17. Interview questions (E = Easy, M = Medium, H = Hard)
Format: **Q** - answer. *Follow-ups* chain deeper. Section 17.6 lists the wrong answers interviewers hear most.

## 17.1 Indexes and storage
**Q1 (E) What is an index and what does it cost?** Sorted structure (B+ tree) giving O(log n) seek instead of a scan; costs storage, slower INSERT/UPDATE/DELETE (each index maintained), buffer pool pressure. *F: Why not index everything? -> write amplification + optimizer noise. F: How many levels for 100M rows? -> 4 (fan-out ~1000, section 2.3).*
**Q2 (E) Clustered vs non-clustered index?** InnoDB: PK B+ tree leaves *are* the rows; secondary leaves hold (key, PK). One clustered index per table. *F: Table without PK? -> first unique not-null index, else hidden 6-byte row id. F: PG? -> heap + all indexes secondary, `CLUSTER` is one-shot.*
**Q3 (M) Why do secondary index lookups need a second lookup and how to avoid it?** Leaf stores PK not row address; fetch the row by PK (bookmark lookup). Avoid with a covering index (`Using index`). *F: Cost when 30% of rows match? -> random PK lookups exceed a sequential scan, optimizer scans. F: Does `SELECT id` from a secondary index need a lookup? -> no, PK is in the entry.*
**Q4 (M) Explain the leftmost-prefix rule with `(a,b,c)`.** Sorted by a, then b, then c; usable for `a`, `a,b`, `a,b,c` (and `a,c` only for `a`); not for `b` or `c` alone. Columns after the first range column cannot narrow the seek. *F: `WHERE a=1 AND b>5 AND c=3`? -> seek on a,b range; c filtered (ICP). F: skip scan? -> 8.0.13+, few distinct leading values only.*
**Q5 (M) How do you choose column order in a composite index?** Equality columns first, then one range column, then ORDER BY/covering columns; consider which prefixes other queries reuse; selectivity matters for the combination. *F: Why not simply most selective first? -> `(id_like, status)` serves fewer query shapes; and range/sort position rule beats raw selectivity.*
**Q6 (M) When is `Using filesort` avoided?** When the index order after the equality prefix matches ORDER BY (same direction or fully reversed; 8.0 supports mixed direction indexes). `IN (...)` on the prefix column breaks global order. *F: Does filesort mean disk? -> no, explicit sort; disk only when it exceeds `sort_buffer_size`.*
**Q7 (M) UUID vs auto-increment PK?** UUIDv4 inserts randomly -> page splits, low fill factor, cache-unfriendly, and the wide PK is copied to every secondary index. Auto-increment appends. Use `BIGINT` or time-ordered ids (UUIDv7/ULID, `BINARY(16)`). *F: Downsides of auto-increment? -> enumerable, gaps, cross-node coordination, single insertion hotspot on huge write rates.*
**Q8 (M) Prefix index: when and limits?** Long strings; `INDEX(col(20))`; choose length by distinct-ratio; cannot cover, cannot fully serve ORDER BY. *F: Alternative? -> hash column (generated `CRC32/SHA` + index) for equality.*
**Q9 (M) Functional index in MySQL?** 8.0.13+: `CREATE INDEX i ON t ((LOWER(email)))`; the query must use the same expression. Alternatively generated column + index, or `_ci` collation. *F: PG partial index? -> `WHERE status='OPEN'` index, small and hot; MySQL none.*
**Q10 (M) How do you safely remove an index?** `ALTER INDEX ... INVISIBLE`, watch p99 and slow log across a business cycle, then `DROP`; rollback = `VISIBLE` (instant). Check `sys.schema_unused_indexes`/`schema_redundant_indexes` first; FK-supporting and unique indexes need care.
**Q11 (H) Estimate the height of a B+ tree for 1M and 100M rows.** Fan-out ~1000 (16 KB / ~16 B key+pointer), rows/leaf ~16 at 1 KB -> 1M rows = 62.5k leaves -> 63 -> 1 root = height 3; 100M = 6.25M leaves -> 6.25k -> 7 -> 1 = height 4. Upper levels are cached so ~1 disk read. *F: What changes with 400 B keys? -> fan-out ~40, height 5+. F: Why does a wide PK hurt? -> shrinks secondary fan-out and grows all secondary indexes.*
**Q12 (H) Cardinality and statistics: what goes wrong?** Optimizer estimates rows from sampled index stats (20 pages default) and histograms; skew or stale stats -> wrong index/join order. Fix: `ANALYZE TABLE`, `UPDATE HISTOGRAM`, compare EXPLAIN vs EXPLAIN ANALYZE, last resort hints. *F: Why did the plan flip after a bulk load? -> stats auto-recalc threshold/sampling variance.*
**Q13 (H) Do hash indexes exist in InnoDB?** No user-created; InnoDB builds an adaptive hash index automatically over hot pages (equality only); can be a contention point. MEMORY engine has real hash indexes. Hash = no range/order.

## 17.2 Optimizer, EXPLAIN, query tuning
**Q14 (E) What does EXPLAIN show?** The chosen plan with estimates: access `type`, `key`, `rows`, `filtered`, `Extra`. `EXPLAIN ANALYZE` (8.0.18+) executes and shows actual time/rows/loops.
**Q15 (M) Rank access types.** system/const > eq_ref > ref > range > index > ALL. *F: `index` vs `ALL`? -> `index` scans a whole index (cheaper if covering); ALL scans the table.*
**Q16 (M) Read: `type=ref, key=idx_a, rows=380000, filtered=10, Extra=Using where; Using filesort`.** Index finds 380k rows, 90% discarded by a non-indexed predicate, then sorted. Fix: composite `(a, filter_col, sort_col)`. *F: What if rows estimate is 100 but actual is 380000? -> stale stats/skew: ANALYZE, histogram.*
**Q17 (M) A query has an index but does not use it. Why?** Function/expression on column, implicit type cast, leading wildcard, OR over different columns, `<>`/NOT IN, low selectivity (scan cheaper), collation/charset mismatch in joins, leftmost-prefix violation, stale stats. *F: How prove it? -> EXPLAIN `possible_keys` vs `key`, `FORCE INDEX` to compare costs.*
**Q18 (M) Explain SARGability.** Predicate with bare indexed column vs constant can become a seek: `created >= x AND created < y` not `DATE(created)=x`. Half-open ranges for timestamps.
**Q19 (M) Join algorithms in MySQL 8?** Index nested loop (ref/eq_ref), hash join (8.0.18+, replaced BNL in 8.0.20, equi-join without index), no merge join. PG/Oracle also have merge join. *F: How to tell hash join in EXPLAIN? -> `Using join buffer (hash join)`; the fix is usually an index on the join column.*
**Q20 (M) How does the optimizer pick join order?** Cost-based on estimated rows: start from the most selective/smallest filtered table, prefer joins via index; greedy search with depth limit. *F: When use STRAIGHT_JOIN? -> proven bad order due to wrong estimates.*
**Q21 (H) Why is `SELECT COUNT(*)` slow on InnoDB and what can you do?** No stored row count due to MVCC; scans smallest index. Use estimates, counters, cached count, cap with `LIMIT`, or avoid total counts in pagination.
**Q22 (M) Explain N+1 and fixes.** 1 + N queries, latency-bound; fix with JOIN/`IN` batch/`@BatchSize`/entity graph; detect by statement count per request. Beware cartesian explosion joining two collections.
**Q23 (H) OFFSET pagination on 50M rows is slow: options?** OFFSET must produce and discard rows; use keyset `WHERE (k, id) > (?, ?) ORDER BY k, id LIMIT n` with a matching index, or deferred join through a covering index. Trade-offs: no random page access. Measured 900k-offset vs keyset in section 6.2. *F: Ties in the sort key? -> add unique tiebreaker. F: Sorting by non-unique, descending? -> same with `<` and DESC index.*
**Q24 (M) IN vs EXISTS vs JOIN?** Modern optimizers turn IN/EXISTS into semi-joins; JOIN can duplicate rows; NOT IN is NULL-hostile so use NOT EXISTS. Verify with EXPLAIN.
**Q25 (M) Rewrite a correlated subquery.** Pre-aggregate in a derived table and join, or use a window function; index the correlating columns.
**Q26 (M) UNION vs UNION ALL?** UNION dedupes (sort/temp), ALL only appends; prefer ALL.
**Q27 (H) A query is fast in dev and slow in prod. Why?** Data volume/skew changes plan, stats differ, cold buffer pool, parameter sniffing-like plan choice on bound values (histograms), different indexes/collations, lock contention, replica vs primary. Reproduce with production-size data and EXPLAIN ANALYZE.
**Q28 (M) Type of query the slow log misses?** N+1 (many fast queries) and lock waits (Lock_time). Use digest summaries by total time, not just slowest.

## 17.3 SQL semantics
**Q29 (E) WHERE vs HAVING?** WHERE filters rows before grouping (cannot use aggregates); HAVING filters groups after.
**Q30 (E) DELETE vs TRUNCATE vs DROP?** DELETE: row-by-row, WHERE, logged per row, triggers, rollback-able; TRUNCATE: deallocates, resets auto-increment, DDL (implicit commit in MySQL), no WHERE, fast; DROP removes table definition. *F: TRUNCATE rollback? -> not in MySQL/Oracle (DDL), transactional in PG.*
**Q31 (E) INNER vs LEFT JOIN? Anti-join?** LEFT keeps unmatched left rows with NULLs; anti-join = LEFT + `right.pk IS NULL` or NOT EXISTS.
**Q32 (M) Explain `NOT IN` with NULL.** `x NOT IN (…NULL…)` is UNKNOWN for every x that is not found -> no rows. Use NOT EXISTS or filter NULLs. Validated in 7.1 (0 vs 1).
**Q33 (M) COUNT(*) vs COUNT(col) vs COUNT(DISTINCT col)?** All rows; non-NULL values; distinct non-NULL values.
**Q34 (M) ROW_NUMBER vs RANK vs DENSE_RANK?** Unique sequence; ties same rank with gaps; ties same rank no gaps. Nth highest distinct salary -> DENSE_RANK.
**Q35 (M) Window function vs GROUP BY?** Window keeps rows and adds computed columns; GROUP BY collapses. Windows evaluated after WHERE/GROUP BY/HAVING, so cannot be filtered in WHERE directly.
**Q36 (H) Default window frame gotcha?** With ORDER BY the default is RANGE UNBOUNDED PRECEDING..CURRENT ROW: peers with equal keys get the same running total; `LAST_VALUE` returns the current row. Write ROWS frames explicitly.
**Q37 (M) What is a CTE; recursive CTE; when not to use?** Named temporary result; recursive = anchor + recursive member until no rows (org trees, series). Guard cycles/depth. Not automatically materialized/faster.
**Q38 (M) LEFT JOIN with a WHERE condition on the right table?** Turns into an inner join; move to ON.
**Q39 (H) Find users with 3+ consecutive login days.** Distinct (uid, day), `day - ROW_NUMBER()` constant per island, group and `HAVING COUNT(*) >= 3` (Q25).

## 17.4 Transactions, isolation, locking
**Q40 (E) ACID in one line each and who provides them in InnoDB.** Atomicity=undo log, Consistency=constraints+app, Isolation=MVCC+locks, Durability=redo log fsync (+binlog).
**Q41 (M) Isolation levels and anomalies?** RU dirty reads; RC non-repeatable reads/phantoms; RR (InnoDB default) snapshot per transaction; SERIALIZABLE serial-equivalent (InnoDB: shared locks; PG: SSI aborts). *F: Defaults of PG/Oracle? -> RC. F: Does RR stop lost update in InnoDB? -> no (use atomic update/FOR UPDATE/version).*
**Q42 (H) Explain MVCC in InnoDB.** Hidden `DB_TRX_ID`/`DB_ROLL_PTR`; old versions in undo forming a chain; read view (active ids, min, max) decides visibility; RR builds it once, RC per statement; purge removes undo when no view needs it. Walk-through in 8.2. *F: Why do long transactions hurt? -> undo cannot be purged, history list grows, longer chains. F: Do readers block writers? -> not for consistent reads.*
**Q43 (H) Show a phantom under InnoDB REPEATABLE READ.** Snapshot read sees no new rows; but `UPDATE ... WHERE` (current read) in the same tx touches rows inserted by others, which then become visible (8.4c). Locking reads take next-key locks to prevent it.
**Q44 (H) Gap lock and next-key lock: what and why?** Lock on the interval between index records (gap) or record+gap (next-key) in RR to block inserts into scanned ranges; gap locks are compatible with each other; disabled in RC; without an index on the filter, a huge range/all rows get locked. *F: Insert-intention lock? -> gap-lock variant an INSERT requests; it conflicts with gap locks -> source of insert deadlocks.*
**Q45 (H) Deadlock: how detected, who is victim, what should the app do?** Wait-for graph cycle; victim = cheapest to roll back (fewest changes); whole tx rolled back with error 1213; retry entire transaction with backoff; prevent via ordering, short tx, indexes, RC. *F: Lock wait timeout differs how? -> 50 s default, error 1205, statement rollback only (tx still open).*
**Q46 (H) Write skew and fix?** Two tx read overlapping data and write different rows breaking an invariant under snapshot isolation. Fix: lock the read set (`FOR UPDATE`), materialize conflict on one row, or SERIALIZABLE/retry.
**Q47 (M) `SELECT ... FOR UPDATE SKIP LOCKED` use?** Work-queue: workers claim distinct rows without blocking; inconsistent view by design. NOWAIT fails fast (3572).
**Q48 (M) How do you prevent overselling stock?** Atomic conditional update `UPDATE p SET stock=stock-? WHERE id=? AND stock>=?` and check affected rows (or FOR UPDATE then update, or version column); never read-then-write without a lock.
**Q49 (M) Pessimistic vs optimistic locking?** FOR UPDATE blocks (good under high contention/short tx); `@Version` column compare-and-swap (good under low contention, retries on conflict).
**Q50 (H) Design to avoid deadlocks?** Consistent ordering, short transactions, no remote calls inside, index WHERE columns, atomic statements, retry, chunked bulk work, RC where gap locks unneeded, split hot rows.

## 17.5 Durability, replication, scale, schema, ops
**Q51 (M) redo vs undo vs binlog?** Redo = crash recovery (roll forward, InnoDB); undo = rollback and MVCC; binlog = server-level logical log for replication/PITR/CDC. Two-phase commit keeps redo and binlog consistent.
**Q52 (H) What do `innodb_flush_log_at_trx_commit` and `sync_binlog` control?** Commit-time fsync policy; 1 = fsync each commit (no loss); 2 = OS-flush each commit, fsync/second (loses up to ~1 s on OS/power crash only); 0 = per second even on mysqld crash. Double-1 for money.
**Q53 (H) Describe crash recovery.** Load checkpoint LSN, redo replay (doublewrite fixes torn pages), decide prepared transactions from binlog, undo uncommitted, resume purge.
**Q54 (M) Async vs semi-sync replication and failure modes?** Async: lag and lost commits on failover; semi-sync: waits for one replica's relay-log ack (lossless AFTER_SYNC), adds latency, degrades to async on timeout. *F: Split brain? -> fencing.*
**Q55 (H) Read-your-writes with replicas?** Sticky primary reads after writes, GTID wait tokens, route critical reads to primary, return written data. Root causes of lag: single-thread apply, big transactions, DDL.
**Q56 (M) Partitioning vs sharding?** Same server, transparent, maintenance/pruning vs multi-server, app-aware, scales writes. Partition key must be in every unique key; no FKs in MySQL partitioned InnoDB.
**Q57 (H) Choose a shard key.** High cardinality, even, in most queries, immutable, keeps transactions single-shard; beware hot shards (time keys, whales); plan resharding (logical shards).
**Q58 (M) How big should the connection pool be?** Small: Little's law (tps x hold time) or ~2 x cores; bounded by instances x pool <= max_connections. Bigger pools slow the DB.
**Q59 (M) DECIMAL vs FLOAT for money; DATETIME vs TIMESTAMP?** DECIMAL/minor-unit ints; TIMESTAMP is UTC-converted and ends 2038; store UTC.
**Q60 (H) Add a column to a 500M-row table with zero downtime?** `ALGORITHM=INSTANT` if supported, else gh-ost/pt-osc; watch MDL waits (`lock_wait_timeout`); expand/contract for renames or type changes; backfill in chunks.
**Q61 (M) Backup and PITR?** Full physical/logical backup + binlog replay to a timestamp/GTID; test restores; replicas are not backups.
**Q62 (M) When choose NoSQL, and CAP?** Measured need (write scale, key-value latency, document shape, search, graph, analytics). CAP: on partition choose C or A; PACELC adds latency-vs-consistency in normal operation.
**Q63 (M) Cache-aside pitfalls?** Stampede, penetration, stale after write (delete not update, TTL backstop), replica re-populating stale data.
**Q64 (H) 3 a.m. alert: DB CPU 95%, pool exhausted. Steps?** Processlist/`Threads_running`, top digests, `INNODB_TRX` for long tx, lock waits, recent deploy/plan flip, kill offenders, add index via online DDL, shed load (rate limit), postmortem with EXPLAIN evidence (section 13.3).

## 17.6 Common wrong answers
| Claim | Correction |
|---|---|
| "More indexes always speed things up" | each write updates all indexes; unused indexes only cost |
| "Index on gender/boolean helps" | selectivity ~50%; scan wins; use composite or none |
| "`COUNT(1)` is faster than `COUNT(*)`" | identical in InnoDB/PG/Oracle |
| "`OFFSET` is fine, DB jumps to the row" | it reads and discards all skipped rows |
| "`LIMIT 1 OFFSET 1` gives second highest" | wrong with duplicates |
| "InnoDB REPEATABLE READ has phantoms like the standard" | snapshot reads no; locking reads use next-key locks; mixed use can leak |
| "REPEATABLE READ prevents lost updates" | not in InnoDB for read-then-write in the app |
| "Deadlock = lock wait timeout" | deadlock is a detected cycle (1213, immediate); timeout is 1205 after 50 s |
| "Read Committed allows dirty reads" | it only reads committed data; dirty reads are READ UNCOMMITTED |
| "`SELECT *` is harmless" | defeats covering indexes, moves TEXT/BLOB, breaks with schema change |
| "`NOT IN` and `NOT EXISTS` are equivalent" | differ under NULL |
| "Foreign keys are unnecessary because the app validates" | app checks race; FK/unique are the real guard |
| "Replicas are backups" | `DROP TABLE` replicates |
| "Sharding first when slow" | fix queries/indexes/cache/replicas first |
| "`TIMESTAMP` and `DATETIME` are the same" | zone conversion and 2038 limit |
| "`EXPLAIN ANALYZE` is just EXPLAIN with more info" | it executes the statement |
| "EXISTS is always faster than IN" | 5.5-era folklore; modern optimizers use semi-joins |
| "UUID is always better for scale" | random UUID PK fragments the clustered index |

---

# 18. One-page cheat sheet

**Index**
- InnoDB page 16 KB; fan-out ~1000; height 3 (1M) / 4 (100M). PK = clustered = table; secondary leaf = (key + PK) -> bookmark lookup unless covering.
- Composite: equality cols -> one range col -> sort cols. Leftmost prefix. Columns after first range col do not seek.
- Not used: function/cast on column, leading `%`, OR across cols, type/collation mismatch, low selectivity, stale stats.
- Tools: `EXPLAIN`, `EXPLAIN ANALYZE`, `ANALYZE TABLE`, invisible index, `sys.schema_redundant_indexes`, functional index `((expr))`.
- PK: BIGINT auto-inc or time-ordered ids; avoid random UUID/CHAR(36).

**EXPLAIN**
- `type`: const > eq_ref > ref > range > index > ALL. `Extra`: good = Using index / index condition; suspicious = Using filesort / temporary / join buffer.
- `rows` x `filtered` = rows passing on. Compare estimate to actual (ANALYZE).

**Query patterns**
- Pagination: keyset `(k,id) > (?,?)`; deferred join; avoid OFFSET deep. COUNT(*) scans in InnoDB.
- NOT EXISTS over NOT IN. UNION ALL over UNION. Semi/anti joins. Half-open time ranges.
- Windows: ROW_NUMBER unique; RANK gaps; DENSE_RANK none; default frame RANGE; islands = `x - ROW_NUMBER()`; sessions = LAG flag + running SUM.

**Transactions**
- Isolation: RU dirty; RC non-repeatable; RR (InnoDB default) snapshot; SER locks. PG/Oracle default RC.
- MVCC: trx_id + roll_ptr + undo chain + read view (RR once, RC per statement). Purge blocked by long tx.
- Lost update -> atomic UPDATE / FOR UPDATE / @Version. Write skew -> lock read set / SERIALIZABLE.
- Locks: record, gap, next-key (RR), insert-intention, IS/IX, MDL. No index = huge lock range. `FOR UPDATE NOWAIT|SKIP LOCKED`.
- Deadlock 1213 (retry whole tx); lock wait 1205 after 50 s (statement only). Order locks, short tx, index, retry.
- Diagnose: `SHOW ENGINE INNODB STATUS`, `performance_schema.data_locks`, `sys.innodb_lock_waits`, `INNODB_TRX`.

**Durability and scale**
- Redo (crash), undo (rollback/MVCC), binlog (replication/PITR). Double-1: `innodb_flush_log_at_trx_commit=1`, `sync_binlog=1`.
- Async replication lags; semi-sync waits for relay-log ack; fix read-your-writes with primary reads/GTID wait.
- Partition (one server, pruning, drop old data) vs shard (many servers, shard key, no cross-shard tx).
- Pool = tps x hold time; instances x pool <= max_connections.

**Schema and ops**
- 3NF write model; denormalize read models. Surrogate PK + unique natural key. DECIMAL money. UTC. Avoid ENUM/JSON for filtered data. BIGINT ids.
- DDL: INSTANT > INPLACE > COPY; gh-ost/pt-osc; MDL hazard; expand/contract; backup + binlog PITR; test restores.
- Observe: slow log + pt-query-digest, `events_statements_summary_by_digest`, `sys.statement_analysis`, `Rows_examined/Rows_sent`.
- Cache: cache-aside, delete on write, TTL + jitter, single-flight. NoSQL only for measured needs (CAP: C or A under partition).

**Dialect quick map**: LIMIT/OFFSET (MySQL, PG) = `FETCH FIRST n ROWS ONLY` (Oracle 12c+); `GROUP_CONCAT` = `string_agg` (PG) = `LISTAGG` (Oracle); `''` is NULL in Oracle; PG `VACUUM`/`jsonb`/partial+INCLUDE indexes; Oracle `MINUS`, `CONNECT BY`, `DUAL`; PG default RC, RR = snapshot isolation, SSI serializable.
