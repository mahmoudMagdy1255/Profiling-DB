Mysql Setup 

Storage Engine

[mysqld]
# Use InnoDB as default storage engine
default_storage_engine = InnoDB
# Disable other storage engines to prevent accidental usage
disabled_storage_engines = MyISAM,BLACKHOLE,ARCHIVE

# Character set - UTF8MB4 supports full Unicode including emojis
character_set_server = utf8mb4
collation_server = utf8mb4_unicode_ci

Why: InnoDB provides ACID compliance, crash recovery, and row-level locking. UTF8MB4 ensures proper international character support.

InnoDB Settings

# InnoDB file per table (easier management, recovery)
innodb_file_per_table = 1 //  enables better space management
innodb_data_file_path = ibdata1:1G:autoextend

# Log files size - larger = better performance but longer recovery
innodb_log_file_size = 2G // affects write performance (2G is good for high-write systems)
innodb_log_files_in_group = 2
innodb_log_buffer_size = 64M

# I/O settings
innodb_flush_method = O_DIRECT bypasses OS cache (prevents double caching)
innodb_flush_log_at_trx_commit = 1 ensures ACID compliance (sacrifices some performance for safety)
innodb_doublewrite = 1
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

Connection & Security Settings

# Security: Don't resolve hostnames (faster, more secure)
skip_name_resolve = 1 // improves connection speed and security

# Connection settings
max_connections = 200 // should be set based on your application's needs
max_connect_errors = 1000000
wait_timeout = 600 // Timeouts prevent idle connections from consuming resources
interactive_timeout = 600


Buffer Pool Configuration (Most Critical for Performance)

# InnoDB Buffer Pool - This is the most important setting
# Should be 50-75% of total RAM for dedicated DB server
innodb_buffer_pool_size = 24G  # For 32GB RAM system
innodb_buffer_pool_instances = 8  # Reduces contention
innodb_buffer_pool_chunk_size = 128M

# Buffer pool behavior
innodb_buffer_pool_dump_at_shutdown = 1
innodb_buffer_pool_load_at_startup = 1

Why: The buffer pool caches data and indexes in memory. Multiple instances reduce contention on busy systems. Dump/load settings preserve cache across restarts.


Binary Logging & Replication
# Binary logging (for replication and point-in-time recovery)
log_bin = /var/lib/mysql/mysql-bin
binlog_format = ROW
binlog_row_image = minimal
expire_logs_days = 7
max_binlog_size = 256M
sync_binlog = 1
binlog_cache_size = 32K

Why:

ROW format is safer for replication

sync_binlog = 1 ensures crash safety

expire_logs_days manages disk space

Monitoring & Limits

# Performance schema (monitoring)
performance_schema = 1

# Limits to prevent runaway queries
max_allowed_packet = 256M
max_execution_time = 30000  # milliseconds

# Slow query logging
slow_query_log = 1
slow_query_log_file = /var/lib/mysql/slow.log
long_query_time = 2
log_queries_not_using_indexes = 0
min_examined_row_limit = 100


Important Additional Considerations
A. Hardware Considerations:
RAM: Minimum 8GB, ideally 16GB+ for production

Storage: Use SSDs (NVMe preferred), RAID 10 for data safety

CPU: Multiple cores benefit from parallel operations

B. Monitoring Setup:
sql
-- Enable performance schema instruments
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES';

-- Key metrics to monitor regularly:
-- 1. Buffer pool hit rate (>99% ideal)
-- 2. Connection usage
-- 3. Slow queries
-- 4. Lock waits
-- 5. Replication lag (if using replication)
C. Maintenance Scripts:
bash
# Regular maintenance
mysqlcheck -u root -p --optimize --all-databases
mysql -u root -p -e "PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 7 DAY);"
10. Configuration Validation
Before applying:

Test configuration: mysqld --verbose --help

Check for errors: mysqld --defaults-file=/etc/my.cnf --validate-config

Monitor after changes: SHOW ENGINE INNODB STATUS\G


For Read-Heavy Workloads:
ini
innodb_buffer_pool_size = 70-80% of RAM
innodb_read_io_threads = 8
innodb_write_io_threads = 4
query_cache_size = 128M  # Only for MySQL 5.7 and earlier
For Write-Heavy Workloads:
ini
innodb_flush_log_at_trx_commit = 2  # Better performance, less safe
innodb_log_file_size = 4G
innodb_io_capacity = 4000
innodb_io_capacity_max = 8000
For Mixed Workloads:
ini
innodb_buffer_pool_size = 60-70% of RAM
innodb_log_file_size = 2G
innodb_flush_log_at_trx_commit = 1

