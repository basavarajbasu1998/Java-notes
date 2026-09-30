# SQL & Database Performance

## Index — the book index
Without index: **full table scan** (read every row). With index: B+ tree lookup → O(log n).

```
Query: SELECT * FROM users WHERE email='a@x.com'

No index:  scan row1, row2, ... row 5,000,000     (slow)
Index:     root → branch → leaf → row pointer      (3-4 reads)
```
- Primary key → clustered index (data stored in key order in InnoDB).
- Cost of indexes: slower INSERT/UPDATE/DELETE + storage. Don't index everything.

**Composite index — leftmost prefix rule**
Index on `(country, city, age)`:
- ✔ `WHERE country=?`, `country=? AND city=?`, all three
- ✘ `WHERE city=?` alone (skips leftmost column)

**When index is NOT used:**
- Function on column: `WHERE YEAR(created)=2024`, `LOWER(email)=…`
- Leading wildcard: `LIKE '%son'` (but `LIKE 'son%'` works)
- Type mismatch (string column compared to number)
- Low-selectivity column (gender) – optimizer prefers scan
- `OR` across non-indexed columns

**Covering index:** index contains all selected columns → DB never touches the table.

## Slow query investigation flow
```
User complains: "page is slow"
   ▼
Find slow query (slow-query log / APM / Hibernate show_sql)
   ▼
EXPLAIN / EXPLAIN ANALYZE
   ▼
type = ALL (full scan)? rows huge? "Using filesort"?
   ├─ yes → add/fix index, rewrite query
   └─ no  → N+1 in app? too many round trips? missing pagination? connection pool exhausted?
   ▼
Re-measure
```

## Joins
| Join | Returns |
|---|---|
| INNER | only matching rows in both |
| LEFT | all left rows + matches (NULL if none) |
| RIGHT | all right + matches |
| FULL | everything from both |
| CROSS | every combination (m×n) |

## Typical SQL interview questions
```sql
-- 2nd highest salary
SELECT MAX(salary) FROM emp WHERE salary < (SELECT MAX(salary) FROM emp);
-- or:  SELECT salary FROM emp ORDER BY salary DESC LIMIT 1 OFFSET 1;

-- Nth highest with ties handled
SELECT DISTINCT salary FROM (
   SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) rk FROM emp) t
WHERE rk = 3;

-- Duplicates
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Delete duplicates keep lowest id
DELETE FROM users WHERE id NOT IN (SELECT MIN(id) FROM users GROUP BY email);

-- Employees earning more than their manager
SELECT e.name FROM emp e JOIN emp m ON e.manager_id = m.id WHERE e.salary > m.salary;
```
- `WHERE` filters rows **before** grouping; `HAVING` filters **after** `GROUP BY`.
- `DELETE` (row by row, rollback-able, WHERE) vs `TRUNCATE` (fast, resets) vs `DROP` (removes table).
- `UNION` (removes duplicates) vs `UNION ALL` (faster).
- Window functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG/LEAD`.

## ACID
**A**tomicity (all or nothing), **C**onsistency (rules/constraints always hold), **I**solation (concurrent tx don't interfere – see isolation levels in `Spring_Transactional.md`), **D**urability (committed = survives crash).

## Others
- **Normalization** (remove redundancy: 1NF, 2NF, 3NF) vs **denormalization** (duplicate on purpose for read speed).
- **Deadlock in DB:** tx1 locks row A wants B; tx2 locks B wants A. DB kills one → retry. Prevent: access rows in consistent order, keep tx short.
- **Connection pool (HikariCP):** size small (10–20 typical). Pool exhausted → requests hang; usually caused by long transactions or leaked connections.
- **SQL vs NoSQL:** SQL = relations, ACID, joins. NoSQL (Mongo/Cassandra/Redis) = scale, flexible schema, eventual consistency.

---

## Worked Example in the Order Project

```sql
CREATE TABLE customers (id BIGINT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(100) NOT NULL, email VARCHAR(150) UNIQUE NOT NULL);
CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  customer_id BIGINT NOT NULL,
  total DECIMAL(12,2) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (customer_id) REFERENCES customers(id)
);
CREATE INDEX idx_orders_customer_created ON orders (customer_id, created_at DESC);
CREATE TABLE order_items (
  id BIGINT PRIMARY KEY AUTO_INCREMENT, order_id BIGINT NOT NULL, product_id BIGINT NOT NULL,
  qty INT NOT NULL, price DECIMAL(10,2) NOT NULL,
  FOREIGN KEY (order_id) REFERENCES orders(id)
);
```
Relationships: customers 1—* orders 1—* order_items *—1 products.

## Query flow (what the database does)
```
SQL text → Parser → Optimizer (choose index / join order, using statistics) → Executor → result
EXPLAIN shows the chosen plan.
```
Business queries:
```sql
-- customer's last 10 orders (uses composite index)
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 10;

-- revenue per day, only busy days
SELECT DATE(created_at) d, SUM(total) revenue, COUNT(*) cnt
FROM orders WHERE status = 'PAID' GROUP BY DATE(created_at) HAVING COUNT(*) > 100 ORDER BY d;

-- customers who never ordered
SELECT c.* FROM customers c LEFT JOIN orders o ON o.customer_id = c.id WHERE o.id IS NULL;

-- top spender per customer using window function
SELECT * FROM (SELECT customer_id, total, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total DESC) rn FROM orders) t WHERE rn = 1;
```
Transaction (stock reservation, prevents overselling):
```sql
START TRANSACTION;
SELECT stock FROM products WHERE id = 7 FOR UPDATE;     -- row lock
UPDATE products SET stock = stock - 2 WHERE id = 7 AND stock >= 2;   -- safe conditional update
-- affected rows = 0 → out of stock → ROLLBACK
COMMIT;
```
Deeper (ACID, isolation levels, indexing rules, joins): `Spring_Transactional.md` (isolation levels) and the sections above. Also: view, stored procedure, trigger, CTE (`WITH`), pagination keyset, sharding/partitioning, replication (primary writes, replicas read), SQL injection → always parameterised queries.
