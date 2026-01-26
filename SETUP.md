# MySQL Server Setup & Tuning Guide

This document collects recommended MySQL (MariaDB-compatible) settings and operational guidance for a dedicated database server. It provides a sensible starting point for production systems, explains why each setting matters, and includes quick checks and maintenance commands. Use this as a reference and adjust values to your hardware, workload, and risk profile.

> Note: Always test configuration changes in a staging environment and take backups before applying changes to production.

---

## Table of contents
- Overview
- Quick start — apply this config
- Recommended my.cnf (example)
- Explanation of key settings
- Workload-specific suggestions
- Monitoring & maintenance
- Pre-apply checklist & validation
- Hardware recommendations
- Troubleshooting tips
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
- default_storage_engine = InnoDB: InnoDB provides ACID guarantees, crash recovery and better concurrency.
- utf8mb4: Full Unicode support (including emojis).
- innodb_file_per_table = 1: Each table gets its own tablespace (easier reclaiming of space and backups).
- innodb_buffer_pool_size: Caches data and indexes — the single most important tuning knob for InnoDB performance. For a dedicated DB server aim for 50–75% of total RAM.
- innodb_buffer_pool_instances: Split buffer pool to reduce contention on multi-core systems (use more instances with larger buffer pools).
- innodb_flush_log_at_trx_commit: 1 ensures full ACID durability (slower). Use 2 only if you can accept some transactional risk for higher throughput.
- innodb_log_file_size: Larger log files can improve write performance, but increase recovery time; tune according to write workload.
- sync_binlog = 1: Ensures binary logs are flushed to disk on transaction commit (important for crash-safe replication and PITR).
- binlog_format = ROW + binlog_row_image = minimal: Safer for replication and more consistent row-based replication.

---

## Workload-specific recommendations
- Read-heavy:
  - Set innodb_buffer_pool_size to 70–80% of RAM (if DB is dedicated).
  - innodb_read_io_threads = 8, innodb_write_io_threads = 4 (depending on MySQL version and kernel).
  - Query cache (only applies to MySQL 5.7 and earlier): `query_cache_size = 128M` (use with caution — often better to disable on modern workloads).

- Write-heavy:
  - Consider innodb_flush_log_at_trx_commit = 2 for improved throughput (trade-off: transactions less durable on crash).
  - Increase innodb_log_file_size e.g., 4G (reduces checkpoint pressure).
  - Increase innodb_io_capacity / innodb_io_capacity_max if underlying storage can sustain it.

- Mixed:
  - innodb_buffer_pool_size = 60–70% of RAM
  - innodb_log_file_size = 2G
  - innodb_flush_log_at_trx_commit = 1

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
- If MySQL fails to start after changing innodb_log_file_size:
  - You must safely remove old InnoDB log files only after shutdown and following documented steps (do not delete without following MySQL docs).
- If buffer pool is not large enough:
  - Increase innodb_buffer_pool_size gradually and measure OOM or swap usage.
- If disk I/O is the bottleneck:
  - Check innodb_io_capacity / max; review storage performance; consider faster storage or better RAID.

---

## References
- MySQL/InnoDB official docs: https://dev.mysql.com/doc/
- Percona Server and Percona Monitoring and Management (PMM) for advanced monitoring
- MariaDB docs (if using MariaDB)

---

## Contributing / License
- Suggest edits via PRs or open an issue if you find inaccuracies or improvements.
- This document is a recommended starting point — tune per your workload and environment.
