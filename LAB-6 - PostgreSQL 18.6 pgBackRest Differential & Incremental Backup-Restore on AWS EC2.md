# LAB-6 — PostgreSQL 18.6 pgBackRest Differential & Incremental Backup/Restore on AWS EC2

## 1. Lab Objective

Validate a complete pgBackRest backup chain and prove restore behavior using the `dvdrental` database:

**Full Backup → Differential Backup → Incremental Backup → Differential Restore → Incremental Restore → Write Validation**

## 2. Lab Environment

| Component | pgsql01 | pgsql02 |
|---|---|---|
| Purpose | Backup Source | Restore/Recovery Server |
| Private IP | `172.30.2.177` | `172.30.2.128` |
| Public IP | `18.221.154.4` | `18.220.37.63` |
| Public DNS | `ec2-18-221-154-4.us-east-2.compute.amazonaws.com` | `ec2-18-220-37-63.us-east-2.compute.amazonaws.com` |
| OS | Amazon Linux 2023 | Amazon Linux 2023 |
| PostgreSQL | 18.6 | 18.6 |
| pgBackRest | 2.59.1 | 2.59.1 |

**Database:** `dvdrental`  
**PGDATA:** `/var/lib/pgsql/data`  
**Stanza:** `lab5`  
**Repository:** `/var/lib/pgsql/pgbackrest_repo`

## 3. Backup Chain Used

```text
Full : 20260903-013127F
  |
  +-- Differential : 20260903-013127F_20260903-180115D
        |
        +-- Incremental : 20260903-013127F_20260903-180248I
```

Verified repository output:

```text
full backup: 20260903-013127F
    database backup size: 45.9MB
    repo1 backup size: 6.7MB

diff backup: 20260903-013127F_20260903-180115D
    database backup size: 1.9MB
    repo1 backup size: 226.3KB
    backup reference total: 1 full

incr backup: 20260903-013127F_20260903-180248I
    database backup size: 60.1KB
    repo1 backup size: 3KB
    backup reference total: 1 full, 1 diff
```

---

# PART A — BACKUP ON pgsql01

## Step 1 — Validate pgBackRest

```bash
sudo -u postgres pgbackrest --stanza=lab5 check
```

Expected:

```text
WAL segment ... successfully archived
check command end: completed successfully
```

## Step 2 — Create Validation Table

```bash
sudo -u postgres psql -d dvdrental -c "
CREATE TABLE IF NOT EXISTS lab6_backup_validation
(
    id           bigserial PRIMARY KEY,
    backup_stage varchar(30) NOT NULL,
    test_note    text NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now()
);
"
```

Output:

```text
CREATE TABLE
```

## Step 3 — Insert Change Set A

```bash
sudo -u postgres psql -d dvdrental -c "
INSERT INTO lab6_backup_validation
    (backup_stage, test_note)
VALUES
    ('DIFFERENTIAL', 'LAB-6 Change Set A - Row 1'),
    ('DIFFERENTIAL', 'LAB-6 Change Set A - Row 2'),
    ('DIFFERENTIAL', 'LAB-6 Change Set A - Row 3');
"
```

Output:

```text
INSERT 0 3
```

## Step 4 — Switch WAL

```bash
sudo -u postgres psql -d postgres -c "
SELECT pg_switch_wal();
"
```

Validated WAL archive:

```text
last_archived_wal                 failed_count
000000030000000000000013          0
```

## Step 5 — Take Differential Backup

```bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--type=diff \
backup
```

Successful result:

```text
new backup label = 20260903-013127F_20260903-180115D
diff backup size = 1.9MB
file total = 1668
backup command end: completed successfully
```

## Step 6 — Insert Change Set B

```bash
sudo -u postgres psql -d dvdrental -c "
INSERT INTO lab6_backup_validation
    (backup_stage, test_note)
VALUES
    ('INCREMENTAL', 'LAB-6 Change Set B - Row 1'),
    ('INCREMENTAL', 'LAB-6 Change Set B - Row 2'),
    ('INCREMENTAL', 'LAB-6 Change Set B - Row 3');
"
```

Output:

```text
INSERT 0 3
```

Validate:

```bash
sudo -u postgres psql -d dvdrental -c "
SELECT backup_stage, COUNT(*) AS row_count
FROM lab6_backup_validation
GROUP BY backup_stage
ORDER BY backup_stage;
"
```

Output:

```text
 backup_stage | row_count
--------------+----------
 DIFFERENTIAL | 3
 INCREMENTAL  | 3
```

## Step 7 — Switch WAL and Take Incremental Backup

```bash
sudo -u postgres psql -d postgres -c "
SELECT pg_switch_wal();
"
```

```bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--type=incr \
backup
```

Successful result:

```text
new backup label = 20260903-013127F_20260903-180248I
incr backup size = 60.1KB
file total = 1668
backup command end: completed successfully
```

## Step 8 — Verify Complete Backup Chain

```bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
info
```

Expected chain:

```text
Full         : 20260903-013127F
Differential : 20260903-013127F_20260903-180115D
Incremental  : 20260903-013127F_20260903-180248I
Status       : ok
```

---

# PART B — TRANSFER REPOSITORY TO pgsql02

## Step 9 — Package Repository on pgsql01

```bash
sudo tar -czf \
/tmp/lab6_pgbackrest_repo.tar.gz \
-C /var/lib/pgsql \
pgbackrest_repo

sudo chown ec2-user:ec2-user \
/tmp/lab6_pgbackrest_repo.tar.gz
```

Validated archive:

```text
/tmp/lab6_pgbackrest_repo.tar.gz
Size: approximately 9.8M
```

## Step 10 — Transfer Through Windows

Download from `pgsql01`:

```powershell
scp -i "C:\Users\DELL\Downloads\pgsqllabssep2026.pem" `
ec2-user@ec2-18-221-154-4.us-east-2.compute.amazonaws.com:/tmp/lab6_pgbackrest_repo.tar.gz `
"C:\Users\DELL\Downloads\lab6_pgbackrest_repo.tar.gz"
```

Validated Windows file:

```text
Length: 10171651 bytes
Name  : lab6_pgbackrest_repo.tar.gz
```

Upload to `pgsql02`:

```powershell
scp -i "C:\Users\DELL\Downloads\pgsqllabssep2026.pem" `
"C:\Users\DELL\Downloads\lab6_pgbackrest_repo.tar.gz" `
ec2-user@ec2-18-220-37-63.us-east-2.compute.amazonaws.com:/tmp/lab6_pgbackrest_repo.tar.gz
```

Output:

```text
lab6_pgbackrest_repo.tar.gz  100%
```

## Step 11 — Prepare pgsql02 and Extract Repository

```bash
sudo systemctl stop postgresql
sudo systemctl is-active postgresql
```

Output:

```text
inactive
```

Preserve the existing cluster:

```bash
sudo mv \
/var/lib/pgsql/data \
/var/lib/pgsql/data_before_lab6_restore_20260903
```

Install the transferred repository:

```bash
sudo rm -rf /var/lib/pgsql/pgbackrest_repo

sudo tar -xzf \
/tmp/lab6_pgbackrest_repo.tar.gz \
-C /var/lib/pgsql

sudo chown -R postgres:postgres \
/var/lib/pgsql/pgbackrest_repo

sudo chmod 750 \
/var/lib/pgsql/pgbackrest_repo
```

Validate:

```bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
info
```

Expected:

```text
stanza: lab5
status: ok

full : 20260903-013127F
diff : 20260903-013127F_20260903-180115D
incr : 20260903-013127F_20260903-180248I
```

---

# PART C — DIFFERENTIAL RESTORE

## Step 12 — Prepare Empty PGDATA

```bash
sudo mkdir -p /var/lib/pgsql/data
sudo chown postgres:postgres /var/lib/pgsql/data
sudo chmod 700 /var/lib/pgsql/data
```

## Step 13 — Restore Differential Backup

```bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--set=20260903-013127F_20260903-180115D \
--type=immediate \
restore
```

Successful result:

```text
restore backup set 20260903-013127F_20260903-180115D
restore size = 45.9MB
file total = 1668
restore command end: completed successfully
```

Recovery configuration:

```text
restore_command = 'pgbackrest --stanza=lab5 archive-get %f "%p"'
recovery_target = 'immediate'
recovery_target_timeline = 'current'
```

## Step 14 — Start and Promote Differential Restore

```bash
sudo systemctl start postgresql
sudo systemctl is-active postgresql
```

Output:

```text
active
```

Check recovery:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS in_recovery,
    current_setting('transaction_read_only') AS transaction_read_only,
    pg_is_wal_replay_paused() AS wal_replay_paused;
"
```

Expected:

```text
in_recovery | transaction_read_only | wal_replay_paused
t           | on                    | t
```

Promote:

```bash
sudo -u postgres psql -d postgres -c "
SELECT pg_promote(
    wait => true,
    wait_seconds => 60
);
"
```

Output:

```text
pg_promote
----------
t
```

## Step 15 — Validate Differential Restore

```bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    backup_stage,
    COUNT(*) AS row_count
FROM lab6_backup_validation
GROUP BY backup_stage
ORDER BY backup_stage;
"
```

Validated output:

```text
 backup_stage | row_count
--------------+----------
 DIFFERENTIAL | 3
```

**Result:** Change Set A is present and Change Set B is not present.

---

# PART D — INCREMENTAL RESTORE

## Step 16 — Preserve Differential Restore and Prepare PGDATA

```bash
sudo systemctl stop postgresql

sudo mv \
/var/lib/pgsql/data \
/var/lib/pgsql/data_after_lab6_diff_restore_20260903

sudo mkdir -p /var/lib/pgsql/data
sudo chown postgres:postgres /var/lib/pgsql/data
sudo chmod 700 /var/lib/pgsql/data
```

Validation:

```bash
sudo ls -ld /var/lib/pgsql/data
```

Output:

```text
drwx------ postgres postgres /var/lib/pgsql/data
```

## Step 17 — Restore Incremental Backup

```bash
sudo -u postgres pgbackrest \
--stanza=lab5 \
--set=20260903-013127F_20260903-180248I \
--type=immediate \
restore
```

Successful result:

```text
restore backup set 20260903-013127F_20260903-180248I
restore size = 45.9MB
file total = 1668
restore command end: completed successfully
```

## Step 18 — Start PostgreSQL and Validate Recovery

```bash
sudo systemctl start postgresql
sudo systemctl is-active postgresql
```

Output:

```text
active
```

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS in_recovery,
    current_setting('transaction_read_only') AS transaction_read_only,
    pg_is_wal_replay_paused() AS wal_replay_paused;
"
```

Validated output:

```text
in_recovery | transaction_read_only | wal_replay_paused
t           | on                    | t
```

## Step 19 — Promote Incremental Restore

```bash
sudo -u postgres psql -d postgres -c "
SELECT pg_promote(
    wait => true,
    wait_seconds => 60
);
"
```

Output:

```text
pg_promote
----------
t
```

Validate:

```bash
sudo -u postgres psql -d postgres -c "
SELECT
    pg_is_in_recovery() AS in_recovery,
    current_setting('transaction_read_only') AS transaction_read_only;
"
```

Output:

```text
in_recovery | transaction_read_only
f           | off
```

## Step 20 — Validate Incremental Restore

```bash
sudo -u postgres psql -d dvdrental -c "
SELECT
    backup_stage,
    COUNT(*) AS row_count
FROM lab6_backup_validation
GROUP BY backup_stage
ORDER BY backup_stage;
"
```

Validated output:

```text
 backup_stage | row_count
--------------+----------
 DIFFERENTIAL | 3
 INCREMENTAL  | 3
```

**Result:** Change Set A and Change Set B are both present.

---

# PART E — FINAL WRITE TEST

## Step 21 — Verify Restored Database Is Writable

```bash
sudo -u postgres psql -d dvdrental -c "
INSERT INTO lab6_backup_validation
    (backup_stage, test_note)
VALUES
    ('POST_RESTORE_WRITE',
     'LAB-6 Final write validation after incremental restore');

SELECT
    id,
    backup_stage,
    test_note
FROM lab6_backup_validation
ORDER BY id;
"
```

Validated final result:

```text
1  DIFFERENTIAL        LAB-6 Change Set A - Row 1
2  DIFFERENTIAL        LAB-6 Change Set A - Row 2
3  DIFFERENTIAL        LAB-6 Change Set A - Row 3
4  INCREMENTAL         LAB-6 Change Set B - Row 1
5  INCREMENTAL         LAB-6 Change Set B - Row 2
6  INCREMENTAL         LAB-6 Change Set B - Row 3
7  POST_RESTORE_WRITE  LAB-6 Final write validation after incremental restore
```

# Final Validation

| Test | Result |
|---|---|
| Existing full backup available | PASS |
| Differential backup | PASS |
| Incremental backup | PASS |
| Repository transferred to pgsql02 | PASS |
| Differential restore | PASS |
| Only Change Set A after differential restore | PASS |
| Incremental restore | PASS |
| Change Sets A + B after incremental restore | PASS |
| Promotion after recovery | PASS |
| Post-restore write | PASS |

## LAB-6 Status

**COMPLETED — PASS**

The tested backup and restore chain was:

```text
FULL
  ↓
DIFFERENTIAL
  ↓
INCREMENTAL
  ↓
DIFFERENTIAL RESTORE
  ↓
INCREMENTAL RESTORE
  ↓
WRITE VALIDATION
```

> Important: For this lab, `--type=immediate` is intentionally used with the exact `--set` label so recovery stops at the selected backup's consistent recovery point before promotion.
