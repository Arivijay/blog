# Read Committed Isolation vs Serializable in CockroachDB

> :warning: **Disclaimer**: This document is a work in progress and has not yet been peer-reviewed. The information presented here is subject to change and should not be considered complete or final. Reader discretion is advised.



<a id="toc"></a>
## Table to contents
- [2 Major Drawbacks](https://github.com/Arivijay/blog/edit/crdb/CRDBReadCommitted.md#drawbacks)
- [Setting Up Read Committed Isolation in CockroachDB](https://github.com/Arivijay/blog/edit/crdb/CRDBReadCommitted.md#setting-up-read-committed-isolation-in-cockroachdb)
- 4 Major Distinctions: RC vs Serializable
  - [Transaction Model](https://github.com/Arivijay/blog/edit/crdb/CRDBReadCommitted.md#transaction-model)
  - [Locking Behavior](https://github.com/Arivijay/blog/edit/crdb/CRDBReadCommitted.md#locking-behavior)
  - [Isolation Levels and Their Guarantees](https://github.com/Arivijay/blog/edit/crdb/CRDBReadCommitted.md#isolation-levels-and-their-guarantees)
  - [Retry Management](https://github.com/Arivijay/blog/edit/crdb/CRDBReadCommitted.md#retry-management)
- [Tests and Observations](https://github.com/Arivijay/blog/edit/crdb/CRDBReadCommitted.md#tests-and-observations)



## Drawbacks

### Risk of Data Integrity Issues:
- There is an increased risk of the application behaving in unexpected ways, potentially leading to data integrity issues.
### Greater Developer Burden: 
- App Devs need a deeper understanding of potential conflicts and must implement additional logic to handle these cases, which can be error-prone and require rigorous testing.

[Back to Table of Contents](#toc)


## Setting Up Read Committed Isolation in CockroachDB

This guide provides steps to configure the Read Committed isolation level in CockroachDB at different scopes: cluster level, session level, transaction level, and user level.
Cluster setting is a prerequisite for enabling the Read Committed isolation level, but it's not sufficient on its own. This settings simply allows the database to recognize the Read Committed isolation level syntax.

### Prerequisite: Enabling Read Committed Isolation at Cluster Level

Enable the Read Committed isolation at the cluster level:

```sql
SET CLUSTER SETTING sql.txn.read_committed_isolation.enabled = true;
```
Verify the setting:
```sql
SHOW CLUSTER SETTING sql.txn.read_committed_isolation.enabled;
```
### Option 1: Session-Level Configuration
Set the default isolation level for future transactions in the current session:
```sql
SET default_transaction_isolation = 'read committed';
```
Quick verification:
```sql
SHOW default_transaction_isolation;
```
### Option 2: Transaction-Level Configuration
Immediate setting:
```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
```
After beginning a transaction:
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
Alternative syntax:
```sql
SET transaction_isolation = 'read committed';
```
Quick verification:
```sql
SHOW transaction_isolation;
```
### Option 4: User-Level Configuration
```sql
ALTER ROLE rc_user SET default_transaction_isolation = 'read committed';
```
Quick verification:
```sql
SHOW default_transaction_isolation;
```
[Back to Table of Contents](#toc)

## Transaction Model

| Isolation Level            | Scope of Read Snapshot     |
|----------------------------|----------------------------|
| Serializable               |   Per-Transaction          |
| Read Committed             |   Per-Statement            |


### Illustration 

|         --  setup                  |
| ------------------------------------------ |
| DROP   TABLE IF EXISTS  t;                 |
| CREATE TABLE t (k INT PRIMARY KEY, v INT); |
| INSERT INTO t VALUES (1,1),(2,2),(3,3);    |

#### SERIALIZABLE Isolation: Multi-Statement Transaction Example
Let's walk through a transaction in Serializable isolation to observe the behavior of read timestamps:

|   Multi-Statement  Transaction      | 
|----------------------------|
| BEGIN;                     |
| select * from t where k=1; |
|  `Note the statement read timestamp (rts):1702758697.393337437, as retrieved from the statement bundle.` |
|  `Let's pause here for a moment, take a sip of water, and perform an update`|
| UPDATE t SET v=32 WHERE k=1;|
|  `Note the statement read timestamp (rts):1702758697.393337437, as retrieved from the statement bundle.` |
|  `Despite the pause, observe that the statement read timestamp remains unchanged`|
|  `Let's pause here again, then perform a SELECT`|
| select * from t where k=1; |
|  `Statement rts=1702758697.393337437 - Still hasn't changed`                |
| COMMIT;                    |

**So what happeneded here:**  Transaction maintains a consistent view of the database throughout the transaction's duration, in Serializable isolation.

---

#### READ COMMITED Isolation: Multi-Statement Transaction Example
Now let's walk through the same transaction in RC  isolation to observe the behavior of read timestamps:

|  Multi-Statement  Transaction       | 
|----------------------------|
| BEGIN;                     |
| select * from t where k=1; |
|  `Note the statement read timestamp (rts): 1702759180.206868720 as retrieved from the statement bundle.`                |
|  `Let's pause here for a moment, take another sip of water, and perform an update`|
| UPDATE t SET v=32 WHERE k=1;|
|  `Statement rts=1702759199.596345416`                |
|  ` ^^ Statement read stamp advanced`                |
|  `Let's pause here again, then perform a SELECT`|
| select * from t where k=1; |
|  `Statement rts=1702759216.784114568`                |
|  ` ^^ Statement read stamp advanced again`                |
| `kv.Txn 's ReadTimestamp to hlc.Now() on each statement boundary`|
| COMMIT;                    |

**So what happeneded here:** Each statement within a transaction sees the database as of a unique point in time, in Read Committed isolation. This means the view of the database can change within the transaction, as it operates over multiple read snapshots.

---

### Advantages and Implementation of the Per-Statement Snapshot Model in Read Committed Isolation
This section explores what we gain from implementing the Per-Statement Snapshot Model in Read Committed isolation and how it is implemented:

 - Advantages:
    - Each statement in a Read Committed transaction observes a unique read snapshot, allowing errors to be handled at the statement level rather than at the transaction level.
    - This facilitates statement retries with a new read snapshot, reducing client involvement, similar to the behavior in implicit transactions.
 - Implementation of Statement-Level Retries:
    - A retry loop is used for executing each SQL statement, catching and handling errors internally reducing client involvement.
    - Transaction savepoints are established at the start of each statement. They are released after successful execution or rolled back in case of errors.
    - For statements that stream results to the client, transparent retries are limited. Result sets are buffered up to 16KiB before streaming, which confines the extent of transparent retries.

### Considerations in the Per-Statement Read Snapshot Model
While the Per-Statement Snapshot Model in Read Committed isolation offers several benefits, there are important considerations to keep in mind, particularly regarding isolation guarantees. For more details on these considerations, refer to [ Isolation Levels and Their Guarantees](https://github.com/cockroachlabs/CRDB-PlatformHealthCheck/blob/main/CRDBReadCommitted.md#isolation-levels-and-their-guarantees)


[Back to Table of Contents](#toc)

## Locking Behavior
  
### Illustration

|         --  setup                  |
| ------------------------------------------ |
| DROP   TABLE IF EXISTS  t;                 |
| CREATE TABLE t (k INT PRIMARY KEY, v INT); |
| INSERT INTO t VALUES (1,1),(2,2),(3,3);    |


#### Here, we have three concurrent transactions involving reads and a write. Notice how the transactions interact under both Serializable and RC isolations.

| Transaction 1 (read) | Transaction 2 (read)       | Transaction 3 (write)        |
| -------------------- | -------------------------- | ---------------------------- |
| BEGIN;               |                            |                              |
| SELECT * FROM t;     | BEGIN;                     |                              |
|                      | SELECT * FROM t WHERE k=2; | BEGIN;                       |
|                      |                            | UPDATE t SET v=12 WHERE k=2; |
|                      |                            | COMMIT;                      |
|                      | COMMIT;                    | `success`                    |
| COMMIT;              | `success`                  |                              |
| `success`            |                            |                              |

**Observation:**  In both isolations, Reads DO NOT BLOCK reads & Reads DO NOT BLOCK writes

---

####  Now let's consider a scenario under Serializable Isolation. Observe how the write transaction influences the other transactions.

| Transaction 1 (write)          | Transaction 2 (read)       | Transaction 3 (write)           |
| ------------------------------ | -------------------------- | ------------------------------- |
| BEGIN;                         |                            |                                 |
| UPDATE t SET v=12 WHERE k=2;   | BEGIN;                     |                                 |
| `lock k=2`                     | SELECT * FROM t WHERE k=2; | BEGIN;                          |
|                                | `waiting...`               | UPDATE t SET v=22  WHERE k=2;   |
|                                |                            | `waiting...`                    |
| COMMIT;                        | `unblocked to proceed...`  | `unblocked to proceed...`       |
| `success`                      | COMMIT;                    | COMMIT;                         |
|                                | `success`                  | `success, k=2,v=22`             |

**Observation:** In Serializable, Writes BLOCK reads as well as writes

---

#### Now let's see the same transaction pattern under Read Committed Isolation. Notice the differences in how the write transaction impacts the reads and other writes.

| Transaction 1 (write)          | Transaction 2 (read)       | Transaction 3 (write)           |
| ------------------------------ | -------------------------- | ------------------------------- |
| BEGIN;                         |                            |                                 |
| UPDATE t SET v=32 WHERE k=2;   | BEGIN;                     |                                 |
| `lock k=2`                     | SELECT * FROM t WHERE k=2; | BEGIN;                          |
|                                | `proceeds,k=2,v=22`        | UPDATE t SET v=42  WHERE k=2;   |
|                                |  COMMIT;                   | `waiting...`                    |
| COMMIT;                        | `success,k=2,v=22`         | `unblocked to proceed...`       |
| `success, k=2,v=32`            |                            | COMMIT;                         |
|                                |                            | `success, k=2,v=42`             |

**Observation:** In RC, Writes BLOCK writes but not reads

---

### To Sum-up:
### In Serializable 
- Reads DO NOT BLOCK reads
- Reads DO NOT BLOCK writes
- Writes BLOCK writes
- Writes BLOCK current reads
### In Read Committed 
- Reads DO NOT BLOCK reads
- Reads DO NOT BLOCK writes
- Writes BLOCK writes
- **HOWEVER** Writes DO NOT BLOCK current reads

[Back to Table of Contents](#toc)

## Isolation Levels and Their Guarantees

| Isolation Level      | Dirty read     | Non-repeatable read | Phantom Read   | Write Skew     |
|----------------------|:--------------:|:-------------------:|:--------------:|:--------------:|
| READ UNCOMMITTED     | ❌ Possible    | ❌ Possible         | ❌ Possible    | ❌ Possible    |
| READ COMMITTED       | ✅ Not Possible | ❌ Possible         | ❌ Possible    | ❌ Possible    |
| REPEATABLE READ      | ✅ Not Possible | ✅ Not Possible     | ❌ Possible    | ❌ Possible    |
| SNAPSHOT ISOLATION   | ✅ Not Possible | ✅ Not Possible     | ✅ Not Possible | ❌ Possible    |
| SERIALIZABLE         | ✅ Not Possible | ✅ Not Possible     | ✅ Not Possible | ✅ Not Possible |

### Illustration

|         --  setup                  |
| ------------------------------------------ |
| DROP   TABLE IF EXISTS  t;                 |
| CREATE TABLE t (k INT PRIMARY KEY, v INT); |
| INSERT INTO t VALUES (1,1),(2,2),(3,3);    |


#### Non-Repeatable Read
A non-repeatable read occurs when a transaction reads the same row twice and gets different results because another transaction modifies and commits that row in between the reads.

| Transaction 1 (Read)       | Transaction 2 (Write)      |
|----------------------------|----------------------------|
| BEGIN;                     |                            |
| SELECT * FROM t WHERE k=2; | BEGIN;                     |
| `Sees v=2`         | UPDATE t SET v=22 WHERE k=2; |
|                            | COMMIT;                    |
| SELECT * FROM t WHERE k=2; |                            |
| `Now sees v=22`        |                            |
| COMMIT;                    |                            |

---

#### Phantom Read
A phantom read occurs when a transaction reads a set of rows that satisfy a condition, then another transaction inserts or deletes rows that would change the original set, and the first transaction reads the set again and gets different results.
|         -- Reset to Initial State          |
| ------------------------------------------ |
| truncate TABLE t;                          |
| INSERT INTO t VALUES (1,1),(2,2),(3,3);    |

| Transaction 1 (Read Range) | Transaction 2 (Insert)     |
|----------------------------|----------------------------|
| BEGIN;                     |                            |
| SELECT * FROM t WHERE v <= 3; | BEGIN;                     |
|  `Range Query: Outputs rows with v=1, v =2, v=3`                           | INSERT INTO t VALUES (4, 3); |
|                            | COMMIT;                    |
| SELECT * FROM t WHERE v <= 3; |                            |
|  `Different Result Set`                    |                            |
|  `Now outputs an additional row with v=3`                    |                            |
|COMMIT; |                        |

---

#### Write Skew
Write skew occurs when two transactions concurrently read and then update overlapping data based on their initial reads, leading to a logical inconsistency.

|         --  setup                  |
| ------------------------------------------ |
| DROP TABLE IF EXISTS meeting_rooms;               |
|CREATE TABLE meeting_rooms ( room_id INT PRIMARY KEY, room_availability BOOLEAN, emp_id INT);|
| INSERT INTO meeting_rooms (room_id, room_availability,emp_id) VALUES (1, true,NULL),(2, false,NULL);|
| `-- Room 1 is available -- Room 2 is already booked` |

| Transaction 1 (Employee A)                                      | Transaction 2 (Employee B)                                      |
|------------------------------------------------------------------|------------------------------------------------------------------|
| `BEGIN;`                                                         | `BEGIN;`                                                         |
| -- Employee A checks for an available meeting room               | -- Employee B checks for an available meeting room               |
| `SELECT * FROM meeting_rooms WHERE room_availability = true;`    | `SELECT * FROM meeting_rooms WHERE room_availability = true;`    |
| -- Finds room_id 1 available                                     | -- Finds room_id 1 available                                     |
| `UPDATE meeting_rooms SET room_availability = false,emp_id = 1 WHERE room_id = 1;` |                                                                  |
| `COMMIT;`                                                        |                                                                  |
|                                                                  | `UPDATE meeting_rooms SET room_availability = false,emp_id = 2 WHERE room_id = 1;` |
|                                                                  | `COMMIT;`                                                        |


[Back to Table of Contents](#toc)

## Retry Management

### RC vs Serializable 
1. No Serializable Isolation-Related Retries in RC
2. Reduced Client Side Retries during Consistency-Related and Esoteric Situations
3. Not all 40001 errors, especially those related to lease transfers, are resolved solely by RC

### 1. No Serializable Isolation-Related Retries in RC

### Illustration 1

|         --  setup                  |
| ------------------------------------------ |
| DROP   TABLE IF EXISTS  t;                 |
| CREATE TABLE t (k INT PRIMARY KEY, v INT); |
| INSERT INTO t VALUES (1,1),(2,2),(3,3);    |

#### SERIALIZABLE Isolation vs. Read Committed Isolation

| TXN 1 (multi-statement)               | TXN 2 (multi-statement)    | - | TXN 1 (multi-statement)               | TXN 2 (multi-statement)       |
|---------------------------------------|----------------------------|---|---------------------------------------|----------------------------|
| BEGIN;                                |                            | - | BEGIN;                                |                            |
| SELECT * FROM t;                      |                            | - | SELECT * FROM t;                      |                            |
|                                       | BEGIN;                     | - |                                       | BEGIN;                     |
|                                       | SELECT * FROM t;           | - |                                       | SELECT * FROM t;           |
| UPDATE t SET v=222 WHERE k=2;         |                            | - | UPDATE t SET v=222 WHERE k=2;         |                            |
|                                       | UPDATE t SET v=333 WHERE k=3; | - |                                    | UPDATE t SET v=333 WHERE k=3; |
| COMMIT;                               |                            | - | COMMIT;                               |                            |
| `success`                             | COMMIT;                    | - | `success`                            | COMMIT;                  |
|                            | `Error 40001: RETRY_SERIALIZABLE`                  | - |                              | `success`                  |
|                            | `rollback`                  | - |                            |                  |

Complete ERROR: restart transaction: TransactionRetryWithProtoRefreshError: TransactionRetryError: retry txn (RETRY_SERIALIZABLE  - failed preemptive refresh due to conflicting locks

### Illustration 2

|         -- Reset to Initial State          |
| ------------------------------------------ |
| truncate TABLE t;                          |
| INSERT INTO t VALUES (1,1),(2,2),(3,3);    |

#### SERIALIZABLE Isolation vs. Read Committed Isolation

| TXN 1 (multi-statement)              | TXN 2 (Update)             | - | TXN 1 (multi-statement)              | TXN 2 (Update)      |
|--------------------------------------|----------------------------| - |--------------------------------------|----------------------------|
| BEGIN;                               | BEGIN;                     | - | BEGIN;                               | BEGIN;                     |
| select * from t where k=1;           |                            | - | select * from t where k=1;           |                            |
|                                      | UPDATE t SET v=22 WHERE k=1;| - |                                     | UPDATE t SET v=22 WHERE k=1;|
|                                      | COMMIT;                    | - |                                      | COMMIT;                    |
| UPDATE t SET v=32 WHERE k=1;         |  `success`                 | - | UPDATE t SET v=32 WHERE k=1;         |  `success`                 |
| COMMIT;                              |                            | - | COMMIT;                              |                            |
| `Error 40001: RETRY_SERIALIZABLE`    |                            | - | `success`                            |                            |
| `rollback`                           |                            | - |                                      |                            |

Complete ERROR: restart transaction: TransactionRetryWithProtoRefreshError: WriteTooOldError:

 **Observation:** By tolerating specific anomalies, Read Committed isolation avoids Isolation-related retries that are common under SERIALIZABLE isolation.

 ---
 
###  2. Reduced Client Side Retries during Consistency-Related and Esoteric Situations

#### Illustration - Closed Timestamp related error
|         -- Reset to Initial State          |
| ------------------------------------------ |
| truncate TABLE t;                          |
| INSERT INTO t VALUES (1,1),(2,2),(3,3);    |

#### SERIALIZABLE Isolation vs. Read Committed Isolation

| TXN 1 (multi-statement)                                           | TXN 2 (write, implicit)                                    | - | TXN 1 (multi-statement)                                           | TXN 2 (write, implicit)                                    |
|-------------------------------------------------------------------|------------------------------------------------------------| - |-------------------------------------------------------------------|------------------------------------------------------------|
| SHOW CLUSTER SETTING kv.closed_timestamp.target_duration;         |                                                            | - | SHOW CLUSTER SETTING kv.closed_timestamp.target_duration;         |                                                            |
| `Default closed timestamp period is 3 seconds`                  |                                                            | - | `Default closed timestamp period is 3 seconds`                  |                                                            |
| BEGIN;                                                            |                                                            | - | BEGIN;                                                            |                                                            |
| SELECT * FROM t;                                                  |                                                            | - | SELECT * FROM t;                                                  |                                                            |
|                                                                   | UPDATE t SET v=333 WHERE k=3;    `-- lock k=3`             | - |                                                                   | UPDATE t SET v=333 WHERE k=3;    `-- lock k=3`             |
| `Interactively switching windows and typing takes a user more than 3 seconds` |                                                            | - | `Interactively switching windows and typing takes a user more than 3 seconds` |                                                            |
| UPDATE t SET v=222 WHERE k=2;   `-- lock k=2`                     |                                                            | - | UPDATE t SET v=222 WHERE k=2;   `-- lock k=2`                     |                                                            |
| COMMIT;                                                           |                                                            | - | COMMIT;                                                           |                                                            |
| `Error 40001: RETRY_SERIALIZABLE - failed preemptive refresh (on commit)` |  | - | `Success` |  |
| `rollback`                                    |                                                            | - |                                                         |                    
| `client retry`                        |                                                           | - |                                                   |       

**Observation:** By managing errors at the statement level instead of the entire transaction level, we effectively reduce the number of retries needed in both esoteric and consistency-related situations. It's important to note, though, that this illustration doesn't demonstrate the latter. 

---

###  3. Not all 40001 errors, especially those related to lease transfers, are resolved solely by RC

#### Illustration - TX encounters 40001 with RC when lease transfers

| Transaction  (multi-statement)       |
|----------------------------|
| BEGIN;                     |                          
|  SELECT * FROM t where k=1;|
| `Relocate Lease Holder to another node`    |
| UPDATE t SET v=222 WHERE k=2; -- lock k=2 |
| COMMIT;                    |
| `restart transaction: ABORT_REASON_CLIENT_REJECT:40001`|

  
[Back to Table of Contents](#toc)

## Tests and Observations

### Tool used: [Contender](https://github.com/cockroachlabs/contender)
Uses simple key-value table to demo the effects of contention
```sql
CREATE TABLE IF NOT EXISTS contend (
    id UUID NOT NULL PRIMARY KEY DEFAULT gen_random_uuid(),
    value INT NOT NULL DEFAULT 0
)
```
Each worker executes the following SQL:
```sql
BEGIN;
SELECT value FROM contend WHERE id = $1;  -- Optional ‘FOR UPDATE’
UPDATE contend SET value = $1 WHERE id = $2;
COMMIT;
```

### Here's an example to help clarify the use of contender flags:
###  Suppose you have a contender command like this:
 `contender -conn "postgresql://xxx" -uniqueIds 10 -workersPerId 1 -thinkTime 0s`

![image](https://github.com/cockroachlabs/CRDB-PlatformHealthCheck/assets/89301858/181d38d4-2003-4b07-821a-71c77250d849)


### Suppose you have a contender command like this:
`contender -conn "postgresql://xxx" -uniqueIds 10 -workersPerId 2 -thinkTime 0s`

![image](https://github.com/cockroachlabs/CRDB-PlatformHealthCheck/assets/89301858/1daa3833-95fd-439c-b98a-b63b09b714dd)

### Observations
### RC Boosted Throughput (although data anomalies anticipated). Thanks to the absence of serializable errors, there were no transaction aborts, leading to zero client retries.
### Serialiable vs RC
![image](https://github.com/cockroachlabs/CRDB-PlatformHealthCheck/assets/89301858/f747e279-4e31-4554-8c66-67c257f9549e)

### Serialiable vs Serialiable with SFU vs RC
![image](https://github.com/cockroachlabs/CRDB-PlatformHealthCheck/assets/89301858/41363ed3-3b8d-4df6-92a4-b030d0c870f7)

[Back to Table of Contents](#toc)



