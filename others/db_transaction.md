# Isolation and Locking of DB Transactions

- [Isolation and Locking of DB Transactions](#isolation-and-locking-of-db-transactions)
  - [Two Concurrent Transactions of Read-Update\_Writeback](#two-concurrent-transactions-of-read-update_writeback)
    - [The Short Answer](#the-short-answer)
    - [The Detailed Step-by-Step Breakdown](#the-detailed-step-by-step-breakdown)
    - [Blocking vs. Failing](#blocking-vs-failing)
    - [Why is this necessary? The Lost Update Problem](#why-is-this-necessary-the-lost-update-problem)
  - [Transaction Isolation Levels](#transaction-isolation-levels)
    - [The Core Problem: Lost Update Anomaly](#the-core-problem-lost-update-anomaly)
    - [1. Read Uncommitted](#1-read-uncommitted)
    - [2. Read Committed](#2-read-committed)
    - [3. Repeatable Read](#3-repeatable-read)
    - [4. Serializable](#4-serializable)
    - [Summary Table](#summary-table)
  - [Optimistic vs. Pessimistic Locking](#optimistic-vs-pessimistic-locking)
    - [1. Pessimistic Locking with `NOWAIT`](#1-pessimistic-locking-with-nowait)
    - [A Related Option: `SKIP LOCKED`](#a-related-option-skip-locked)
    - [2. Optimistic Locking](#2-optimistic-locking)
    - [Comparison](#comparison)

## Two Concurrent Transactions of Read-Update_Writeback

### The Short Answer

Yes, for the specific row with `id='ID'`, those two transactions are **effectively serialized**. The use of `SELECT ... FOR UPDATE` ensures that one transaction must fully complete its read-modify-write cycle on that row before the other can even begin its read.

### The Detailed Step-by-Step Breakdown

The key to this entire process is the `FOR UPDATE` clause. This is a form of **pessimistic locking**. It signals to the database, "I intend to update this row, so please lock it now to prevent anyone else from modifying it or locking it until my transaction is finished."

Let's trace the execution with Transaction 1 (T1) and Transaction 2 (T2). Assume the initial value of `counter` is 10.

1.  **T1 Starts:**
    * T1 executes: `SELECT counter AS val FROM T WHERE id='ID' FOR UPDATE;`
    * The database finds the row for `id='ID'`.
    * It places an **exclusive lock** on this specific row. This lock prevents other transactions from acquiring a conflicting lock (like another `FOR UPDATE` or a direct `UPDATE`/`DELETE`) on the same row.
    * T1 receives the value `val = 10`.

2.  **T2 Starts (Concurrently):**
    * T2 attempts to execute: `SELECT counter AS val FROM T WHERE id='ID' FOR UPDATE;`
    * The database finds the row for `id='ID'` but sees that T1 already holds an exclusive lock on it.
    * T2 cannot acquire its own lock. It enters a **wait state**. T2 is now **blocked**, pending the release of T1's lock.

3.  **T1 Continues:**
    * In the application layer, T1 calculates `val += 1`, so `val` is now 11.
    * T1 executes: `UPDATE T SET counter = 11 WHERE id='ID';`
    * This is allowed because T1 holds the lock. The counter in the database is now 11.
    * T1 issues a `COMMIT`. This makes the change permanent and, crucially, **releases the exclusive lock** on the row.

4.  **T2 Resumes:**
    * As soon as T1's lock is released, the database unblocks T2.
    * T2's `SELECT ... FOR UPDATE` statement can now proceed.
    * It acquires a new exclusive lock on the row `id='ID'`.
    * It reads the `counter` value. It will read the **committed value from T1**, so it gets `val = 11`.
    * In its application layer, T2 calculates `val += 1`, so `val` is now 12.
    * T2 executes: `UPDATE T SET counter = 12 WHERE id='ID';`
    * T2 issues a `COMMIT`, releasing its lock.

**Final Result:** The `counter` is correctly updated to 12. The "lost update" anomaly has been prevented.

### Blocking vs. Failing

You correctly identified the two possible outcomes for T2 when it hits the lock:

1.  **Block (Default Behavior):** This is the most common and expected behavior. The transaction simply waits until the lock is released.
2.  **Fail (Timeout):** Most database systems have a `lock_wait_timeout` setting. If a transaction is blocked for longer than this configured time (e.g., 50 seconds in MySQL's default), the database will give up, roll back the waiting transaction (T2), and return an error to the application (e.g., "Lock wait timeout exceeded; try restarting transaction"). This prevents indefinite waits that could freeze an application.

Some database systems also support a `NOWAIT` or `SKIP LOCKED` option on the `SELECT` statement, which tells the database to fail immediately if a lock cannot be acquired, rather than waiting at all.

### Why is this necessary? The Lost Update Problem

Without `FOR UPDATE`, here's what could happen:

1.  **T1** reads `counter` (value is 10).
2.  **T2** reads `counter` (value is also 10).
3.  **T1** calculates `10 + 1 = 11` and updates the row to 11. T1 commits.
4.  **T2** calculates `10 + 1 = 11` and updates the row to 11. T2 commits.

**Final Result:** The counter is 11. One of the increments has been "lost." The `SELECT ... FOR UPDATE` pattern is the classic solution to this problem.

## Transaction Isolation Levels

Removing the `FOR UPDATE` clause makes the outcome of the two concurrent transactions entirely dependent on the **transaction isolation level** configured in the database.

Without the explicit lock from `FOR UPDATE`, you are now in the world of standard concurrency control, where the database tries to balance performance and correctness according to its set rules. This can lead to a classic race condition known as the **Lost Update Anomaly**.

Let's analyze the behavior at each of the four standard SQL isolation levels. Assume the starting `counter` value is `10`.

---

### The Core Problem: Lost Update Anomaly

This is the race condition your scenario is designed to create. Here is the sequence of events that leads to the problem:

1.  **T1 Reads:** Transaction 1 executes `select counter...` and gets the value `10`.
2.  **T2 Reads:** Transaction 2 executes `select counter...` and also gets the value `10`.
3.  **T1 Writes:** T1 calculates `10 + 1 = 11` in its application logic and runs `update T with counter = 11`.
4.  **T2 Writes:** T2, which is completely unaware of T1's work, also calculates `10 + 1 = 11` and runs `update T with counter = 11`.

The final value in the database is `11`, even though two increments were performed. T1's update has been "lost" or overwritten by T2.

Now, let's see how the different isolation levels handle this.

### 1. Read Uncommitted

This is the lowest, most lenient level of isolation.

* **Behavior:** Transactions can read data that has been modified by other transactions but not yet committed ("dirty reads").
* **In Your Scenario:** The Lost Update anomaly can easily happen exactly as described above. T1 and T2 will both read `10` before either has a chance to write. One will write `11`, and the other will overwrite it with `11`.
* **Outcome:** **High probability of a Lost Update.** This level offers virtually no protection against this race condition.

### 2. Read Committed

This is the default isolation level for many databases (e.g., PostgreSQL, Oracle, SQL Server).

* **Behavior:** A transaction can only read data that has been committed. It prevents dirty reads. However, multiple reads of the same row within the *same* transaction are not guaranteed to return the same value if another transaction commits a change in between (this is called a "non-repeatable read").
* **In Your Scenario:** `Read Committed` **does not prevent the Lost Update anomaly**. The sequence described above is still the most likely outcome:
    1.  T1 reads the committed value `10`.
    2.  T2 reads the committed value `10`.
    3.  T1 updates the value to `11` and commits.
    4.  T2 updates the value to `11` (based on the `10` it originally read) and commits.
* **Outcome:** **Lost Update occurs.** The final value is `11`.

### 3. Repeatable Read

This is a stricter level, and the default for MySQL (InnoDB).

* **Behavior:** Guarantees that if a transaction reads a row, it will see the exact same data if it reads that row again later. It prevents dirty reads and non-repeatable reads. This is often implemented using Multi-Version Concurrency Control (MVCC), where each transaction operates on a consistent "snapshot" of the database as it existed when the transaction began.
* **In Your Scenario:** This is where things get interesting. The Lost Update is generally prevented, but not by blocking. It is prevented by **forcing one transaction to fail**.
    1.  T1 starts and its transaction snapshot sees `counter = 10`. T1 reads `10`.
    2.  T2 starts and its transaction snapshot also sees `counter = 10`. T2 reads `10`.
    3.  T1 updates the counter to `11` and commits successfully.
    4.  T2 now attempts to update the counter to `11`. When it tries to commit, the database's MVCC mechanism detects a conflict. T2 is trying to update a row that has been changed since its snapshot was taken.
* **Outcome:** The database will **roll back Transaction 2** and return a **serialization error**. The application for T2 would have to catch this error and retry the entire transaction (read-increment-update). The Lost Update is prevented, but at the cost of application-level complexity to handle retries.

### 4. Serializable

This is the highest, strictest level of isolation.

* **Behavior:** Guarantees that the result of running transactions concurrently is the same as if they were executed one after another in some sequential order. It prevents all race conditions, including Phantom Reads.
* **In Your Scenario:** The behavior is identical to `Repeatable Read`. The database will detect the write conflict when T2 attempts to commit and will roll back T2 with a serialization error, forcing the application to retry.
* **Outcome:** **Lost Update is prevented** by causing one of the conflicting transactions to fail.

### Summary Table

| Isolation Level | Behavior for Your Scenario | Does Lost Update Occur? |
| :--- | :--- | :--- |
| **Read Uncommitted** | Both transactions read the same initial value and the second write overwrites the first. | **Yes** |
| **Read Committed** | Both transactions read the same initial committed value and the second write overwrites the first. | **Yes** |
| **Repeatable Read** | Both transactions read from their own snapshot. When the second transaction tries to commit its update, the database detects a conflict and rolls it back with an error. | **No (but one transaction fails)** |
| **Serializable** | Same as Repeatable Read for this specific case. The database detects the write conflict and rolls back the second transaction to enforce serial execution. | **No (but one transaction fails)** |

In conclusion, relying on isolation levels like `Repeatable Read` or `Serializable` can prevent lost updates, but it does so by forcing one transaction to fail, which your application must be designed to handle. Using an explicit `SELECT ... FOR UPDATE` is a more direct and often clearer way to signal your intent to perform a safe "read-modify-write" operation, as it serializes the access through locking rather than failure.

## Optimistic vs. Pessimistic Locking

The default behavior of a locking statement like `SELECT ... FOR UPDATE` is to wait. To change this to immediately abort the transaction (or at least the statement) if a lock cannot be acquired, you need to use specific, non-standard SQL clauses that are supported by most major database systems.

The two most common approaches are using the `NOWAIT` clause or implementing an optimistic locking strategy.

### 1\. Pessimistic Locking with `NOWAIT`

This is the most direct way to achieve what you're asking for. You modify your `SELECT ... FOR UPDATE` statement to tell the database not to wait for the lock.

**How it Works:**

You add the `NOWAIT` keyword to your `SELECT` statement. When the statement is executed:

  * If the row(s) are **not** locked, your transaction acquires the lock and proceeds normally.
  * If the row(s) **are** locked by another transaction, the database does not wait. It immediately raises an error and aborts the statement. Your application code must catch this specific error to handle it gracefully.

**SQL Example (PostgreSQL / Oracle):**

```sql
-- This statement will fail instantly if the row is already locked.
SELECT counter AS val FROM T WHERE id='ID' FOR UPDATE NOWAIT;
```

**MySQL / MariaDB Example:**

MySQL uses a slightly different syntax with the same meaning.

```sql
-- This statement will fail instantly if the row is already locked.
SELECT counter AS val FROM T WHERE id='ID' FOR UPDATE NOWAIT;
```

**Application Logic Implementation (Python Pseudocode):**

Your application code is responsible for handling the error raised by the database.

```python
import database_driver

# Assume 'conn' is an active database connection
try:
    with conn.cursor() as cursor:
        # Start the transaction
        cursor.execute("BEGIN;")

        # Attempt to acquire the lock without waiting
        cursor.execute("SELECT counter FROM T WHERE id='ID' FOR UPDATE NOWAIT;")
        val = cursor.fetchone()[0]

        val += 1
        cursor.execute("UPDATE T SET counter = %s WHERE id='ID';", (val,))

        # If all successful, commit
        cursor.execute("COMMIT;")
        print("Update successful!")

except database_driver.errors.LockNotAvailable as e:
    # This is the specific error for "NOWAIT" failure. The exact exception class
    # depends on the database driver (e.g., psycopg2.errors.LockNotAvailable).
    print("Could not acquire lock, resource is busy. Aborting transaction.")
    conn.rollback() # Roll back the transaction

except Exception as e:
    # Handle other potential errors
    print(f"An unexpected error occurred: {e}")
    conn.rollback()

```

**Use Case:** This is perfect for interactive, user-facing operations where making a user wait for an unknown amount of time would be a poor experience. You can immediately return a "Try again later" message.

-----

### A Related Option: `SKIP LOCKED`

It's important to know about `SKIP LOCKED` as it solves a related, but different, problem.

  * `NOWAIT`: Tries to lock specific rows. If *any* of them are locked, the whole statement fails.
  * `SKIP LOCKED`: Tries to lock rows matching the `WHERE` clause. It locks all the ones it can and simply **ignores** the ones that are already locked by another transaction, returning only the unlocked rows. The statement doesn't fail.

**SQL Example:**

```sql
-- This statement will lock and return all unlocked rows with status 'pending'.
-- It will simply not include any rows that are already locked.
SELECT * FROM job_queue WHERE status='pending' LIMIT 10 FOR UPDATE SKIP LOCKED;
```

**Use Case:** This is the canonical pattern for building a **work queue**. Multiple worker processes can run this exact same query concurrently. Each worker will grab a different set of available jobs from the queue without blocking each other or failing.

-----
- [Isolation and Locking of DB Transactions](#isolation-and-locking-of-db-transactions)
  - [Two Concurrent Transactions of Read-Update\_Writeback](#two-concurrent-transactions-of-read-update_writeback)
    - [The Short Answer](#the-short-answer)
    - [The Detailed Step-by-Step Breakdown](#the-detailed-step-by-step-breakdown)
    - [Blocking vs. Failing](#blocking-vs-failing)
    - [Why is this necessary? The Lost Update Problem](#why-is-this-necessary-the-lost-update-problem)
  - [Transaction Isolation Levels](#transaction-isolation-levels)
    - [The Core Problem: Lost Update Anomaly](#the-core-problem-lost-update-anomaly)
    - [1. Read Uncommitted](#1-read-uncommitted)
    - [2. Read Committed](#2-read-committed)
    - [3. Repeatable Read](#3-repeatable-read)
    - [4. Serializable](#4-serializable)
    - [Summary Table](#summary-table)
  - [Optimistic vs. Pessimistic Locking](#optimistic-vs-pessimistic-locking)
    - [1. Pessimistic Locking with `NOWAIT`](#1-pessimistic-locking-with-nowait)
    - [A Related Option: `SKIP LOCKED`](#a-related-option-skip-locked)
    - [2. Optimistic Locking](#2-optimistic-locking)
    - [Comparison](#comparison)

### 2\. Optimistic Locking

This is a completely different strategy that avoids long-held database locks altogether and therefore never has to wait on a `SELECT`. The "locking" conflict is detected at the `UPDATE` stage.

**How it Works:**

1.  **Add a Version Column:** Add an integer `version` column (or a `last_updated` timestamp) to your table.
2.  **Read without Locking:** Perform a simple `SELECT` to get the current data, including the `version` number.
3.  **Perform Update with a Check:** When you write the data back, your `UPDATE` statement's `WHERE` clause must check that the `version` is still the same as the one you originally read. You also increment the version number in the `SET` clause.
4.  **Check Affected Rows:** After the `UPDATE`, check how many rows were affected.
      * If **1 row** was affected, your update was successful.
      * If **0 rows** were affected, it means another transaction updated the row (and thus incremented the version) between your `SELECT` and `UPDATE`. Your transaction has lost the race.

**Implementation Example:**

```sql
-- Step 1: Read the data (no lock)
-- Assume T1 reads and gets val=10, version=5
SELECT counter, version FROM T WHERE id='ID';

-- (Application logic: val += 1)

-- Step 2: Attempt the atomic update
-- T1 tries to write back its change
UPDATE T
SET
    counter = 11,
    version = version + 1 -- Increment the version
WHERE
    id = 'ID' AND version = 5; -- Check the original version
```

**Application Logic:** The application must check the result of the `UPDATE` statement. If the row count is 0, it means a conflict occurred, and the application must decide whether to abort and inform the user, or to re-run the entire transaction (read again, calculate again, try to update again).

### Comparison

| Method | How it Works | Database Behavior | Best Use Case |
| :--- | :--- | :--- | :--- |
| **`... FOR UPDATE NOWAIT`** | Pessimistic Locking. Explicitly tells the DB not to wait for a lock. | Throws an immediate error if the row is locked. | User-facing actions where a wait is unacceptable. "Fail-fast" logic. |
| **`... FOR UPDATE SKIP LOCKED`** | Pessimistic Locking. Tells the DB to ignore any locked rows. | Returns a subset of rows that could be locked. Does not error. | Job queues, where concurrent workers need to claim distinct items to process. |
| **Optimistic Locking** | No DB locks on read. Adds a `version` check to the `UPDATE` statement. | `UPDATE` affects 0 rows if data has changed. No blocking. | High-throughput systems with low contention, where locking every read would be a bottleneck. |