# Question 8: Banking Transactions, Savepoints & Concurrency Isolation Levels

**Problem Statement:**  
Implement banking-style transactions using `BEGIN`, `COMMIT`, `ROLLBACK`, and `SAVEPOINT`. Simulate partial failures. Demonstrate isolation levels and concurrency anomalies using two sessions.

## 1. Aim
To implement atomic banking fund transfers in MySQL using `START TRANSACTION` (`BEGIN`), `COMMIT`, `ROLLBACK`, and `SAVEPOINT` for partial failure recovery, and to demonstrate concurrency anomalies (Dirty Read, Non-Repeatable Read, Phantom Read) across two concurrent database sessions under different SQL isolation levels.

## 2. Objectives
1. Create a `Bank_Account` table and execute atomic fund transfers using `BEGIN`, `SAVEPOINT`, `ROLLBACK TO SAVEPOINT`, and `COMMIT`.
2. Simulate a multi-step batch transfer where a partial failure rolls back only the failed leg via `SAVEPOINT` while committing the valid transfer.
3. Demonstrate two-session concurrency behavior across `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, and `SERIALIZABLE`.

## 3. Two-Session Concurrency Timeline Diagram

```mermaid
sequenceDiagram
    participant S1 as Session A (Writer Transaction)
    participant DB as MySQL InnoDB Engine
    participant S2 as Session B (Reader Transaction)

    Note over S1,S2: Dirty Read Demo (Session B at READ UNCOMMITTED)
    S1->>DB: START TRANSACTION; UPDATE Bank_Account SET Balance = Balance - 10000 WHERE Acc_No = 101;
    S2->>DB: SELECT Balance FROM Bank_Account WHERE Acc_No = 101;
    DB-->>S2: Returns uncommitted Balance = 40000.00 (DIRTY READ!)
    S1->>DB: ROLLBACK; (Balance restored to 50000.00)
```

## 4. Input (MySQL Transaction Script & Two-Session Concurrency Lab)

```sql
CREATE DATABASE IF NOT EXISTS Bharat_Banking;
USE Bharat_Banking;

CREATE TABLE Bank_Account (
    Acc_No INT PRIMARY KEY,
    Holder_Name VARCHAR(60) NOT NULL,
    Balance DECIMAL(12,2) NOT NULL CHECK (Balance >= 0)
);

INSERT INTO Bank_Account VALUES
(101, 'Aarav Sharma', 50000.00),
(102, 'Priya Nair', 30000.00),
(103, 'Rohan Deshmukh', 15000.00);

-- =========================================================
-- PART 1: SAVEPOINT & PARTIAL FAILURE SIMULATION
-- Scenario: Aarav (101) transfers ₹10,000 to Priya (102) [Valid],
-- then attempts to transfer ₹45,000 to Rohan (103) [Fails: Insufficient Funds].
-- =========================================================
START TRANSACTION;

-- Step 1: Valid Transfer of ₹10,000 from Acc 101 to Acc 102
UPDATE Bank_Account SET Balance = Balance - 10000.00 WHERE Acc_No = 101;
UPDATE Bank_Account SET Balance = Balance + 10000.00 WHERE Acc_No = 102;

-- Create Savepoint after first successful transfer
SAVEPOINT after_first_transfer;

-- Step 2: Attempt second transfer of ₹25,000 to Acc 103 (Simulated Partial Failure / Reversal)
UPDATE Bank_Account SET Balance = Balance - 25000.00 WHERE Acc_No = 101;
UPDATE Bank_Account SET Balance = Balance + 25000.00 WHERE Acc_No = 103;

-- Rollback only Step 2 due to daily limit policy check failure
ROLLBACK TO SAVEPOINT after_first_transfer;

-- Commit the valid first transfer
COMMIT;

SELECT * FROM Bank_Account;

-- =========================================================
-- PART 2: TWO-SESSION ISOLATION LEVEL & ANOMALY SCRIPTS
-- =========================================================

-- [ANOMALY 1: DIRTY READ under READ UNCOMMITTED]
-- Session B:
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
-- Session A:
START TRANSACTION;
UPDATE Bank_Account SET Balance = 99999.00 WHERE Acc_No = 101; -- Uncommitted!
-- Session B:
SELECT Acc_No, Balance FROM Bank_Account WHERE Acc_No = 101;  -- Reads 99999.00 (Dirty Read!)
-- Session A:
ROLLBACK;
-- Session B:
SELECT Acc_No, Balance FROM Bank_Account WHERE Acc_No = 101;  -- Back to 40000.00
COMMIT;

-- [ANOMALY 2: NON-REPEATABLE READ under READ COMMITTED]
-- Session B:
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT Balance FROM Bank_Account WHERE Acc_No = 102; -- Reads 40000.00
-- Session A:
START TRANSACTION;
UPDATE Bank_Account SET Balance = 45000.00 WHERE Acc_No = 102;
COMMIT;
-- Session B (Same Transaction):
SELECT Balance FROM Bank_Account WHERE Acc_No = 102; -- Reads 45000.00 (Non-Repeatable Read!)
COMMIT;

-- [PREVENTION: REPEATABLE READ & SERIALIZABLE]
-- Session B:
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT Balance FROM Bank_Account WHERE Acc_No = 102; -- Snapshot fixed at 45000.00
```

## 5. Output

### 5.1 Output After Savepoint Partial Rollback & `COMMIT`

| Acc_No | Holder_Name | Initial_Balance | Post-Transaction `Balance` | Outcome Explanation |
| ---: | :--- | ---: | ---: | :--- |
| 101 | Aarav Sharma | ₹50,000.00 | **₹40,000.00** | First transfer (`-₹10,000`) committed; second transfer (`-₹25,000`) rolled back to `SAVEPOINT`. |
| 102 | Priya Nair | ₹30,000.00 | **₹40,000.00** | Received `+₹10,000` prior to `SAVEPOINT after_first_transfer`. |
| 103 | Rohan Deshmukh | ₹15,000.00 | **₹15,000.00** | Unchanged because second leg was undone by `ROLLBACK TO SAVEPOINT`. |

### 5.2 Concurrency Anomalies & Isolation Level Comparison Output

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Observed Two-Session Behavior in MySQL InnoDB |
| :--- | :--- | :--- | :--- | :--- |
| **`READ UNCOMMITTED`** | **Occurs** | **Occurs** | **Occurs** | Session B reads Session A's uncommitted `99999.00` balance before Session A rolls back. |
| **`READ COMMITTED`** | Prevented | **Occurs** | **Occurs** | Session B only sees committed data, so two `SELECT` calls in one transaction return `40000.00` then `45000.00`. |
| **`REPEATABLE READ`** *(Default)* | Prevented | Prevented | Prevented (MVCC / Next-Key Locks) | Session B maintains a consistent MVCC read snapshot from its first read (`45000.00`). |
| **`SERIALIZABLE`** | Prevented | Prevented | Prevented | Converts plain `SELECT` into `SELECT ... FOR SHARE`, blocking concurrent `UPDATE`/`INSERT` until commit. |
