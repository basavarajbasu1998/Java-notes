# Coding Problems: SQL
```sql
-- Nth highest salary (N=3)
SELECT salary FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) rk FROM emp) t WHERE rk = 3;

-- Highest salary per department
SELECT dept, MAX(salary) FROM emp GROUP BY dept;
-- Employee(s) with highest salary per dept
SELECT * FROM emp e WHERE salary = (SELECT MAX(salary) FROM emp WHERE dept = e.dept);

-- Departments with more than 5 employees
SELECT dept, COUNT(*) FROM emp GROUP BY dept HAVING COUNT(*) > 5;

-- Employees with no orders
SELECT e.* FROM emp e LEFT JOIN orders o ON o.emp_id = e.id WHERE o.id IS NULL;

-- Duplicate emails
SELECT email FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Running total
SELECT id, amount, SUM(amount) OVER (ORDER BY id) AS running FROM payments;

-- Top 3 per department
SELECT * FROM (SELECT e.*, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) rn FROM emp e) t WHERE rn <= 3;

-- Consecutive days / gaps: use LAG(date) OVER (ORDER BY date)
-- Swap gender: UPDATE t SET sex = CASE sex WHEN 'M' THEN 'F' ELSE 'M' END;
```
