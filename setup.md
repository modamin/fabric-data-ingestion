# Oracle 26ai Free Test Database for Microsoft Fabric

This guide sets up an Oracle 26ai Free database on an Oracle Linux VM, generates a representative `FACT_SALES` table, and connects Microsoft Fabric Data Factory pipelines to Oracle through an On-premises Data Gateway (OPDG).

## Architecture

```text
Microsoft Fabric
     |
     | Data Pipeline / Copy Activity
     v
On-premises Data Gateway
     |
     | Oracle Client for Microsoft Tools (OCMT)
     |
     | TCP 1521
     v
Oracle Linux VM
     |
     +-- Oracle 26ai Free
           |
           +-- FREEPDB1
                |
                +-- FABRIC_TEST
                     |
                     +-- FACT_SALES
```

## 1. Install Oracle 26ai Free

Install Oracle Database 26ai Free on the Oracle Linux VM.

Once the Oracle packages have been installed, configure the database:

```bash
sudo /etc/init.d/oracle-free-26ai configure
```

The default database configuration used by this guide is:

```text
CDB:      FREE
PDB:      FREEPDB1
Port:     1521
```

Check the service:

```bash
sudo /etc/init.d/oracle-free-26ai status
```

Expected:

```text
LISTENER status: RUNNING
FREE Database status: RUNNING
```

## 2. Enable Oracle After VM Reboots

Enable the Oracle service so the database and listener automatically start after the VM reboots:

```bash
sudo systemctl enable oracle-free-26ai
```

Start it if necessary:

```bash
sudo systemctl start oracle-free-26ai
```

Check:

```bash
sudo /etc/init.d/oracle-free-26ai status
```

Oracle should report both the listener and database as running.

## 3. Configure the Oracle Environment

Switch to the Oracle Linux account:

```bash
sudo su - oracle
```

Set the Oracle environment:

```bash
export ORACLE_HOME=/opt/oracle/product/26ai/dbhomeFree
export ORACLE_SID=FREE
export PATH=$ORACLE_HOME/bin:$PATH
```

Test local administrative access:

```bash
sqlplus / as sysdba
```

Check the PDB:

```sql
SHOW PDBS;
```

`FREEPDB1` should be `READ WRITE`.

If necessary:

```sql
ALTER PLUGGABLE DATABASE ALL OPEN;
```

Then exit:

```sql
EXIT;
```

## 4. Create the Fabric Test User

Connect as SYSDBA:

```bash
sqlplus / as sysdba
```

Switch to `FREEPDB1`:

```sql
ALTER SESSION SET CONTAINER=FREEPDB1;
```

Create the test schema:

```sql
CREATE USER fabric_test IDENTIFIED BY "YourStrongPassword";
GRANT CREATE SESSION, CREATE TABLE TO fabric_test;
ALTER USER fabric_test QUOTA UNLIMITED ON USERS;
```

Exit:

```sql
EXIT;
```

Test the account:

```bash
sqlplus fabric_test@localhost:1521/FREEPDB1
```

## 5. Create the Representative Fact Table

Connect as `fabric_test`:

```bash
sqlplus fabric_test@localhost:1521/FREEPDB1
```

Create the table once:

```sql
CREATE TABLE fact_sales (
    sales_id          NUMBER(12)       NOT NULL,
    customer_key      NUMBER(10)       NOT NULL,
    product_key       NUMBER(10)       NOT NULL,
    store_key         NUMBER(6)        NOT NULL,
    order_date        DATE             NOT NULL,
    ship_date         DATE,
    quantity          NUMBER(5)        NOT NULL,
    unit_price        NUMBER(10,2)     NOT NULL,
    discount_amount   NUMBER(10,2),
    tax_amount        NUMBER(10,2),
    sales_amount      NUMBER(12,2)     NOT NULL,
    cost_amount       NUMBER(12,2),
    channel_code      VARCHAR2(10),
    status_code       VARCHAR2(10),
    created_timestamp TIMESTAMP
);
```

The table contains a representative mixture of:

- Numeric keys
- Dates
- Timestamps
- Quantities
- Currency/decimal values
- String attributes

This makes it more representative of a warehouse fact table than simply generating millions of integers.

## 6. Generate 1 Million Rows

The following is the generator used for this test.

Each execution appends another **1,000,000 rows**.

Do not include `TRUNCATE TABLE` if the goal is to gradually increase the database size.

```sql
SET TIMING ON
INSERT /*+ APPEND */ INTO fact_sales
SELECT
(SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL AS sales_id,
MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,100000) + 1 AS customer_key,
MOD(((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL) * 31,10000) + 1 AS product_key,
MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,500) + 1 AS store_key,
DATE '2021-01-01' + MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,1825) AS order_date,
DATE '2021-01-01' + MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,1825) + MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,7) AS ship_date,
MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,10) + 1 AS quantity,
ROUND(5 + MOD(((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL) * 13,99500) / 100,2) AS unit_price,
CASE WHEN MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,5) = 0 THEN ROUND(MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,5000) / 100,2) ELSE 0 END AS discount_amount,
ROUND((5 + MOD(((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL) * 13,99500) / 100) * (MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,10) + 1) * 0.13,2) AS tax_amount,
ROUND((5 + MOD(((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL) * 13,99500) / 100) * (MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,10) + 1),2) AS sales_amount,
ROUND((5 + MOD(((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL) * 13,99500) / 100) * (MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,10) + 1) * 0.65,2) AS cost_amount,
CASE MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,4) WHEN 0 THEN 'WEB' WHEN 1 THEN 'STORE' WHEN 2 THEN 'MOBILE' ELSE 'PARTNER' END AS channel_code,
CASE MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,10) WHEN 0 THEN 'RETURNED' WHEN 1 THEN 'CANCELLED' ELSE 'COMPLETE' END AS status_code,
TIMESTAMP '2021-01-01 00:00:00' + NUMTODSINTERVAL(MOD((SELECT NVL(MAX(sales_id),0) FROM fact_sales) + LEVEL,157680000),'SECOND') AS created_timestamp
FROM dual
CONNECT BY LEVEL <= 1000000;
COMMIT;
SELECT COUNT(*) AS row_count, MIN(sales_id) AS min_id, MAX(sales_id) AS max_id FROM fact_sales;
```

Running the generator repeatedly gives:

```text
Run 1 -> +1M
Run 2 -> +1M
Run 3 -> +1M
...
```

For example:

```text
ROW_COUNT     MIN_ID       MAX_ID
---------     ------       -------
5000000       1            5000000
```

### Why Generate in 1M Batches?

An attempt to generate 40 million rows using:

```sql
CONNECT BY LEVEL <= 40000000
```

failed with:

```text
ORA-30009: Not enough memory for CONNECT BY operation
```

Generating data in 1-million-row increments avoids requiring Oracle to process a 40-million-level hierarchy at once and makes it easier to increase the test dataset gradually.

## 7. Check Oracle Table Size

From SQL*Plus:

```sql
SELECT
    COUNT(*) AS row_count
FROM fact_sales;
```

Check allocated segment size:

```sql
SELECT
    ROUND(SUM(bytes) / POWER(1024,3),3) AS size_gb
FROM user_segments
WHERE segment_name = 'FACT_SALES';
```

For more detail:

```sql
SELECT
    segment_name,
    segment_type,
    ROUND(bytes / POWER(1024,2),2) AS size_mb,
    ROUND(bytes / POWER(1024,3),3) AS size_gb
FROM user_segments
WHERE segment_name = 'FACT_SALES';
```

## 8. Check VM Disk Capacity

From Linux:

```bash
df -h /opt/oracle
```

Check the Oracle data footprint:

```bash
sudo du -sh /opt/oracle/oradata
```

Check the overall Oracle installation footprint:

```bash
sudo du -sh /opt/oracle
```

This should be monitored when repeatedly adding 1M-row batches.

## 9. Verify the Oracle Listener

Check port 1521:

```bash
sudo ss -lntp | grep 1521
```

A working listener should show something similar to:

```text
LISTEN ... *:1521 ... tnslsnr
```

Check the overall Oracle service:

```bash
sudo /etc/init.d/oracle-free-26ai status
```

Expected:

```text
LISTENER status: RUNNING
FREE Database status: RUNNING
```

## 10. Configure Oracle Linux Firewall

Check the firewall:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --list-ports
```

If TCP 1521 is not enabled:

```bash
sudo firewall-cmd --permanent --add-port=1521/tcp
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-ports
```

## 11. Configure Azure Networking

The Oracle VM must accept TCP traffic from the machine/network hosting the On-premises Data Gateway.

Configure the Azure Network Security Group associated with the Oracle VM/subnet to allow:

```text
Protocol:          TCP
Destination Port:  1521
Source:            OPDG network/IP
Action:            Allow
```

Restrict the source to the gateway network or IP rather than exposing TCP 1521 broadly.

## 12. Verify Connectivity from the OPDG Server

On the Windows server running the On-premises Data Gateway:

```powershell
Test-NetConnection <oracle-vm-ip> -Port 1521
```

The required result is:

```text
TcpTestSucceeded : True
```

Do not troubleshoot Fabric credentials until this succeeds.

If it returns `False`, investigate:

1. Oracle listener
2. Oracle Linux firewall
3. Azure NSG
4. Routing between the OPDG server and Oracle VM

## 13. Install Oracle Client for Microsoft Tools on OPDG

Install the **64-bit Oracle Client for Microsoft Tools (OCMT)** on the same Windows machine running the On-premises Data Gateway.

Use the default Oracle Client installation option.

After installation, restart the On-premises Data Gateway Windows service.

Without the required Oracle provider, the Fabric connection can fail with errors such as:

```text
Cannot load managed ODP.NET driver.

Unable to find the requested .Net Framework Data Provider.
It may not be installed.
```

## 14. Create the Oracle Connection in Fabric

In Microsoft Fabric:

```text
Settings
    -> Manage connections and gateways
        -> New
            -> On-premises
                -> Oracle Database
```

Select the existing On-premises Data Gateway.

Configure the connection:

```text
Connection name:
Oracle-Free-Test

Gateway:
<OPDG gateway>

Server:
<oracle-vm-ip>:1521/FREEPDB1

Authentication:
Basic

Username:
fabric_test

Password:
<password>
```

Example server syntax:

```text
20.150.211.25:1521/FREEPDB1
```

Prefer a private Oracle VM address/FQDN when private network connectivity exists between the gateway and Oracle VM.

## 15. Configure the Fabric Pipeline

Create a Fabric Data Factory pipeline and add:

```text
Copy Data
```

Configure the source:

```text
Source type:
Oracle Database

Connection:
Oracle-Free-Test

Table:
FABRIC_TEST.FACT_SALES
```

Select the required Fabric destination, such as a Lakehouse.

The complete data path is now:

```text
FABRIC_TEST.FACT_SALES
       |
       | Oracle TCP 1521
       v
On-premises Data Gateway + OCMT
       |
       v
Fabric Data Factory
       |
       v
Copy Activity
       |
       v
Fabric destination
```

## Troubleshooting

### ORA-12541: No listener

Check:

```bash
sudo /etc/init.d/oracle-free-26ai status
```

Start Oracle:

```bash
sudo systemctl start oracle-free-26ai
```

Enable automatic startup:

```bash
sudo systemctl enable oracle-free-26ai
```

### ORA-01034: Oracle instance is not available

Start the Oracle service:

```bash
sudo systemctl start oracle-free-26ai
```

Or connect locally:

```bash
sqlplus / as sysdba
```

Then:

```sql
STARTUP;
ALTER PLUGGABLE DATABASE ALL OPEN;
```

### ORA-30009: Not Enough Memory for CONNECT BY

Don't generate tens of millions of rows in one `CONNECT BY`.

Use the 1-million-row generator repeatedly.

### OPDG Cannot Reach Oracle

From the gateway machine:

```powershell
Test-NetConnection <oracle-vm-ip> -Port 1521
```

If:

```text
TcpTestSucceeded : False
```

check the Oracle Linux firewall, Azure NSG, routing, and listener.

### Managed ODP.NET Driver Error

If Fabric reports:

```text
Cannot load managed ODP.NET driver
```

verify that 64-bit OCMT is installed on the computer running OPDG and restart the gateway service.

## Final Test Environment

```text
Oracle Database:     Oracle 26ai Free
Container:           FREE
PDB:                 FREEPDB1
Schema:              FABRIC_TEST
Test table:          FACT_SALES
Generator batch:     1,000,000 rows
Oracle port:         TCP 1521
Authentication:      Basic
Connectivity:        On-premises Data Gateway
Oracle client:       64-bit OCMT on gateway server
Fabric workload:     Data Factory Pipeline / Copy Activity
```

The main validation points are:

1. Oracle database and listener are running.
2. `FREEPDB1` is open.
3. `fabric_test` can connect to `FREEPDB1`.
4. `FACT_SALES` has the desired number of rows.
5. The OPDG machine can reach Oracle TCP 1521.
6. OCMT is installed on the OPDG machine.
7. The Fabric Oracle connection uses `FREEPDB1`.
8. Fabric Copy Activity can read `FABRIC_TEST.FACT_SALES`.