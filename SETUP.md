# MySQL Server Setup & Tuning Guide

This document collects recommended MySQL settings and operational guidance for a dedicated database server. It provides a sensible starting point for production systems, explains why each setting matters, and includes checks and maintenance commands. Use this as a reference and adjust values to your hardware, workload, and risk profile.

> Note: Always test configuration changes in a staging environment and take backups before applying changes to production.

---

## Table of contents
- Overview
- Quick start — apply this config
- Recommended my.cnf (example)
- Explanation of key settings
- Comprehensive MySQL Configuration Analysis
- Workload-specific suggestions
- Monitoring & maintenance
- Pre-apply checklist & validation
- Hardware recommendations
- Troubleshooting tips
- References
- Contributing / License

---

## Overview
This guide favors InnoDB as the storage engine (ACID-compliant, crash recovery, row-level locking) and UTF8MB4 for full Unicode support. It emphasizes appropriate buffer pool sizing, safe binary logging, and practical monitoring and maintenance steps.

---

## Quick start — apply this config
1. Copy the example configuration below to your MySQL configuration file (commonly `/etc/my.cnf` or `/etc/mysql/my.cnf`).
2. Adjust values to match your server resources (especially memory settings).
3. Validate the configuration:
   - `mysqld --verbose --help` (prints compiled-in defaults)
   - `mysqld --defaults-file=/etc/my.cnf --validate-config`
4. Restart MySQL: `systemctl restart mysqld` (or equivalent)
5. Monitor logs for errors and check `SHOW ENGINE INNODB STATUS\G`.

---

## Recommended my.cnf (example)

```ini
[mysqld]
# Storage engine & charset
default_storage_engine = InnoDB
disabled_storage_engines = MyISAM,BLACKHOLE,ARCHIVE

character_set_server = utf8mb4
collation_server = utf8mb4_unicode_ci

# InnoDB basics
innodb_file_per_table = 1                   # enables better space management
innodb_data_file_path = ibdata1:1G:autoextend

# Log files
innodb_log_file_size = 2G                   # increase for high-write workloads
innodb_log_files_in_group = 2
innodb_log_buffer_size = 64M

# I/O and durability
innodb_flush_method = O_DIRECT              # bypass OS cache to avoid double caching
innodb_flush_log_at_trx_commit = 1          # 1 = safest (ACID), 2 = faster, less safe
innodb_doublewrite = 1
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# Connections & security
skip_name_resolve = 1                       # disable DNS lookups for client auth
max_connections = 200
max_connect_errors = 1000000
wait_timeout = 600
interactive_timeout = 600

# Buffer pool (critical)
innodb_buffer_pool_size = 24G               # set to ~50-75% of available RAM on dedicated DB
innodb_buffer_pool_instances = 8
innodb_buffer_pool_chunk_size = 128M
innodb_buffer_pool_dump_at_shutdown = 1
innodb_buffer_pool_load_at_startup = 1

# Binary logging (replication & PITR)
log_bin = /var/lib/mysql/mysql-bin
binlog_format = ROW
binlog_row_image = minimal
expire_logs_days = 7
max_binlog_size = 256M
sync_binlog = 1
binlog_cache_size = 32K

# Monitoring & limits
performance_schema = 1
max_allowed_packet = 256M
max_execution_time = 30000                  # milliseconds

# Slow query logging
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 2
log_queries_not_using_indexes = 0
min_examined_row_limit = 100
```

---

## Explanation of key settings
- `default_storage_engine = InnoDB`: InnoDB provides ACID guarantees, crash recovery and better concurrency.
- `utf8mb4`: Full Unicode support (including emojis).
- `innodb_file_per_table = 1`: Each table gets its own tablespace (easier reclaiming of space and backups).
- `innodb_buffer_pool_size`: Caches data and indexes — the single most important tuning knob for InnoDB performance. For a dedicated DB server aim for 50–75% of total RAM.
- `innodb_buffer_pool_instances`: Split buffer pool to reduce contention on multi-core systems (use more instances with larger buffer pools).
- `innodb_flush_log_at_trx_commit`: `1` ensures full ACID durability (slower). Use `2` only if you can accept some transactional risk for higher throughput.
- `innodb_log_file_size`: Larger log files can improve write performance, but increase recovery time; tune according to write workload.
- `sync_binlog = 1`: Ensures binary logs are flushed to disk on transaction commit (important for crash-safe replication and PITR).
- `binlog_format = ROW` + `binlog_row_image = minimal`: Safer for replication and more consistent row-based replication.

---

## Comprehensive MySQL Configuration Analysis
Below is a detailed breakdown of each configuration option and the trade-offs between choices.

### 1. Storage Engine & Charset
- `default_storage_engine = InnoDB`
  - What it does: Sets InnoDB as the default storage engine for new tables.
  - Alternative choices:
    - MyISAM: No transactions, table-level locking, faster for reads only
    - MEMORY: In-memory tables (volatile)
    - ARCHIVE: Compressed storage for archival
  - Why InnoDB: ACID compliance, row-level locking, crash recovery, foreign key support

- `disabled_storage_engines = MyISAM,BLACKHOLE,ARCHIVE`
  - What it does: Prevents creation of tables using these engines.
  - Impact: Prevents accidental use of non-transactional engines and forces InnoDB usage.
  - Alternative: Keep enabled for compatibility with legacy apps.

- `character_set_server = utf8mb4` vs `utf8`
  - Key difference: `utf8` in MySQL supports up to 3 bytes per character (BMP only); `utf8mb4` supports 4 bytes (full Unicode including emojis).
  - Historical context: MySQL's `utf8` was implemented before Unicode finalized 4-byte characters.
  - Recommendation: Always use `utf8mb4` in modern applications.

### 2. InnoDB Configuration
- `innodb_file_per_table = 1`
  - What it does: Creates separate `.ibd` files for each table instead of storing all data in `ibdata1`.
  - Advantages: Easier disk space reclamation after DROP TABLE; can run `OPTIMIZE TABLE` to reclaim space; better tablespace management.
  - Disadvantages: More file descriptors needed; `DROP TABLE` can be slower (file deletion).
  - Alternative (`=0`): All tables in system tablespace (legacy mode).

- `innodb_log_file_size = 2G`
  - What it does: Size of each redo log file. Total redo log space = size × number of files.
  - Performance impact: Small size (e.g., 128M) -> frequent checkpointing and more I/O but faster recovery; large size (e.g., 4G) -> less checkpointing, better write performance, slower recovery.
  - Rule of thumb: Set to hold 1–2 hours of writes during peak. Monitor `Innodb_os_log_written` to calculate optimal size.

- `innodb_flush_method = O_DIRECT`
  - Available options and impact:

```text
O_DIRECT:
├── Pros: No double caching, more memory for buffer pool
├── Cons: Requires properly aligned I/O
└── Best for: Dedicated DB servers

fsync:
├── Pros: Works well with mixed workloads
├── Cons: Memory wasted in OS cache
└── Best for: Shared servers, virtualization
```

- `innodb_flush_log_at_trx_commit = 1` vs `2` vs `0`
  - Comparison:

```text
Value   Durability          Performance         Use Case
1       Full ACID           Slowest             Financial transactions, must survive crash
2       Survive OS crash only  Medium           Can lose last ~1s of data if MySQL crashes
0       Periodic flush (~1s)  Fastest           Batch processing, analytics
```
Safety vs Performance visual:

```text
Safety <-------------------> Performance
    1               2               0
    │               │               │
Must survive   Can lose     Can lose
power loss      MySQL       OS crash
              crash only
```

### 3. Buffer Pool Configuration
- `innodb_buffer_pool_size = 24G`
  - Calculation formula for dedicated server example:

```text
Total RAM: 32G
OS/MySQL overhead: ~4G
Buffer pool: 24G (75% of RAM)
Remaining: 4G for connections, temp tables, etc.
```

- Too small: Constant disk I/O, poor performance. Too large: Swapping, OOM kills.

- Monitoring example:

```sql
-- Check buffer pool efficiency
SELECT 
    (1 - ((Variable_value / 
        (SELECT Variable_value 
         FROM information_schema.global_status 
         WHERE Variable_name = 'Innodb_buffer_pool_read_requests'))
    )) * 100 AS 'Buffer Pool Hit Rate'
FROM information_schema.global_status 
WHERE Variable_name = 'Innodb_buffer_pool_reads';
-- Aim for >99%
```

- `innodb_buffer_pool_instances = 8`
  - What it does: Splits buffer pool into multiple instances to reduce contention.
  - Rule of thumb: 1 instance per GB of buffer pool (up to 64). Must be power of 2. Each instance should be at least 1GB.

### 4. Binary Logging Configuration
- `binlog_format = ROW` vs `STATEMENT` vs `MIXED`

```text
STATEMENT:
├── Pros: Small logs, human readable
├── Cons: Non-deterministic statements, replication drift
└── Example: RAND(), UUID() differ on replica

ROW:
├── Pros: Replicates exact data changes, safe
├── Cons: Larger logs, not human readable
└── Example: Replicates actual row changes

MIXED:
├── Pros: Attempts to use STATEMENT, falls back to ROW
├── Cons: Complexity, still can have drift
└── Example: Uses STATEMENT for simple updates, ROW for complex
```

- `sync_binlog = 1` vs `0` vs `N`

```text
0: OS decides when to flush (risky, best performance)
1: Flush after each transaction (safest, worst performance)
N: Flush after N transactions (balance)
```

- `binlog_row_image = minimal` vs `full`

```text
minimal: Logs only changed columns and primary key
full: Logs all columns (before and after)
noblob: Like minimal but excludes unchanged BLOB columns
```

Space savings example with `minimal`:

```text
UPDATE users SET last_login = NOW() WHERE id = 1;

full:    Logs all 50 columns of the row
minimal: Logs only: id=1, last_login='2024-01-01 12:00:00'
```

### 5. Connection & Security
- `skip_name_resolve = 1`
  - What it does: Disables DNS lookups for client connections.
  - Impact: Connections use IP addresses only in `mysql.user` table. Faster connections and no dependency on DNS.
  - Requirement: Must use IP addresses in grant statements.

```sql
-- Works: GRANT ALL ON db.* TO 'user'@'192.168.1.%'
-- Fails: GRANT ALL ON db.* TO 'user'@'%.example.com'
```

- `max_connections = 200`
  - Calculation considerations example:

```text
Application connections: 50
Maintenance/backup: 20
Buffer: 30
Monitoring: 10
Replication: 10
Total: ~120

Setting 200 provides headroom for connection spikes
```

- Too high: Each connection consumes memory (~512KB-2MB each).

Monitoring commands:

```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
```

### 6. Performance Monitoring Settings
- `long_query_time = 2`
  - What it does: Log queries taking longer than this value (seconds).
  - Tuning approach: Start with 10 seconds in production and gradually reduce.

- Alternative approaches: Use Performance Schema for deeper analysis.

- `performance_schema = 1`:
  - Memory overhead: ~100MB-1GB depending on configuration.

Key Performance Schema queries:

```sql
-- Top wait events
SELECT * FROM performance_schema.events_waits_summary_global_by_event_name 
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;

-- Statement analysis
SELECT * FROM performance_schema.events_statements_summary_by_digest 
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

### 7. Critical Configuration Comparisons
- Write Performance vs Durability Trade-offs

```text
Setting    Safe Configuration   Performance Configuration  Difference
innodb_flush_log_at_trx_commit  1        2             2-10x faster writes, risk last second data loss
sync_binlog  1        0             3-5x faster binary logging, risk binlog corruption
innodb_doublewrite  1        0             2x faster writes, risk partial page writes
```

- Memory Allocation Strategy (example breakdown):

```text
Your configuration (24G buffer pool):
├── Buffer pool: 24G (75% of RAM)
├── OS cache: ~2G
├── Connections: 200 × 2MB = 400MB
├── Temp tables: 256MB
└── Log buffer: 64MB

Alternative (16G RAM server):
├── Buffer pool: 10G (62.5% of RAM)
├── OS cache: ~3G
├── Connections: 100 × 2MB = 200MB
└── Log buffer: 32MB
```

### 8. Version-Specific Considerations
- MySQL 5.7 vs 8.0 Differences

```text
Feature      MySQL 5.7  MySQL 8.0     Impact
Query Cache  Available (deprecated)    Removed  Set query_cache_size=0 in 5.7
Default charset  latin1    utf8mb4   Your config standardizes
innodb_buffer_pool_dump_at_shutdown    Manual tuning needed  Default ON   Your config ensures persistence
max_execution_time  Not available    Available   Prevents runaway queries
```

- Default Value Comparison

```text
Setting  MySQL 8.0 Default  Your Config  Improvement
innodb_buffer_pool_size  128M    24G    192x larger
innodb_log_file_size  48M    2G    42x larger
max_connections  151    200    33% more
table_open_cache  2000    2000    Same
tmp_table_size  16M    256M    16x larger
```

### 9. Monitoring Queries for Each Setting

```sql
-- Buffer pool efficiency
SHOW ENGINE INNODB STATUS\G
-- Look at "BUFFER POOL AND MEMORY" section

-- Connection effectiveness
SHOW GLOBAL STATUS LIKE 'Aborted_connects';
SHOW GLOBAL STATUS LIKE 'Aborted_clients';

-- I/O capacity utilization
SELECT 
    (innodb_data_reads + innodb_data_writes) / 
    (@@innodb_io_capacity * 60) AS io_utilization
FROM information_schema.global_status
WHERE variable_name IN ('Innodb_data_reads', 'Innodb_data_writes');

-- Binary log sizing
SHOW BINARY LOGS;
PURGE BINARY LOGS BEFORE NOW() - INTERVAL 7 DAY;

-- Table efficiency
SELECT 
    ENGINE,
    COUNT(*) as tables,
    SUM(DATA_LENGTH + INDEX_LENGTH) as total_size,
    SUM(DATA_FREE) as free_space
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema')
GROUP BY ENGINE;
```

### 10. Risk Assessment
- High-Risk Changes (Test First!)
  - `innodb_log_file_size`: Requires MySQL shutdown, backup recommended
  - `innodb_buffer_pool_size`: Too high causes swapping, OOM kills
  - `skip_name_resolve`: Breaks hostname-based grants

- Safe Changes (Dynamic)
  - `max_connections`, `long_query_time`, `innodb_io_capacity`, most timeout settings

- Monitoring After Changes

```bash
# Immediate monitoring after config change
watch -n 1 "mysql -e 'SHOW GLOBAL STATUS LIKE "Innodb_buffer_pool%";'"

# Check for errors
tail -f /var/log/mysql/error.log

# Performance impact
pt-query-digest /var/lib/mysql/slow.log --since=1h
```

### 11. Alternative Configuration Scenarios
- For Cloud/Virtualized Environment

```ini
innodb_flush_method = O_DIRECT_NO_FSYNC
innodb_use_native_aio = 1
innodb_buffer_pool_size = 70% of RAM
performance_schema = 0  # If memory constrained
```

- For Read Replica

```ini
innodb_buffer_pool_size = 80% of RAM
innodb_read_io_threads = 8
read_only = 1
super_read_only = 1
skip_slave_start = 0
```

- For Development/Staging

```ini
innodb_buffer_pool_size = 2G
innodb_flush_log_at_trx_commit = 2
sync_binlog = 0
slow_query_log = 0
performance_schema = 0
```

---

## Workload-specific recommendations
- Read-heavy:
  - Set `innodb_buffer_pool_size` to 70–80% of RAM (if DB is dedicated).
  - `innodb_read_io_threads = 8`, `innodb_write_io_threads = 4` (depending on MySQL version and kernel).
  - Query cache (only applies to MySQL 5.7 and earlier): `query_cache_size = 128M` (use with caution — often better to disable on modern workloads).

- Write-heavy:
  - Consider `innodb_flush_log_at_trx_commit = 2` for improved throughput (trade-off: transactions less durable on crash).
  - Increase `innodb_log_file_size` e.g., 4G (reduces checkpoint pressure).
  - Increase `innodb_io_capacity` / `innodb_io_capacity_max` if underlying storage can sustain it.

- Mixed:
  - `innodb_buffer_pool_size = 60–70% of RAM`
  - `innodb_log_file_size = 2G`
  - `innodb_flush_log_at_trx_commit = 1`

Always benchmark and measure (Sysbench, application load testing).

---

## Monitoring & maintenance

- Enable performance instrumentation:
  - Example: `UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES';`

- Key metrics to watch:
  - Buffer pool hit rate (>99% ideal)
  - Buffer pool usage (pages dirty/free)
  - Connection usage
  - Slow queries & queries not using indexes
  - Lock waits & contention
  - Replication lag (if applicable)
  - Disk I/O latency

- Useful commands:
  - `SHOW ENGINE INNODB STATUS\G`
  - `SHOW GLOBAL STATUS LIKE 'Threads_connected';`
  - `SHOW VARIABLES LIKE 'innodb_buffer_pool_size';`
  - `mysqladmin ext -i 5` (periodic extended status)
  - `mysql -u root -p -e "PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 7 DAY);"`

- Regular maintenance:
  - `mysqlcheck -u root -p --optimize --all-databases` (use during maintenance windows)
  - Rotate and purge binary logs to free disk: `PURGE BINARY LOGS BEFORE ...`

---

## Pre-apply checklist & validation
1. Backup your data (logical and/or physical backup).
2. Test changes on a staging server that mirrors production.
3. Validate config: `mysqld --defaults-file=/etc/my.cnf --validate-config`
4. Monitor error logs on restart (`/var/log/mysqld.log` or `/var/log/mysql/error.log`).
5. Be prepared to restore previous configuration and restart if needed.

---

## Hardware recommendations
- RAM: Minimum 8GB for small deployments; 16GB+ recommended for production. Buffer pool sizing is the primary reason for more RAM.
- Storage: SSDs (NVMe preferred). For durability, RAID 10 is recommended.
- CPU: Multiple cores benefit parallel queries and background tasks.
- Network: Low-latency connections for app ↔ DB; consider placement close to application servers.

---

## Troubleshooting tips
- If MySQL fails to start after changing `innodb_log_file_size`:
  - You must safely remove old InnoDB log files only after shutdown and following documented steps (do not delete without following MySQL docs).
- If buffer pool is not large enough:
  - Increase `innodb_buffer_pool_size` gradually and measure OOM or swap usage.
- If disk I/O is the bottleneck:
  - Check `innodb_io_capacity` / `innodb_io_capacity_max`; review storage performance; consider faster storage or better RAID.

---

## References
- MySQL/InnoDB official docs: https://dev.mysql.com/doc/
- Percona Server and Percona Monitoring and Management (PMM) for advanced monitoring
- MariaDB docs (if using MariaDB)

---

## Contributing / License
- Suggest edits via PRs or open an issue if you find inaccuracies or improvements.
- This document is a recommended starting point — tune per your workload and environment.