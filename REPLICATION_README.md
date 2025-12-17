# Database Replication Commands

This document contains a list of SQL commands that are commonly used to configure and monitor database replication. Each command is accompanied by a brief explanation.

## Replication Variables and Logs

* **Show Server ID:**

  ```sql
  SHOW variables LIKE 'server_id';
  ```

  Displays the unique identifier for the server in the replication setup.

* **Show Binary Logs:**

  ```sql
  SHOW BINARY LOGS;
  ```

  Lists the binary log files on the server.

* **Show Master Status:**

  ```sql
  SHOW MASTER STATUS;
  ```

  Provides the current binary log file and position of the master server.

* **Show Binary Log Format:**

  ```sql
  SHOW variables LIKE 'binlog_format';
  ```

  Displays the format used for binary logging (ROW, STATEMENT, or MIXED).

* **Show GTID Mode:**

  ```sql
  SHOW variables LIKE 'gtid_mode';
  ```

  Indicates whether GTID (Global Transaction Identifiers) is enabled.

* **Show Global GTID Executed:**

  ```sql
  SELECT @@GLOBAL.GTID_EXECUTED;
  ```

  Displays the set of GTIDs executed on the server.

* **Enforce GTID Consistency:**

  ```sql
  SHOW VARIABLES LIKE 'enforce_gtid_consistency';
  ```

  Checks if GTID consistency is enforced.

## Authentication and Parallel Replication

* **Default Authentication Plugin:**

  ```sql
  SHOW VARIABLES LIKE 'default_authentication_plugin';
  ```

  Shows the default authentication plugin used by the server.

* **Slave Parallel Type:**

  ```sql
  SHOW VARIABLES LIKE 'slave_parallel_type';
  ```

  Displays the type of parallel replication used (LOGICAL_CLOCK or DATABASE).

* **Slave Parallel Workers:**

  ```sql
  SHOW variables LIKE 'slave_parallel_workers';
  ```

  Indicates the number of parallel workers for slave replication.

* **Binlog Transaction Dependency Tracking:**

  ```sql
  SHOW variables LIKE 'binlog_transaction_dependency_tracking';
  ```

  Shows how transaction dependencies are tracked in binary logs.

## InnoDB and Performance Settings

* **InnoDB Buffer Pool Size:**

  ```sql
  SHOW variables LIKE 'innodb_buffer_pool_size';
  ```

  Displays the size of the InnoDB buffer pool.

* **InnoDB Status:**

  ```sql
  SHOW ENGINE INNODB STATUS;
  ```

  Provides detailed status information about the InnoDB storage engine.

* **Read-Only Mode:**

  ```sql
  SHOW variables LIKE 'read_only';
  ```

  Indicates whether the server is in read-only mode.

## Connection and Logging Settings

* **Bind Address:**

  ```sql
  SHOW variables LIKE 'bind_address';
  ```

  Displays the IP address that the server is bound to.

* **Max Connections:**

  ```sql
  SHOW variables LIKE 'max_connections';
  ```

  Indicates the maximum number of simultaneous client connections.

* **Threads Connected:**

  ```sql
  SHOW status LIKE 'Threads_connected';
  ```

  Shows the number of currently open connections.

* **Expire Logs Days:**

  ```sql
  SHOW variables LIKE 'expire_logs_days';
  ```

  Displays the number of days binary logs are retained.

* **Sync Binlog Setting:**

  ```sql
  SHOW variables LIKE 'sync_binlog';
  ```

  Indicates how often the binary log is synchronized to disk.

* **InnoDB Flush Log at Transaction Commit:**

  ```sql
  SHOW variables LIKE 'innodb_flush_log_at_trx_commit';
  ```

  Displays the InnoDB flush log setting for transaction commits.

* **InnoDB File Per Table:**

  ```sql
  SHOW variables LIKE 'innodb_file_per_table';
  ```

  Indicates whether InnoDB uses separate tablespaces for each table.

* **Skip Name Resolve:**

  ```sql
  SHOW VARIABLES LIKE 'skip_name_resolve';
  ```

  Checks if DNS resolution is skipped when authenticating clients.

* **SQL Mode:**

  ```sql
  SHOW VARIABLES LIKE 'sql_mode';
  ```

  Displays the current SQL mode settings.

---

## Replication Server Configuration (my.cnf / Docker)

The following configuration options are commonly used when running MySQL replication, especially in **Docker**, **Kubernetes**, or when defining options in `my.cnf` / `mysqld.cnf`.

```yaml
- --server-id=1
- --log-bin=mysql-bin
- --binlog-format=ROW
- --gtid-mode=ON
- --enforce-gtid-consistency=ON
- --default-authentication-plugin=mysql_native_password
- --slave-parallel-type=LOGICAL_CLOCK
- --slave-parallel-workers=4
- --binlog-transaction-dependency-tracking=WRITESET
- --innodb-buffer-pool-size=256M
- --bind-address=0.0.0.0
- --max_connections=200
- --max_user_connections=50
- --expire_logs_days=7
- --max_binlog_size=100M
- --sync_binlog=1
- --innodb_log_file_size=64M
- --innodb_flush_log_at_trx_commit=1
- --innodb_file_per_table=1
- --skip_name_resolve
- --character-set-server=utf8mb4
- --collation-server=utf8mb4_unicode_ci
- --sql-mode=STRICT_TRANS_TABLE
```

### Configuration Explanation

* **server-id**: Unique identifier for each MySQL server in the replication topology. Must be different on every node.

* **log-bin**: Enables binary logging, which is mandatory for replication.

* **binlog-format=ROW**: Ensures row-based replication for better consistency and fewer edge cases.

* **gtid-mode=ON**: Enables GTID-based replication for easier failover and recovery.

* **enforce-gtid-consistency=ON**: Prevents execution of non-GTID-safe statements.

* **default-authentication-plugin=mysql_native_password**: Ensures compatibility with older MySQL clients and libraries.

* **slave-parallel-type=LOGICAL_CLOCK**: Enables parallel replication based on transaction dependency tracking.

* **slave-parallel-workers=4**: Number of parallel worker threads applying replication events.

* **binlog-transaction-dependency-tracking=WRITESET**: Improves parallel replication by tracking row-level dependencies.

* **innodb-buffer-pool-size=256M**: Memory allocated for caching InnoDB data and indexes.

* **bind-address=0.0.0.0**: Allows MySQL to accept connections from any network interface.

* **max_connections=200**: Maximum total client connections allowed.

* **max_user_connections=50**: Limits concurrent connections per user to prevent abuse.

* **expire_logs_days=7**: Automatically purges binary logs older than 7 days.

* **max_binlog_size=100M**: Maximum size of a single binary log file.

* **sync_binlog=1**: Forces binary log synchronization to disk on every transaction (safest, slower).

* **innodb_log_file_size=64M**: Size of each InnoDB redo log file.

* **innodb_flush_log_at_trx_commit=1**: Flushes logs at every transaction commit for maximum durability.

* **innodb_file_per_table=1**: Stores each table in its own tablespace for easier management.

* **skip_name_resolve**: Disables DNS lookups during authentication to improve connection performance.

* **character-set-server=utf8mb4**: Default character set supporting full Unicode (including emojis).

* **collation-server=utf8mb4_unicode_ci**: Default collation for Unicode-aware comparisons.

* **sql-mode=STRICT_TRANS_TABLE**: Enforces strict SQL validation to prevent invalid data inserts.

---

Feel free to adjust these settings based on workload, hardware capacity, and replication topology (single-primary, multi-replica, or multi-primary).
