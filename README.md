# 📘 Zabbix Upgrade Guide (6 LTS → 7 LTS) with PostgreSQL + TimescaleDB

This document describes the tested steps to upgrade **Zabbix 6 LTS** to
**Zabbix 7 LTS** when using **PostgreSQL + TimescaleDB** as the
backend.\
It covers database restore, TimescaleDB upgrade, foreign key consistency
fixes, and index cleanup.

------------------------------------------------------------------------

## 🚀 1. Restore Database from Backup

1.  Start PostgreSQL after restoring the backup:

    ``` bash
    systemctl start postgresql
    ```

2.  Wait until `psql` accepts connections. Then:

    -   Remove `recovery.signal`
    -   Comment out restore command lines in `postgres.auto.conf`

    Example:

    ``` bash
    rm /var/lib/postgresql/14/main/recovery.signal
    nano /var/lib/postgresql/14/main/postgresql.auto.conf
    ```

3.  Restart PostgreSQL and monitor logs until recovery completes:

    ``` bash
    journalctl -u postgresql -f
    ```

    You should eventually see:

        LOG:  redo done at ...
        LOG:  database system is ready to accept connections

------------------------------------------------------------------------

## 🛠 2. Upgrade TimescaleDB Extension

Once PostgreSQL is up and Zabbix DB is accessible:

``` sql
ALTER EXTENSION timescaledb UPDATE;
```

------------------------------------------------------------------------

## 🔧 3. Fix Missing Foreign Key Constraints

Zabbix DB upgrades may leave orphaned rows in the `items` table (entries
with non-existent `hostid`).\
We ensure referential integrity with the following steps:

### 3.1 Check if constraint exists

``` sql
DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1
    FROM pg_constraint c
    JOIN pg_class t ON c.conrelid = t.oid
    WHERE t.relname = 'items' AND c.conname = 'c_items_1'
  ) THEN
    ALTER TABLE ONLY items
      ADD CONSTRAINT c_items_1
      FOREIGN KEY (hostid)
      REFERENCES hosts(hostid)
      ON DELETE CASCADE
      NOT VALID;
  END IF;
END$$;
```

### 3.2 Find orphaned items

``` sql
SELECT i.hostid, COUNT(*) AS item_count
FROM items i
LEFT JOIN hosts h USING (hostid)
WHERE h.hostid IS NULL
GROUP BY i.hostid
ORDER BY item_count DESC;
```

### 3.3 Clean up orphans

Option 1: Delete them

``` sql
DELETE FROM items
WHERE hostid NOT IN (SELECT hostid FROM hosts);
```

Option 2: Insert dummy hosts (if you need to keep orphaned items):

``` sql
INSERT INTO hosts (hostid, name, host)
VALUES
  (21823, 'dummy_host_21823', 'dummy_host_21823'),
  (21834, 'dummy_host_21834', 'dummy_host_21834');
```

### 3.4 Add missing constraint

``` sql
ALTER TABLE ONLY items
  ADD CONSTRAINT c_items_1
  FOREIGN KEY (hostid)
  REFERENCES hosts (hostid)
  ON DELETE CASCADE;
```

------------------------------------------------------------------------

## 📊 4. Rebuild Indexes (Optional but Recommended)

During schema upgrades, indexes may become bloated. Run:

``` sql
REINDEX DATABASE zabbix;
```

------------------------------------------------------------------------

## ✅ Final Checklist

-   [x] PostgreSQL restored and running without recovery\
-   [x] TimescaleDB extension upgraded\
-   [x] Orphaned `items.hostid` entries resolved\
-   [x] Foreign key `c_items_1` applied\
-   [x] Indexes reindexed

------------------------------------------------------------------------

## 📝 Notes

-   Always back up your DB before applying changes.\
-   For very large Zabbix DBs, cleaning orphans may take hours.\
-   Use dummy hosts **only if deletion is not an option**.\
-   TimescaleDB upgrade is mandatory for Zabbix 7.

------------------------------------------------------------------------

📅 Last tested: **Aug 2025**
