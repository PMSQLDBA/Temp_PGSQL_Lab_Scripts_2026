# LAB-6 - PostgreSQL 18.6 pgBackRest Differential & Incremental Backup-Restore on AWS EC2

**Lab Status:** Planned / Ready for Testing\
**Previous Lab:** LAB-5 - pgBackRest Full Backup & Restore - SIGNED OFF\
**Platform:** AWS EC2\
**Operating System:** Amazon Linux 2023\
**PostgreSQL:** 18.6\
**pgBackRest:** 2.59.1\
**Test Database:** `dvdrental`\
**Source Server:** pgsql01 / `172.30.2.177`\
**Restore Server:** pgsql02 / `172.30.2.128`\
**Stanza:** `lab5`

> This README is the end-to-end LAB-6 execution plan. Run the lab
> step-by-step and validate each stage before continuing.

------------------------------------------------------------------------

## 1. LAB Objective

LAB-6 extends the signed-off LAB-5 full-backup baseline and demonstrates
the pgBackRest backup chain.

We will test:

1.  Existing LAB-5 full backup as the base.
2.  Generate controlled changes in `dvdrental`.
3.  Take a **differential backup**.
4.  Generate another controlled set of changes.
5.  Take an **incremental backup**.
6.  Inspect the backup dependency chain.
7.  Transfer the updated repository to pgsql02.
8.  Restore the **differential backup** and validate the data state.
9.  Restore the **incremental backup** and validate the later data
    state.
10. Confirm pgsql02 is writable after the final restore.

pgBackRest defines the backup types as:

-   **Full (`full`)** - copies all database-cluster files and does not
    depend on a previous backup.
-   **Differential (`diff`)** - copies files changed since the last full
    backup.
-   **Incremental (`incr`)** - copies files changed since the last
    successful backup.

Archived WAL is used by pgBackRest to make online backups consistent and
is separate from these backup types.

------------------------------------------------------------------------

## 2. Architecture

``` text
                    LAB-6 BACKUP CHAIN

pgsql01 - PostgreSQL 18.6
172.30.2.177
        |
        | dvdrental
        |
        +-----------------------------+
        |                             |
        | Existing LAB-5 FULL         |
        | 20260903-013127F            |
        v                             |
   pgBackRest Repository              |
        |                             |
        +-- FULL ---------------------+
        |
        | Change Set A
        v
   DIFFERENTIAL BACKUP
   (--type=diff)
        |
        | Change Set B
        v
   INCREMENTAL BACKUP
   (--type=incr)
        |
        +------ Archived WAL
        |
        | Repository Transfer
        v
pgsql02 - PostgreSQL 18.6
172.30.2.128
        |
        +-- Restore Differential
        |      |
        |      +-- Validate Change Set A
        |
        +-- Restore Incremental
               |
               +-- Validate Change Sets A + B
               |
               v
          LAB-6 PASS
```

------------------------------------------------------------------------

## 3. Current LAB-5 Baseline

The LAB-5 full backup is our dependency root.

Known full backup:

``` text
Backup Label : 20260903-013127F
Backup Type  : Full
Database     : dvdrental
Database Size: ~15 MB
Stanza       : lab5
Status       : OK
```

Known source baseline:

  Object             Count
  --------------- --------
  Public tables         15
  actor                200
  customer             599
  film               1,000
  inventory          4,581
  payment           14,596
  rental            16,044

LAB-5 created `lab5_restore_validation` only on pgsql02 after restore. Do
not use that pgsql02-only table as a LAB-6 source baseline.

------------------------------------------------------------------------

# PHASE A - PRE-LAB VALIDATION ON pgsql01

## Step 1 - Connect to pgsql01

From Windows PowerShell:

``` powershell
ssh -i "C:\Users\DELL\Downloads\pgsqllabssep2026.pem" `
ec2-user@ec2-18-221-154-4.us-east-2.compute.amazonaws.com
```

Validate:

``` bash
hostname
hostname -I
```

Expected:

``` text
pgsql01
172.30.2.177
```

------------------------------------------------------------------------

## Step 2 - Validate PostgreSQL

``` bash
sudo systemctl is-active postgresql
```

``` bash
sudo -u postgres psql -d postgres -c "
SELECT version();
"
```

``` bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS in_recovery,
    current_setting('transaction_read_only') AS transaction_read_only;
"
```

Expected:

``` text
PostgreSQL service       : active
pg_is_in_recovery()      : false
transaction_read_only    : off
```

------------------------------------------------------------------------

## Step 3 - Validate pgBackRest

``` bash
pgbackrest version
```

Expected:

``` text
pgBackRest 2.59.1
```

Validate configuration:

``` bash
sudo cat /etc/pgbackrest/pgbackrest.conf
```

Expected LAB configuration:

``` ini
[lab5]
pg1-path=/var/lib/pgsql/data

[global]
repo1-path=/var/lib/pgsql/pgbackrest_repo
repo1-retention-full=2
start-fast=y
process-max=2
log-level-console=info
log-level-file=detail
```

------------------------------------------------------------------------

## Step 4 - Validate Existing Full Backup

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
info
```

Confirm the full backup exists:

``` text
20260903-013127F
```

Optional detailed validation:

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--output=json \
info
```

Do not continue unless the stanza status is `ok` and the full backup is
visible.

------------------------------------------------------------------------

## Step 5 - Validate WAL Archiving

``` bash
sudo -u postgres psql -d postgres -c "
SHOW archive_mode;
SHOW archive_command;
SHOW wal_level;
"
```

Expected:

``` text
archive_mode    = on
archive_command = pgbackrest --stanza=lab5 archive-push %p
wal_level       = replica
```

Validate pgBackRest:

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
check
```

Expected result:

``` text
check command end: completed successfully
```

------------------------------------------------------------------------

## Step 6 - Validate DVD Rental Baseline

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT COUNT(*) AS table_count
FROM pg_tables
WHERE schemaname='public';
"
```

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT 'actor' AS table_name, COUNT(*) AS row_count FROM actor
UNION ALL
SELECT 'customer', COUNT(*) FROM customer
UNION ALL
SELECT 'film', COUNT(*) FROM film
UNION ALL
SELECT 'inventory', COUNT(*) FROM inventory
UNION ALL
SELECT 'payment', COUNT(*) FROM payment
UNION ALL
SELECT 'rental', COUNT(*) FROM rental;
"
```

Expected LAB-5 source baseline:

``` text
actor      200
customer   599
film       1000
inventory  4581
payment    14596
rental     16044
```

------------------------------------------------------------------------

# PHASE B - CREATE DIFFERENTIAL BACKUP

## Step 7 - Create LAB-6 Validation Table

Use a dedicated table so the DVD Rental sample tables remain unchanged.

``` bash
sudo -u postgres psql -d dvdrental -c "
CREATE TABLE IF NOT EXISTS lab6_backup_validation
(
    id          bigserial PRIMARY KEY,
    backup_stage varchar(30) NOT NULL,
    test_note   text NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);
"
```

Validate:

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT COUNT(*) AS lab6_rows
FROM lab6_backup_validation;
"
```

------------------------------------------------------------------------

## Step 8 - Create Change Set A

These rows must exist in the differential backup.

``` bash
sudo -u postgres psql -d dvdrental -c "
INSERT INTO lab6_backup_validation
    (backup_stage, test_note)
VALUES
    ('DIFFERENTIAL', 'LAB-6 Change Set A - Row 1'),
    ('DIFFERENTIAL', 'LAB-6 Change Set A - Row 2'),
    ('DIFFERENTIAL', 'LAB-6 Change Set A - Row 3');
"
```

Validate:

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    id,
    backup_stage,
    test_note,
    created_at
FROM lab6_backup_validation
ORDER BY id;
"
```

Expected:

``` text
3 rows
backup_stage = DIFFERENTIAL
```

------------------------------------------------------------------------

## Step 9 - Force WAL Switch

``` bash
sudo -u postgres psql -d postgres -c "
SELECT pg_switch_wal();
"
```

Check archiver:

``` bash
sudo -u postgres psql -d postgres -c "
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count
FROM pg_stat_archiver;
"
```

Expected:

``` text
failed_count = 0
```

------------------------------------------------------------------------

## Step 10 - Take Differential Backup

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--type=diff \
backup
```

Expected successful ending:

``` text
backup command end: completed successfully
```

A differential backup label will normally contain the full-backup prefix
and end in `D`.

Example structure:

``` text
20260903-013127F_<timestamp>D
```

Record the actual differential label from your output.

------------------------------------------------------------------------

## Step 11 - Validate Differential Backup

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
info
```

You should now see:

``` text
full backup: ...
diff backup: ...
```

For JSON:

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--output=json \
info
```

Record:

``` text
FULL_LABEL=<actual full label>
DIFF_LABEL=<actual differential label>
```

For this lab the existing full label is:

``` text
FULL_LABEL=20260903-013127F
```

------------------------------------------------------------------------

# PHASE C - CREATE INCREMENTAL BACKUP

## Step 12 - Create Change Set B

Run this only **after the differential backup has completed**.

``` bash
sudo -u postgres psql -d dvdrental -c "
INSERT INTO lab6_backup_validation
    (backup_stage, test_note)
VALUES
    ('INCREMENTAL', 'LAB-6 Change Set B - Row 1'),
    ('INCREMENTAL', 'LAB-6 Change Set B - Row 2'),
    ('INCREMENTAL', 'LAB-6 Change Set B - Row 3');
"
```

Validate:

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    id,
    backup_stage,
    test_note,
    created_at
FROM lab6_backup_validation
ORDER BY id;
"
```

Expected total:

``` text
6 rows

3 DIFFERENTIAL
3 INCREMENTAL
```

Summary query:

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    backup_stage,
    COUNT(*) AS row_count
FROM lab6_backup_validation
GROUP BY backup_stage
ORDER BY backup_stage;
"
```

Expected:

``` text
DIFFERENTIAL  3
INCREMENTAL   3
```

------------------------------------------------------------------------

## Step 13 - Force Another WAL Switch

``` bash
sudo -u postgres psql -d postgres -c "
SELECT pg_switch_wal();
"
```

Validate:

``` bash
sudo -u postgres psql -d postgres -c "
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count
FROM pg_stat_archiver;
"
```

------------------------------------------------------------------------

## Step 14 - Take Incremental Backup

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--type=incr \
backup
```

Expected:

``` text
backup command end: completed successfully
```

The incremental label should end in `I`.

Example structure:

``` text
20260903-013127F_<timestamp>D_<timestamp>I
```

Record the actual incremental label.

------------------------------------------------------------------------

## Step 15 - Validate Complete Backup Chain

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
info
```

The repository should contain the dependency chain:

``` text
FULL
  |
  +-- DIFFERENTIAL
        |
        +-- INCREMENTAL
```

Record:

``` text
FULL_LABEL=<actual full label>
DIFF_LABEL=<actual differential label>
INCR_LABEL=<actual incremental label>
```

Detailed inspection:

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--set=<INCR_LABEL> \
info
```

The `--set` detail is useful because pgBackRest reports the additional
backup sets required to restore the selected backup.

------------------------------------------------------------------------

# PHASE D - CAPTURE SOURCE VALIDATION

## Step 16 - Capture pgsql01 Final Data State

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    id,
    backup_stage,
    test_note,
    created_at
FROM lab6_backup_validation
ORDER BY id;
"
```

Expected:

``` text
6 rows
```

Validate original sample data remains intact:

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT 'actor' AS table_name, COUNT(*) AS row_count FROM actor
UNION ALL
SELECT 'customer', COUNT(*) FROM customer
UNION ALL
SELECT 'film', COUNT(*) FROM film
UNION ALL
SELECT 'inventory', COUNT(*) FROM inventory
UNION ALL
SELECT 'payment', COUNT(*) FROM payment
UNION ALL
SELECT 'rental', COUNT(*) FROM rental;
"
```

Expected:

``` text
actor      200
customer   599
film       1000
inventory  4581
payment    14596
rental     16044
```

------------------------------------------------------------------------

# PHASE E - PREPARE pgsql02 FOR RESTORE

## Step 17 - Connect to pgsql02

From Windows PowerShell:

``` powershell
ssh -i "C:\Users\DELL\Downloads\pgsqllabssep2026.pem" `
ec2-user@ec2-18-220-37-63.us-east-2.compute.amazonaws.com
```

Validate:

``` bash
hostname
hostname -I
```

Expected:

``` text
pgsql02
172.30.2.128
```

------------------------------------------------------------------------

## Step 18 - Stop PostgreSQL on pgsql02

``` bash
sudo systemctl stop postgresql
```

Validate:

``` bash
sudo systemctl is-active postgresql
```

``` bash
sudo ss -lntp | grep 5432 || \
echo "pgsql02 PostgreSQL is safely stopped"
```

Expected:

``` text
inactive
pgsql02 PostgreSQL is safely stopped
```

------------------------------------------------------------------------

## Step 19 - Preserve Current pgsql02 LAB-5 Cluster

Use a unique LAB-6 preservation directory.

``` bash
sudo mv \
/var/lib/pgsql/data \
/var/lib/pgsql/data_before_lab6_restore_20260903
```

Validate:

``` bash
sudo ls -ld \
/var/lib/pgsql/data_before_lab6_restore_20260903
```

Do not delete this preservation directory during the lab.

------------------------------------------------------------------------

# PHASE F - TRANSFER UPDATED REPOSITORY

The LAB-6 repository now contains the full, differential, incremental,
and archived WAL required by the backup chain.

Because the current lab uses local repositories on each EC2 server, copy
the updated pgsql01 repository to pgsql02.

## Step 20 - Package Repository on pgsql01

On pgsql01:

``` bash
sudo tar -czf \
/tmp/lab6_pgbackrest_repo.tar.gz \
-C /var/lib/pgsql \
pgbackrest_repo
```

``` bash
sudo chown ec2-user:ec2-user \
/tmp/lab6_pgbackrest_repo.tar.gz
```

Validate:

``` bash
ls -lh /tmp/lab6_pgbackrest_repo.tar.gz
```

------------------------------------------------------------------------

## Step 21 - Copy Repository pgsql01 -\> Windows

From Windows PowerShell:

``` powershell
scp -i "C:\Users\DELL\Downloads\pgsqllabssep2026.pem" `
ec2-user@ec2-18-221-154-4.us-east-2.compute.amazonaws.com:/tmp/lab6_pgbackrest_repo.tar.gz `
"C:\Users\DELL\Downloads\lab6_pgbackrest_repo.tar.gz"
```

Validate:

``` powershell
Get-Item "C:\Users\DELL\Downloads\lab6_pgbackrest_repo.tar.gz"
```

------------------------------------------------------------------------

## Step 22 - Copy Repository Windows -\> pgsql02

``` powershell
scp -i "C:\Users\DELL\Downloads\pgsqllabssep2026.pem" `
"C:\Users\DELL\Downloads\lab6_pgbackrest_repo.tar.gz" `
ec2-user@ec2-18-220-37-63.us-east-2.compute.amazonaws.com:/tmp/lab6_pgbackrest_repo.tar.gz
```

------------------------------------------------------------------------

## Step 23 - Replace LAB Repository Copy on pgsql02

On pgsql02:

``` bash
sudo rm -rf /var/lib/pgsql/pgbackrest_repo
```

``` bash
sudo tar -xzf \
/tmp/lab6_pgbackrest_repo.tar.gz \
-C /var/lib/pgsql
```

``` bash
sudo chown -R postgres:postgres \
/var/lib/pgsql/pgbackrest_repo
```

``` bash
sudo chmod 750 \
/var/lib/pgsql/pgbackrest_repo
```

Validate:

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
info
```

Confirm the full, differential, and incremental backups are all visible.

------------------------------------------------------------------------

# PHASE G - RESTORE DIFFERENTIAL BACKUP

## Step 24 - Create Empty PGDATA

``` bash
sudo mkdir -p /var/lib/pgsql/data
sudo chown postgres:postgres /var/lib/pgsql/data
sudo chmod 700 /var/lib/pgsql/data
```

------------------------------------------------------------------------

## Step 25 - Restore Differential Backup

Replace `<DIFF_LABEL>` with the exact differential label captured
earlier.

``` bash
cd /tmp

sudo -u postgres pgbackrest \
--stanza=lab5 \
--set=<DIFF_LABEL> \
restore
```

Expected:

``` text
restore command end: completed successfully
```

------------------------------------------------------------------------

## Step 26 - Inspect Recovery Configuration

``` bash
sudo cat /var/lib/pgsql/data/postgresql.auto.conf
```

Check for any inherited replication connection setting:

``` bash
sudo grep -n "primary_conninfo" \
/var/lib/pgsql/data/postgresql.auto.conf || \
echo "No primary_conninfo configured"
```

For this independent restore, pgsql02 must not be configured to follow pgsql01
as a streaming standby.

If a stale `primary_conninfo` is present, preserve the file and remove
only that setting:

``` bash
sudo cp \
/var/lib/pgsql/data/postgresql.auto.conf \
/var/lib/pgsql/data/postgresql.auto.conf.before_lab6_diff_cleanup
```

``` bash
sudo sed -i \
'/^[[:space:]]*primary_conninfo[[:space:]]*=/d' \
/var/lib/pgsql/data/postgresql.auto.conf
```

``` bash
sudo chown postgres:postgres \
/var/lib/pgsql/data/postgresql.auto.conf \
/var/lib/pgsql/data/postgresql.auto.conf.before_lab6_diff_cleanup
```

Validate:

``` bash
sudo grep -n "primary_conninfo" \
/var/lib/pgsql/data/postgresql.auto.conf || \
echo "PASS: no primary_conninfo configured"
```

Do not manually remove the pgBackRest-generated recovery signal before
PostgreSQL starts.

------------------------------------------------------------------------

## Step 27 - Start Differential Restore

``` bash
sudo systemctl start postgresql
```

``` bash
sudo systemctl is-active postgresql
```

Validate recovery completion:

``` bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS in_recovery,
    current_setting('transaction_read_only') AS transaction_read_only;
"
```

For an independent completed restore, expected:

``` text
in_recovery              = false
transaction_read_only    = off
```

------------------------------------------------------------------------

## Step 28 - Validate Differential Data State

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    id,
    backup_stage,
    test_note,
    created_at
FROM lab6_backup_validation
ORDER BY id;
"
```

The key differential restore expectation is:

``` text
Change Set A : PRESENT
Change Set B : NOT PRESENT
```

Summary:

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    backup_stage,
    COUNT(*) AS row_count
FROM lab6_backup_validation
GROUP BY backup_stage
ORDER BY backup_stage;
"
```

Expected:

``` text
DIFFERENTIAL  3
```

There should be no `INCREMENTAL` rows at this restore point.

This is the most important proof that the selected differential backup
restored the database to the differential backup state rather than the
later incremental state.

------------------------------------------------------------------------

# PHASE H - RESTORE INCREMENTAL BACKUP

## Step 29 - Stop pgsql02

``` bash
sudo systemctl stop postgresql
sudo systemctl is-active postgresql
```

Expected:

``` text
inactive
```

------------------------------------------------------------------------

## Step 30 - Preserve Differential Restore

``` bash
sudo mv \
/var/lib/pgsql/data \
/var/lib/pgsql/data_after_lab6_diff_restore_20260903
```

This gives us a preserved copy of the successfully validated
differential restore.

------------------------------------------------------------------------

## Step 31 - Create Clean PGDATA

``` bash
sudo mkdir -p /var/lib/pgsql/data
sudo chown postgres:postgres /var/lib/pgsql/data
sudo chmod 700 /var/lib/pgsql/data
```

------------------------------------------------------------------------

## Step 32 - Restore Incremental Backup

Replace `<INCR_LABEL>` with the actual incremental backup label.

``` bash
cd /tmp

sudo -u postgres pgbackrest \
--stanza=lab5 \
--set=<INCR_LABEL> \
restore
```

Expected:

``` text
restore command end: completed successfully
```

------------------------------------------------------------------------

## Step 33 - Validate Restore Configuration

``` bash
sudo cat /var/lib/pgsql/data/postgresql.auto.conf
```

Check:

``` bash
sudo grep -n "primary_conninfo" \
/var/lib/pgsql/data/postgresql.auto.conf || \
echo "No primary_conninfo configured"
```

If the restored backup contains a stale `primary_conninfo`, preserve and
remove it exactly as done for the differential restore:

``` bash
sudo cp \
/var/lib/pgsql/data/postgresql.auto.conf \
/var/lib/pgsql/data/postgresql.auto.conf.before_lab6_incr_cleanup
```

``` bash
sudo sed -i \
'/^[[:space:]]*primary_conninfo[[:space:]]*=/d' \
/var/lib/pgsql/data/postgresql.auto.conf
```

``` bash
sudo chown postgres:postgres \
/var/lib/pgsql/data/postgresql.auto.conf \
/var/lib/pgsql/data/postgresql.auto.conf.before_lab6_incr_cleanup
```

------------------------------------------------------------------------

## Step 34 - Start Incremental Restore

``` bash
sudo systemctl start postgresql
```

Validate:

``` bash
sudo systemctl is-active postgresql
```

``` bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS in_recovery,
    current_setting('transaction_read_only') AS transaction_read_only;
"
```

Expected after completed independent recovery:

``` text
in_recovery              = false
transaction_read_only    = off
```

------------------------------------------------------------------------

## Step 35 - Validate Incremental Data State

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    id,
    backup_stage,
    test_note,
    created_at
FROM lab6_backup_validation
ORDER BY id;
"
```

Expected:

``` text
6 rows
```

Summary:

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    backup_stage,
    COUNT(*) AS row_count
FROM lab6_backup_validation
GROUP BY backup_stage
ORDER BY backup_stage;
"
```

Expected:

``` text
DIFFERENTIAL  3
INCREMENTAL   3
```

This proves the incremental restore contains both:

``` text
Full Backup
   +
Differential Change Set A
   +
Incremental Change Set B
```

------------------------------------------------------------------------

# PHASE I - FINAL DATA VALIDATION

## Step 36 - Validate DVD Rental Tables

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT 'actor' AS table_name, COUNT(*) AS row_count FROM actor
UNION ALL
SELECT 'customer', COUNT(*) FROM customer
UNION ALL
SELECT 'film', COUNT(*) FROM film
UNION ALL
SELECT 'inventory', COUNT(*) FROM inventory
UNION ALL
SELECT 'payment', COUNT(*) FROM payment
UNION ALL
SELECT 'rental', COUNT(*) FROM rental;
"
```

Expected:

``` text
actor      200
customer   599
film       1000
inventory  4581
payment    14596
rental     16044
```

------------------------------------------------------------------------

## Step 37 - Validate Database Size

``` bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_size_pretty(pg_database_size('dvdrental')) AS dvdrental_size;
"
```

The database may be slightly larger than the LAB-5 baseline because
LAB-6 created an additional validation table.

------------------------------------------------------------------------

## Step 38 - Final Write Test on pgsql02

``` bash
sudo -u postgres psql -d dvdrental -c "
INSERT INTO lab6_backup_validation
    (backup_stage, test_note)
VALUES
    ('RESTORE-VALIDATION',
     'LAB-6 Incremental Restore Final Write Test on pgsql02');
"
```

Validate:

``` bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    id,
    backup_stage,
    test_note,
    created_at
FROM lab6_backup_validation
ORDER BY id;
"
```

Expected final result:

``` text
3 DIFFERENTIAL rows
3 INCREMENTAL rows
1 RESTORE-VALIDATION row
```

------------------------------------------------------------------------

# PHASE J - pgBackRest FINAL VALIDATION

## Step 39 - Validate Backup Repository

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
info
```

Confirm:

``` text
Stanza       : lab5
Status       : ok
Full         : present
Differential : present
Incremental  : present
```

------------------------------------------------------------------------

## Step 40 - Validate Backup Dependency Information

Use the exact incremental label:

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--set=<INCR_LABEL> \
info
```

Review the required backup sets for restoring the selected incremental
backup.

Conceptually:

``` text
FULL
  |
  +---- DIFF
          |
          +---- INCR
```

An incremental backup cannot be treated as an independent full backup.
Its required parent backup sets must remain valid.

------------------------------------------------------------------------

# 4. Backup Type Comparison

  -----------------------------------------------------------------------
  Backup            pgBackRest        Dependency        LAB-6 Use
  ----------------- ----------------- ----------------- -----------------
  Full              `--type=full`     None              LAB-5 base

  Differential      `--type=diff`     Last full         Change Set A

  Incremental       `--type=incr`     Last successful   Change Set B
                                      backup            

  Archived WAL      `archive-push`    Backup/recovery   Online backup
                                      architecture      consistency and
                                                        recovery
  -----------------------------------------------------------------------

Important:

``` text
FULL
  |
  +---------- DIFF
                 |
                 +---------- INCR
```

The differential backup always references the last full backup. The
incremental backup references the last successful backup, which in this
scenario is the differential backup.

------------------------------------------------------------------------

# 5. Expected LAB-6 Result

At completion:

``` text
pgsql01
 |
 |-- Full Backup (LAB-5)
 |
 |-- Change Set A
 |       |
 |       +-- Differential Backup
 |
 |-- Change Set B
 |       |
 |       +-- Incremental Backup
 |
 +---------------------------+
                             |
                      pgBackRest Repo
                             |
                             v
                            pgsql02
                             |
               +-------------+-------------+
               |                           |
        Restore DIFF                 Restore INCR
               |                           |
       Change Set A                  Change Set A
          Present                       +
       Change Set B                  Change Set B
        Not Present                    Present
                                           |
                                           v
                                    Final Write Test
                                           |
                                           v
                                      LAB-6 PASS
```

------------------------------------------------------------------------

# 6. LAB-6 Acceptance Checklist

LAB-6 can be signed off only when all of the following are verified:

-   [ ] PostgreSQL 18.6 active on pgsql01.
-   [ ] pgBackRest 2.59.1 available.
-   [ ] Existing LAB-5 full backup is valid.
-   [ ] WAL archiving is healthy.
-   [ ] `pgbackrest check` succeeds.
-   [ ] Change Set A contains 3 rows.
-   [ ] Differential backup completes successfully.
-   [ ] Differential backup is visible in `pgbackrest info`.
-   [ ] Change Set B adds another 3 rows.
-   [ ] Incremental backup completes successfully.
-   [ ] Incremental backup is visible in `pgbackrest info`.
-   [ ] Repository transferred to pgsql02.
-   [ ] Differential restore completes successfully.
-   [ ] Differential restore contains Change Set A only.
-   [ ] Incremental rows are absent from differential restore.
-   [ ] Incremental restore completes successfully.
-   [ ] Incremental restore contains Change Sets A and B.
-   [ ] DVD Rental baseline tables match the source.
-   [ ] pgsql02 exits recovery and is writable.
-   [ ] Final pgsql02 write test succeeds.
-   [ ] pgBackRest repository status remains `ok`.

------------------------------------------------------------------------

# 7. Important Lab Notes

## Differential Backup

A pgBackRest differential backup copies database-cluster files changed
since the **last full backup**.

Therefore:

``` text
Full F1
  |
  +-- Diff D1
```

D1 depends on F1.

## Incremental Backup

A pgBackRest incremental backup copies database-cluster files changed
since the **last successful backup**.

In this LAB:

``` text
F1 -> D1 -> I1
```

I1 is based on D1 because D1 is the immediately preceding successful
backup.

## WAL Archiving

WAL archiving is not another `--type` value.

The backup types are:

``` text
full
diff
incr
```

The PostgreSQL WAL archive supports consistency and recovery of the
online backup chain.

## Restore Selection

Always use the exact label when demonstrating a specific restore:

``` bash
--set=<BACKUP_LABEL>
```

This prevents ambiguity about which backup state is being tested.

## Local Repository Design

This lab currently uses a local pgBackRest repository on pgsql01 and
transfers a copy to pgsql02 for restore testing.

``` text
pgsql01 local repo
     |
     | package + SCP
     v
Windows workstation
     |
     | SCP
     v
pgsql02 local repo copy
```

The two repositories are separate copies; they are not a shared
repository.

------------------------------------------------------------------------

# 8. Production Considerations

The lab is intentionally designed for learning and controlled
validation. For production environments consider:

1.  Use durable backup storage independent of the database server.
2.  Use object storage such as Amazon S3 when it fits the recovery
    design.
3.  Encrypt backup repositories where required.
4.  Define full/differential/incremental schedules according to RPO and
    RTO.
5.  Set retention based on restore requirements, storage capacity, and
    compliance.
6.  Monitor `pg_stat_archiver` and pgBackRest repository health.
7.  Alert on failed WAL archival.
8.  Perform scheduled restore tests.
9.  Protect repository credentials and configuration files.
10. Do not assume that a successful backup command proves
    recoverability; regularly perform restore validation.
11. Preserve all dependent backup sets needed by differential and
    incremental backups.
12. Document recovery procedures separately from backup scheduling.

------------------------------------------------------------------------

# 9. Useful Commands Cheat Sheet

## Backup information

``` bash
sudo -u postgres pgbackrest --stanza=lab5 info
```

## pgBackRest health check

``` bash
sudo -u postgres pgbackrest --stanza=lab5 check
```

## Full backup

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--type=full \
backup
```

## Differential backup

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--type=diff \
backup
```

## Incremental backup

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--type=incr \
backup
```

## Restore exact backup

``` bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--set=<BACKUP_LABEL> \
restore
```

## Force WAL switch

``` bash
sudo -u postgres psql -d postgres -c "
SELECT pg_switch_wal();
"
```

## Check archiver

``` bash
sudo -u postgres psql -d postgres -c "
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count
FROM pg_stat_archiver;
"
```

## Recovery status

``` bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery(),
    current_setting('transaction_read_only');
"
```

------------------------------------------------------------------------

# 10. Official References

-   pgBackRest User Guide: https://pgbackrest.org/user-guide.html
-   pgBackRest Command Reference: https://pgbackrest.org/command.html
-   PostgreSQL 18 Backup and Restore:
    https://www.postgresql.org/docs/18/backup.html
-   PostgreSQL 18 Continuous Archiving and PITR:
    https://www.postgresql.org/docs/18/continuous-archiving.html

------------------------------------------------------------------------

# 11. Final Sign-Off Template

Complete this only after all restore tests pass.

``` text
LAB-6
PostgreSQL 18.6 pgBackRest Differential & Incremental Backup/Restore

Source       : pgsql01 / 172.30.2.177
Restore      : pgsql02 / 172.30.2.128
Database     : dvdrental
PostgreSQL   : 18.6
pgBackRest   : 2.59.1
Stanza       : lab5

Full Backup          : PASS
Differential Backup  : PASS
Incremental Backup   : PASS
Differential Restore : PASS
Incremental Restore  : PASS
Data Validation      : PASS
Final Write Test     : PASS

Overall Status       : PASS / SIGNED OFF
```

------------------------------------------------------------------------

**End of LAB-6 README**
