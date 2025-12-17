# Database Replication Commands

This document contains a list of SQL commands that are commonly used to configure and monitor database replication. Each command is accompanied by a brief explanation.

## Replication Variables and Logs

- **Show Server ID:**
  ```sql
  SHOW variables LIKE 'server_id';
  ```
  Displays the unique identifier for the server in the replication setup.

- **Show Binary Logs:**
  ```sql
  SHOW BINARY LOGS;
  ```
  Lists the binary log files on the server.

- **Show Master Status:**
  ```sql
  SHOW MASTER STATUS;
  ```
  Provides the current binary log file and position of the master server.

- **Show Binary Log Format:**
  ```sql
  SHOW variables LIKE 'binlog_format';
  ```
  Displays the format used for binary logging (ROW, STATEMENT, or MIXED).

- **Show GTID Mode:**
  ```sql
  SHOW variables LIKE 'gtid_mode';
  ```
  Indicates whether GTID (Global Transaction Identifiers) is enabled.

- **Show Global GTID Executed:**
  ```sql
  SELECT @@GLOBAL.GTID_EXECUTED;
  ```
  Displays the set of GTIDs executed on the server.

- **Enforce GTID Consistency:**
  ```sql
  SHOW VARIABLES LIKE 'enforce_gtid_consistency';
  ```
  Checks if GTID consistency is enforced.

## Authentication and Parallel Replication

- **Default Authentication Plugin:**
  ```sql
  SHOW VARIABLES LIKE 'default_authentication_plugin';
  ```
  Shows the default authentication plugin used by the server.

- **Slave Parallel Type:**
  ```sql
  SHOW VARIABLES LIKE 'slave_parallel_type';
  ```
  Displays the type of parallel replication used (LOGICAL_CLOCK or DATABASE).

- **Slave Parallel Workers:**
  ```sql
  SHOW variables LIKE 'slave_parallel_workers';
  ```
  Indicates the number of parallel workers for slave replication.

- **Binlog Transaction Dependency Tracking:**
  ```sql
  SHOW variables LIKE 'binlog_transaction_dependency_tracking';
  ```
  Shows how transaction dependencies are tracked in binary logs.

## InnoDB and Performance Settings

- **InnoDB Buffer Pool Size:**
  ```sql
  SHOW variables LIKE 'innodb_buffer_pool_size';
  ```
  Displays the size of the InnoDB buffer pool.

- **InnoDB Status:**
  ```sql
  SHOW ENGINE INNODB STATUS;
  ```
  Provides detailed status information about the InnoDB storage engine.

- **Read-Only Mode:**
  ```sql
  SHOW variables LIKE 'read_only';
  ```
  Indicates whether the server is in read-only mode.

## Connection and Logging Settings

- **Bind Address:**
  ```sql
  SHOW variables LIKE 'bind_address';
  ```
  Displays the IP address that the server is bound to.

- **Max Connections:**
  ```sql
  SHOW variables LIKE 'max_connections';
  ```
  Indicates the maximum number of simultaneous client connections.

- **Threads Connected:**
  ```sql
  SHOW status LIKE 'Threads_connected';
  ```
  Shows the number of currently open connections.

- **Expire Logs Days:**
  ```sql
  SHOW variables LIKE 'expire_logs_days';
  ```
  Displays the number of days binary logs are retained.

- **Sync Binlog Setting:**
  ```sql
  SHOW variables LIKE 'sync_binlog';
  ```
  Indicates how often the binary log is synchronized to disk.

- **InnoDB Flush Log at Transaction Commit:**
  ```sql
  SHOW variables LIKE 'innodb_flush_log_at_trx_commit';
  ```
  Displays the InnoDB flush log setting for transaction commits.

- **InnoDB File Per Table:**
  ```sql
  SHOW variables LIKE 'innodb_file_per_table';
  ```
  Indicates whether InnoDB uses separate tablespaces for each table.

- **Skip Name Resolve:**
  ```sql
  SHOW VARIABLES LIKE 'skip_name_resolve';
  ```
  Checks if DNS resolution is skipped when authenticating clients.

- **SQL Mode:**
  ```sql
  SHOW VARIABLES LIKE 'sql_mode';
  ```
  Displays the current SQL mode settings.

---

Feel free to modify these commands or add more as necessary to tailor this document to your specific replication setup.