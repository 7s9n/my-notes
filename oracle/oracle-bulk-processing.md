---
title: Deep Dive into BULK COLLECT, FORALL, LIMIT, and SAVE EXCEPTIONS
tags:
  - oracle
  - plsql
  - performance
  - bulk-processing
  - etl
  - database
created: 2026-09-23
---

# Deep Dive into BULK COLLECT, FORALL, LIMIT, and SAVE EXCEPTIONS

In Oracle Database, bulk processing features are designed to solve one major performance issue: context switching between the PL/SQL engine and SQL engine. When processing large datasets row-by-row, performance degrades due to repeated engine communication. These four features work together to optimize speed, memory usage, and reliability.

To solve this, Oracle introduced bulk processing features:

- **BULK COLLECT**
- **FORALL**
- **LIMIT clause**
- **SAVE EXCEPTIONS**

Together, these four concepts form the backbone of high-performance batch processing in PL/SQL.

Let’s break them down in depth.

---

## 1️⃣ BULK COLLECT — High-Speed Data Fetching

### 🔹 What Problem Does It Solve?
By default, PL/SQL fetches rows one at a time. Every time a SQL statement executes inside PL/SQL, Oracle switches between:

**PL/SQL Engine ↔ SQL Engine**

If you’re fetching 10,000 rows, that means 10,000 context switches.  
That is expensive.

### 🔹 What BULK COLLECT Does
BULK COLLECT fetches multiple rows in a single context switch and stores them into a PL/SQL collection.  
Instead of row-by-row fetching, it performs array-based fetching.

### 🔹 Code Example
```plsql
DECLARE
   TYPE emp_tab IS TABLE OF employees%ROWTYPE;
   l_emps emp_tab;
BEGIN
   SELECT *
   BULK COLLECT INTO l_emps
   FROM employees;

   DBMS_OUTPUT.PUT_LINE('Rows fetched: ' || l_emps.COUNT);
END;
```
### Functional Explanation (Detailed)
When this query executes, Oracle fetches all rows from the employees table at once and stores them into the PL/SQL collection l_emps. This drastically reduces context switching because instead of calling the SQL engine repeatedly for each row, the SQL engine returns a full result set in one call.

However, this introduces a new risk: memory usage. All rows are loaded into PGA memory. If the dataset is very large (say 1 million rows), this can cause excessive PGA consumption or even memory errors like ORA-04030.

So while BULK COLLECT improves speed, it must be used carefully for large datasets. That’s where the LIMIT clause comes in.

## 2️⃣ LIMIT Clause — Memory Control Mechanism

### 🔹 Why LIMIT Is Necessary

If BULK COLLECT loads everything at once, it can exhaust memory. The LIMIT clause allows you to fetch rows in batches.

Instead of loading 1,000,000 rows at once, you load 1,000 at a time.

### 🔹 Code Example

```plsql
DECLARE
   CURSOR c_emp IS SELECT * FROM employees;
   TYPE emp_tab IS TABLE OF employees%ROWTYPE;
   l_emps emp_tab;
BEGIN
   OPEN c_emp;

LOOP
      FETCH c_emp BULK COLLECT INTO l_emps LIMIT 1000;
      EXIT WHEN l_emps.COUNT = 0;
      DBMS_OUTPUT.PUT_LINE('Processing batch of ' || l_emps.COUNT);
   END LOOP;
   CLOSE c_emp;
END;
```

### 🔹 Functional Explanation (Detailed)
Here, instead of fetching all rows in one shot, Oracle fetches only 1000 rows per iteration. Once those rows are processed, the loop fetches the next batch.

This approach gives you:
- Controlled memory usage
- Stable performance
- No risk of memory overflow
- Better scalability for large tables

In production systems, choosing the right LIMIT value is important. Typical values range from 500 to 5000 depending on system memory and workload.

LIMIT is what makes bulk processing safe for large-scale enterprise batch jobs.

## 3️⃣ FORALL — High-Speed Bulk DML

### 🔹 The Problem with Row-by-Row DML
Consider this:

```plsql
FOR i IN 1..l_emps.COUNT LOOP
   UPDATE employees
   SET salary = salary * 1.1
   WHERE employee_id = l_emps(i).employee_id;
END LOOP;
```

Even though data was bulk fetched, this still executes UPDATE row-by-row.
That means multiple context switches again.

### 🔹 What FORALL Does
`FORALL` sends all DML statements to the SQL engine in one batch.

### 🔹 Code Example
```plsql
FORALL i IN 1..l_emps.COUNT  
   UPDATE employees  
   SET salary = salary * 1.1  
   WHERE employee_id = l_emps(i).employee_id;
```

### 🔹 Functional Explanation (Detailed)
FORALL does not loop like a normal FOR loop. Instead, it binds an entire collection to a single SQL statement and sends it to the SQL engine in one call.

Internally, Oracle performs array binding. This eliminates repetitive engine switching and dramatically improves performance.

Performance improvements of 10x to 100x are common when replacing row-by-row DML with FORALL.

However, there is a drawback: if one row fails, the entire batch fails.

That leads us to SAVE EXCEPTIONS.

> FORALL is used to perform bulk DML operations such as INSERT, UPDATE, or DELETE using a collection. Unlike a regular FOR loop, FORALL does not execute the DML statement repeatedly row by row. Instead, it sends the entire batch of bind variables to the SQL engine in one call using array binding. This drastically reduces context switching and significantly improves DML performance. In high-volume systems, replacing row-by-row DML with FORALL can improve performance by multiple factors. However, FORALL assumes that the collection indexes are sequential unless special clauses like INDICES OF or VALUES OF are used. It is particularly powerful in ETL processes, data migrations, and mass updates where performance is critical. FORALL transforms PL/SQL from procedural row-based execution to set-based optimized execution.

## 4️⃣ SAVE EXCEPTIONS — Partial Failure Handling
### 🔹 The Problem
Without SAVE EXCEPTIONS:
If one row fails in FORALL → Entire operation fails.
In batch systems, we usually want:
- Valid rows → succeed
- Invalid rows → log error
- Processing → continue

### 🔹 Code Example
```plsql
BEGIN  
   FORALL i IN 1..l_emps.COUNT SAVE EXCEPTIONS  
      UPDATE employees  
      SET salary = salary * 1.1  
      WHERE employee_id = l_emps(i).employee_id;  
  
EXCEPTION  
   WHEN OTHERS THEN  
      FOR j IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP  
         DBMS_OUTPUT.PUT_LINE(  
            'Error Index: ' ||  
            SQL%BULK_EXCEPTIONS(j).ERROR_INDEX ||  
            ' Error Code: ' ||  
            SQL%BULK_EXCEPTIONS(j).ERROR_CODE  
         );  
      END LOOP;  
END;
```
### 🔹 Functional Explanation (Detailed)
When SAVE EXCEPTIONS is used, Oracle attempts to execute all DML operations even if some rows fail. Instead of stopping execution, it records errors in the system collection:

```
SQL%BULK_EXCEPTIONS
```

This special collection stores:
- ERROR_INDEX → Which row failed
- ERROR_CODE → What error occurred
This allows enterprise systems to:
- Log failed records
- Continue processing valid rows
- Maintain high availability
- Avoid full batch rollback

SAVE EXCEPTIONS is extremely important in ETL, migration, and batch update systems where partial failure is expected.

> By default, if any single DML statement inside FORALL fails, Oracle stops execution and raises an exception, causing the entire batch to fail. SAVE EXCEPTIONS changes this behavior by allowing Oracle to continue executing all remaining DML statements even if some rows fail. Instead of halting immediately, Oracle records details of failed operations in the SQL%BULK_EXCEPTIONS collection. This enables developers to analyze which rows failed and why, without rolling back successful operations unnecessarily. SAVE EXCEPTIONS is extremely important in enterprise batch systems where partial failures are expected due to data inconsistencies. It improves robustness and fault tolerance in large-scale data processing jobs. Using SAVE EXCEPTIONS allows a system to process valid records while logging and isolating problematic ones, ensuring operational continuity.

## 🔥 Combined Enterprise Pattern
In real-world batch systems, all four are used together:
```plsql
DECLARE  
   CURSOR c_emp IS SELECT * FROM employees;  
   TYPE emp_tab IS TABLE OF employees%ROWTYPE;  
   l_emps emp_tab;  
BEGIN  
   OPEN c_emp;  
  
LOOP  
      FETCH c_emp BULK COLLECT INTO l_emps LIMIT 1000;  
      EXIT WHEN l_emps.COUNT = 0;  
      FORALL i IN 1..l_emps.COUNT SAVE EXCEPTIONS  
         UPDATE employees  
         SET salary = salary * 1.1  
         WHERE employee_id = l_emps(i).employee_id;  
      COMMIT;  
   END LOOP;  
   CLOSE c_emp;  
END;
```

This approach ensures:
- Minimal context switching
- Controlled memory usage
- High DML performance
- Safe partial error handling
- Stable large-scale execution
## 🧠 Summary —
> When combined, these four features create a highly optimized and scalable batch-processing mechanism. BULK COLLECT retrieves data efficiently in batches, LIMIT ensures controlled memory consumption, FORALL executes DML operations in bulk with minimal context switching, and SAVE EXCEPTIONS provides fault tolerance for partial failures. Together, they transform inefficient row-by-row processing into high-performance, enterprise-grade execution. This pattern is commonly used in large ETL jobs, archival systems, payroll updates, and migration scripts. It balances performance, memory management, and reliability in a structured way. Senior PL/SQL developers must understand not just how to use these features, but also when to apply them appropriately. Proper implementation can reduce execution time drastically while maintaining system stability.

1. In Oracle Database PL/SQL, bulk processing is designed to reduce expensive context switching between the PL/SQL and SQL engines.
2. BULK COLLECT retrieves multiple rows at once into a collection instead of fetching row-by-row.
3. This significantly improves data retrieval performance but increases PGA memory usage.
4. The LIMIT clause controls how many rows are fetched per batch to prevent memory overflow.
5. LIMIT ensures scalable and stable execution for large datasets in production systems.
6. FORALL performs bulk INSERT, UPDATE, or DELETE using array binding instead of row-by-row DML.
7. It dramatically reduces execution time for mass data modifications.
8. By default, a single failure in FORALL stops the entire batch.
9. SAVE EXCEPTIONS allows processing to continue and stores failed row details in SQL%BULK_EXCEPTIONS.
10. Together, these four features form a high-performance, memory-efficient, and fault-tolerant PL/SQL batch processing architecture.
