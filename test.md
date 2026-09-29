# Testing Dynamic Range Partitioning for Oracle Copy in Microsoft Fabric

This test demonstrates how Microsoft Fabric Data Factory can split a single Oracle table into multiple range-based queries during a Copy activity. The test uses `FABRIC_TEST.FACT_SALES` and `SALES_ID` as the dynamic-range partition column.

## Test scenario

The Oracle source table contains 10,000,000 rows with `SALES_ID` values from 1 through 10,000,000.

The goal is to configure one Fabric Copy activity so that Fabric reads the table using multiple `SALES_ID` ranges rather than issuing only one unfiltered `SELECT *`.

The observed Fabric-generated ranges in this test were:

| Partition | SALES_ID range |
|---|---:|
| 1 | 1 - 2,500,000 |
| 2 | 2,500,001 - 5,000,000 |
| 3 | 5,000,001 - 7,500,000 |
| 4 | 7,500,001 - 10,000,000 |

## 1. Validate the Oracle source

Connect to `FREEPDB1` and verify the row count and ID range:

```sql
SELECT
    COUNT(*) AS row_count,
    MIN(sales_id) AS min_id,
    MAX(sales_id) AS max_id
FROM fact_sales;
```

For this test, the expected source range is:

```text
ROW_COUNT    MIN_ID    MAX_ID
----------   ------    --------
10000000     1         10000000
```

## 2. Configure the Fabric Copy activity

In the Fabric Data Factory pipeline, add or open the Copy activity and select the Oracle connection.

Under **Source** configure:

```text
Use query:               Table
Table:                   FABRIC_TEST.FACT_SALES
Partition option:        Dynamic range
Partition column name:   SALES_ID
Partition lower bound:   1
Partition upper bound:   10000000
```

Under the Copy activity **Settings**, configure the desired **Degree of copy parallelism**. For this test, configure a degree that results in four range reads so the behavior can be easily observed.

Run the pipeline and wait for the Copy activity to complete.

## 3. Inspect SQL received by Oracle

Immediately after the Copy activity finishes, connect to Oracle with an account that can query `V$SQL`.

For example, when working directly on the Oracle VM:

```bash
sqlplus / as sysdba
```

If necessary, switch to the PDB containing the test schema:

```sql
ALTER SESSION SET CONTAINER=FREEPDB1;
```

Query the SQL cache for statements parsed under `FABRIC_TEST`:

```sql
SELECT
    sql_id,
    executions,
    parsing_schema_name,
    sql_text
FROM v$sql
WHERE parsing_schema_name = 'FABRIC_TEST';
```

To focus specifically on statements issued against the fact table, use:

```sql
SELECT
    sql_id,
    executions,
    parsing_schema_name,
    sql_text
FROM v$sql
WHERE parsing_schema_name = 'FABRIC_TEST'
  AND UPPER(sql_text) LIKE '%FACT_SALES%';
```

To narrow the output specifically to the dynamic-range queries:

```sql
SELECT
    sql_id,
    executions,
    parsing_schema_name,
    sql_text
FROM v$sql
WHERE parsing_schema_name = 'FABRIC_TEST'
  AND UPPER(sql_text) LIKE '%FACT_SALES%'
  AND UPPER(sql_text) LIKE '%SALES_ID%'
ORDER BY sql_id;
```

## 4. Sample observed Oracle SQL

The test produced four distinct range queries against `FACT_SALES`.

### Partition 1

SQL ID: `d8nkmsjuh3dmy`

```sql
SELECT * FROM "FABRIC_TEST"."FACT_SALES"
WHERE SALES_ID <= 2500000
  AND SALES_ID >= 1
```

### Partition 2

SQL ID: `gjx676vru9wam`

```sql
SELECT * FROM "FABRIC_TEST"."FACT_SALES"
WHERE SALES_ID <= 5000000
  AND SALES_ID >= 2500001
```

### Partition 3

SQL ID: `dmv2df0ab8amx`

```sql
SELECT * FROM "FABRIC_TEST"."FACT_SALES"
WHERE SALES_ID <= 7500000
  AND SALES_ID >= 5000001
```

### Partition 4

SQL ID: `9nnqrhtr8fzdm`

```sql
SELECT * FROM "FABRIC_TEST"."FACT_SALES"
WHERE SALES_ID <= 10000000
  AND SALES_ID >= 7500001
```

Each of these statements showed one execution in the captured `V$SQL` results.

Taken together, the predicates cover the complete `SALES_ID` range from 1 through 10,000,000 without overlap between adjacent ranges.

## 5. Other SQL observed during the test

The Oracle SQL cache also contained statements that were not dynamic-range data reads. These are useful for distinguishing connector metadata/probing activity from the actual partition reads.

A row probe was observed:

```sql
SELECT * FROM "FABRIC_TEST"."FACT_SALES" WHERE ROWNUM <= 1
```

An unfiltered table read was also present in `V$SQL`:

```sql
SELECT * FROM "FABRIC_TEST"."FACT_SALES"
```

Metadata queries were present against Oracle catalog views, including `ALL_TABLES` and `ALL_VIEWS`.

A previously executed validation query was also visible:

```sql
SELECT COUNT(*) AS row_count,
       MIN(sales_id) AS min_id,
       MAX(sales_id) AS max_id
FROM fact_sales
```

For validation of dynamic range behavior, focus on the statements containing both `FACT_SALES` and `SALES_ID` range predicates rather than assuming every `FABRIC_TEST` statement was part of the partitioned data transfer.

## 6. What this test demonstrates

The captured SQL provides direct evidence that the Fabric Copy activity generated separate Oracle queries for different portions of the configured `SALES_ID` range.

For the 10-million-row test, the observed predicates split the source into four contiguous ranges:

```text
1            - 2,500,000
2,500,001    - 5,000,000
5,000,001    - 7,500,000
7,500,001    - 10,000,000
```

This is useful when comparing Copy activity configurations such as:

1. No partitioning
2. Dynamic range partitioning
3. Different degrees of copy parallelism
4. Oracle physical table partitioning

For each run, capture the Copy activity duration/throughput together with the Oracle SQL observed in `V$SQL`. This makes it possible to correlate the Fabric configuration with the statements Oracle actually received.

## 7. Suggested repeatable benchmark

Run the same source table through several configurations while leaving the row count and destination unchanged.

### Baseline

```text
Partition option: None
```

Capture the Fabric Copy activity metrics and the corresponding Oracle SQL.

### Dynamic range

```text
Partition option: Dynamic range
Partition column: SALES_ID
Lower bound:      1
Upper bound:      10000000
```

Capture the same metrics and query `V$SQL` again.

### Validation query

After each dynamic-range run:

```sql
SELECT
    sql_id,
    executions,
    sql_text
FROM v$sql
WHERE parsing_schema_name = 'FABRIC_TEST'
  AND UPPER(sql_text) LIKE '%FACT_SALES%'
  AND UPPER(sql_text) LIKE '%SALES_ID%';
```

The presence of multiple `SELECT` statements with distinct, contiguous `SALES_ID` predicates confirms that the source range was split into separate Oracle queries for that observed run.

## Notes

- `V$SQL` is a shared SQL-area view, not a permanent audit history. Run the inspection close to the Copy activity when using it for this test.
- Statements in `V$SQL` can include queries from manual testing as well as connector activity. Filtering by schema, table, and the partition-column predicate makes the evidence easier to interpret.
- The SQL captured above is the actual sample output from this test environment and should be treated as an observation of this run, not as a guarantee that every Fabric run will produce identical numeric boundaries.
