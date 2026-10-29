# Question 4: Joins, Subqueries, Set Operations & EXPLAIN Plan Analysis

**Problem Statement:**  
Using the Employee schema, write queries with `INNER JOIN`, `LEFT JOIN`, self-join, 3-way join, correlated subqueries, `EXISTS`, and simulated `INTERSECT` and `EXCEPT`. Compare execution plans using `EXPLAIN`.

## 1. Aim
To implement relational joins (`INNER`, `LEFT`, self-join, 3-way join), correlated subqueries, `EXISTS` predicates, and simulated set operations (`INTERSECT`, `EXCEPT`) on the Employee–Department–Project schema, and evaluate their query execution plans using MySQL `EXPLAIN`.

## 2. Objectives
1. Write SQL queries demonstrating `INNER JOIN`, `LEFT JOIN`, self-join (`Employee`–`Manager`), and 3-way join (`Employee`–`Department`–`Project`).
2. Construct a correlated subquery and an `EXISTS` query to filter employees based on departmental and project criteria.
3. Simulate `INTERSECT` and `EXCEPT` in MySQL using `INNER JOIN`/`IN` and `LEFT JOIN ... IS NULL`/`NOT EXISTS`.
4. Compare optimizer execution plans (`type`, `key`, `rows`, `Extra`) using `EXPLAIN`.

## 3. Query Execution Flow Diagram

```mermaid
flowchart LR
    Q["SQL Query<br/>(JOIN / Subquery / EXISTS)"] --> PARSER["MySQL Parser &<br/>Query Optimizer"]
    PARSER --> EXP["EXPLAIN Plan<br/>(type: ALL / ref / eq_ref)"]
    EXP --> EXEC["Storage Engine Execution<br/>& Result Set"]
```

## 4. Input (SQL Queries & `EXPLAIN` Statements)

```sql
USE Company_HR;

-- 1. INNER JOIN: Employees with their Department details
SELECT e.Emp_ID, e.Emp_Name, d.Dept_Name, d.Location
FROM Employee e
INNER JOIN Department d ON e.Dept_ID = d.Dept_ID;

-- 2. LEFT JOIN: All Employees and their Project assignments (including unassigned VPs/Directors)
SELECT e.Emp_ID, e.Emp_Name, e.Job_Role, ep.Proj_ID, ep.Hours_Worked
FROM Employee e
LEFT JOIN Employee_Project ep ON e.Emp_ID = ep.Emp_ID
WHERE ep.Proj_ID IS NULL;

-- 3. SELF-JOIN: Employees with their direct Manager's name
SELECT e.Emp_ID, e.Emp_Name AS Employee, m.Emp_Name AS Manager_Name
FROM Employee e
LEFT JOIN Employee m ON e.Manager_ID = m.Emp_ID;

-- 4. 3-WAY JOIN: Employee -> Employee_Project -> Project -> Department
SELECT e.Emp_Name, d.Dept_Name, p.Proj_Name, ep.Hours_Worked
FROM Employee e
INNER JOIN Employee_Project ep ON e.Emp_ID = ep.Emp_ID
INNER JOIN Project p ON ep.Proj_ID = p.Proj_ID
INNER JOIN Department d ON e.Dept_ID = d.Dept_ID
WHERE ep.Hours_Worked >= 150;

-- 5. CORRELATED SUBQUERY: Employees earning more than their own Department's average salary
SELECT e1.Emp_ID, e1.Emp_Name, e1.Dept_ID, e1.Salary
FROM Employee e1
WHERE e1.Salary > (
    SELECT AVG(e2.Salary)
    FROM Employee e2
    WHERE e2.Dept_ID = e1.Dept_ID
);

-- 6. EXISTS: Employees assigned to at least one high-budget project (> 2,000,000)
SELECT e.Emp_ID, e.Emp_Name, e.Job_Role
FROM Employee e
WHERE EXISTS (
    SELECT 1
    FROM Employee_Project ep
    INNER JOIN Project p ON ep.Proj_ID = p.Proj_ID
    WHERE ep.Emp_ID = e.Emp_ID AND p.Proj_Budget > 2000000.00
);

-- 7. SIMULATED INTERSECT: Employees in Dept 10 (Engineering) INTERSECT Employees working > 130 hours
SELECT DISTINCT e.Emp_ID, e.Emp_Name
FROM Employee e
INNER JOIN Employee_Project ep ON e.Emp_ID = ep.Emp_ID
WHERE e.Dept_ID = 10 AND ep.Hours_Worked > 130;

-- 8. SIMULATED EXCEPT (MINUS): All Employees EXCEPT those assigned to any Project
SELECT e.Emp_ID, e.Emp_Name, e.Job_Role
FROM Employee e
LEFT JOIN Employee_Project ep ON e.Emp_ID = ep.Emp_ID
WHERE ep.Emp_ID IS NULL;

-- 9. EXPLAIN Plan Comparison (Correlated Subquery vs Derived Table JOIN)
EXPLAIN SELECT e1.Emp_ID, e1.Emp_Name, e1.Salary
FROM Employee e1
WHERE e1.Salary > (SELECT AVG(e2.Salary) FROM Employee e2 WHERE e2.Dept_ID = e1.Dept_ID);

EXPLAIN SELECT e.Emp_ID, e.Emp_Name, e.Salary
FROM Employee e
INNER JOIN (SELECT Dept_ID, AVG(Salary) AS avg_sal FROM Employee GROUP BY Dept_ID) dt
ON e.Dept_ID = dt.Dept_ID AND e.Salary > dt.avg_sal;
```

## 5. Output

### 5.1 Query Results Summary

- **3-Way Join Output (`Hours_Worked >= 150`):**

| Emp_Name | Dept_Name | Proj_Name | Hours_Worked |
| :--- | :--- | :--- | ---: |
| Rohan Joshi | Engineering | UPI Gateway v3 | 160 |
| Pooja Sharma | Engineering | Microservices Migration | 150 |
| Divya Menon | Data Science | Fraud Detection AI | 155 |
| Swati Kulkarni | Finance | GST Auto-Reconciliation | 150 |
| Pranav Gokhale | Cloud Operations | Multi-Cloud Kubernetes | 160 |
| Shruti Bhardwaj | Cloud Operations | Multi-Cloud Kubernetes | 150 |

- **Correlated Subquery Output (Earning above Departmental Average):** Returns 11 senior/lead employees (`Emp_ID`: `1, 2, 3, 4, 5, 6, 7, 12, 13, 18, 27`).
- **Simulated `INTERSECT` Output:** Returns `Kavita Reddy (Emp_ID: 7)` and `Rohan Joshi (Emp_ID: 8)` (` Pooja Sharma (Emp_ID: 9)`).
- **Simulated `EXCEPT` / `LEFT JOIN IS NULL` Output:** Returns the 8 employees not assigned to project tables (`Emp_ID`: `1, 2, 3, 4, 5, 21, 22, 26`).

### 5.2 `EXPLAIN` Execution Plan Comparison

| Query Variant | `select_type` | `table` | `type` | `possible_keys` | `key` | `rows` | `Extra` | Performance Analysis |
| :--- | :--- | :--- | :--- | :--- | :--- | ---: | :--- | :--- |
| **3-Way `INNER JOIN`** | `SIMPLE` | `ep` / `e` / `p` / `d` | `ALL` $\rightarrow$ `eq_ref` | `PRIMARY`, `fk_ep_emp` | `PRIMARY` | `22` $\rightarrow$ `1` | `Using where` | Uses primary key index (`eq_ref`) for `1`-row lookups on `Employee`, `Project`, and `Department`. |
| **Correlated Subquery** | `PRIMARY` + `DEPENDENT SUBQUERY` | `e1` $\rightarrow$ `e2` | `ALL` $\rightarrow$ `ref` | `fk_emp_dept` | `fk_emp_dept` | `30` $\times$ `6` | `Using where` | Evaluates dependent subquery per outer row unless rewritten by semi-join/materialization. |
| **Derived Table `JOIN` Rewrite** | `PRIMARY` + `DERIVED` | `<derived2>` $\rightarrow$ `e` | `ALL` $\rightarrow$ `ref` | `fk_emp_dept` | `fk_emp_dept` | `5` $\rightarrow$ `6` | `Using temporary` | Computes departmental averages once (`5` groups) and joins via `fk_emp_dept`, avoiding repeated scans. |
