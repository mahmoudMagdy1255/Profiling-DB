# MySQL Diagnostics & Performance Toolkit

This README provides a well‑organized collection of MySQL diagnostic, auditing, and performance‑monitoring SQL queries. These scripts help identify slow queries, detect locking issues, analyze storage usage, validate data quality, and understand buffer pool efficiency.

---

## 📌 **1. Data Quality & Null Analysis**

### **Check null vs non‑null distribution in `appointments.subscription_id`**

```sql
SELECT
		appointment_date,
       count(*) as total_rows,
       count(subscription_id) as non_null_count_for_subscriptions,
       (count(*) - count(subscription_id)) as null_count_for_subscriptions,
       round((count(*) - count(subscription_id)) * 100.0 / count(*), 2) as null_percentage_for_subscriptions
from appointments
group by appointment_date;
```

| appointment_date | total_rows | non_null_count_for_subscriptions | null_count_for_subscriptions | null_percentage_for_subscriptions |
|------------------|------------|----------------------------------|-----------------------------|-----------------------------------|
| 2025-11-17       | 13         | 9                                | 4                           | 30.77                             |
| 2025-11-18       | 3          | 0                                | 3                           | 100.00                            |
| 2025-11-19       | 1          | 0                                | 1                           | 100.00                            |
| 2025-11-24       | 2          | 2                                | 0                           | 0.00                              |
| 2025-11-26       | 13         | 4                                | 9                           | 69.23                             |
| 2025-11-27       | 3          | 1                                | 2                           | 66.67                             |
| 2025-12-01       | 2          | 2                                | 0                           | 0.00                              |
| 2025-12-02       | 1          | 0                                | 1                           | 100.00                            |
| 2025-12-04       | 3          | 2                                | 1                           | 33.33                             |
| 2025-12-08       | 1          | 1                                | 0                           | 0.00                              |


### **Validate email format for doctors**

```sql
select
    sum(case when email like '%@%.%' then 1 else 0 end) as valid_email_format,
    sum(case when email not like '%@%.%' then 1 else 0 end) as invalid_email_format,
    count(*) as total
from doctors;
```

| valid_email_format | invalid_email_format | total |
|--------------------|---------------------|-------|
| 27                 | 0                   | 27    |

### **Detect duplicate doctor emails**

```sql
select email, count(*) as duplicate_count
from doctors
group by email
having count(*) > 1;
```
| email                 | duplicate_count |
|-----------------------|----------------|
| m.magdy@wakeb.tech    | 2              |

### **Statistical Distribution Analysis**

```sql

with ordered as (
	select net_amount,
		NTILE(4) OVER (ORDER BY net_amount) AS quartile,
        NTILE(2) OVER (ORDER BY net_amount) AS median_group
		from transactions
),

stats as (
	select avg(net_amount) as avg_value, 
		stddev(net_amount) as std_dev_value, 
        min(net_amount) as min_Value, 
        max(net_amount) as max_value,
		count(*) as total_rows,
		count(distinct net_amount) as distinct_values,

        AVG(CASE WHEN median_group = 1 THEN net_amount END) AS median,

        -- Q1 and Q3 (approx)
        AVG(CASE WHEN quartile = 1 THEN net_amount END) AS q1,
        AVG(CASE WHEN quartile = 3 THEN net_amount END) AS q3

		from ordered
)

select *, (max_value - min_value) as range_values, std_dev_value/avg_value as coefficient_of_variation,
    (q3 - q1) as iqr,
    (avg_value - 3*std_dev_value) as lower_3sigma,
    (avg_value + 3*std_dev_value) as upper_3sigma

from stats;
```
| avg_value  | std_dev_value      | min_Value | max_value | total_rows | distinct_values | median    | q1        | q3        | range_values | coefficient_of_variation | iqr       | lower_3sigma          | upper_3sigma          |
|------------|-------------------|-----------|-----------|------------|-----------------|-----------|-----------|-----------|--------------|--------------------------|-----------|-----------------------|-----------------------|
| 27.739583  | 65.20101047141107 | 0.00      | 300.00    | 96         | 7               | 0.708333  | 0.000000  | 22.500000 | 300.00       | 2.3504682990876637       | 87.041667 | -167.8634484142332    | 223.34261441423322    |
---

## 📌 **2. Process Monitoring & Active Queries**

### **Show current MySQL process list**

```sql
show full processlist;
```

### **Filter non‑sleeping sessions ordered by time**

```sql
select *
from information_schema.processlist
where command != 'Sleep'
order by time desc;
```
| ID    | USER            | HOST             | DB     | COMMAND | TIME (seconds) | TIME (human)                                  | STATE                   | INFO (preview)                                                          |
|-------|-----------------|------------------|--------|---------|----------------|-----------------------------------------------|-------------------------|-------------------------------------------------------------------------|
| 5     | event_scheduler | localhost        | NULL   | Daemon  | 2228708        | 25 days, 19 hours, 5 minutes, 8 seconds       | Waiting on empty queue  | NULL                                                                    |
| 19533 | root            | localhost:49296  | clinic | Query   | 0              | 0 seconds                                     | executing               | select * from information_schema.processlist where command != 'Sleep'… |

---

## 📌 **3. Performance Schema Insight**

### **Top queries related to transactions**

```sql
select digest_text, count_star, sum_timer_wait / 1000000000000 AS sum_timer_wait_seconds
from performance_schema.events_statements_summary_by_digest
where digest_text like '%transactions%'
order by sum_timer_wait desc
limit 10;
```
---
| digest_text | count_star | sum_timer_wait_seconds | 
| --- | ---: | ---: | 
| START TRANSACTION | 90052 | 8.8641 | 
| START TRANSACTION | 1585 | 0.1334 | 
| START TRANSACTION | 883 | 0.1217 | 
| SHOW INDEXES FROM `transactions` FROM `clinic` | 3 | 0.0539 | 
| START TRANSACTION | 318 | 0.0432 | 
| START TRANSACTION | 61 | 0.0115 | 
| WITH `ordered` AS ( SELECT `net_amount` , NTILE (?) OVER ( ORDER BY `net_amount` ) AS `quartile` , NTILE (?) OVER ( ORDER BY `net_amount` ) AS `median_group` FROM `transactions` ) , `stats` AS ( SELECT AVG ( `net_amount` ) AS `avg_value` , STDDEV_POP ( `net_amount` ) AS `std_dev_value` , MIN ( `net_amount` ) AS `min_Value` , MAX ( `net_amount` ) AS `max_value` , COUNT ( * ) AS `total_rows` , COUNT ( DISTINCTROW `net_amount` ) AS `distinct_values` , AVG ( CASE WHEN `median_group` = ? THEN `net_amount` END ) AS `median` , AVG ( CASE WHEN `quartile` = ? THEN `net_amount` END ) AS `q1` , AVG ( CASE WHEN `quartile` = ? THEN `net_amount` END ) AS `q3` FROM `ordered` ) SELECT * , ( `max_value` - `min_value` ) AS `range_values` , `std_dev_value` / `avg_value` AS `coefficient_of_variation` , ( `q3` - `q1` ) AS `iqr` , ( `avg_value` - ? * `std_dev_value` ) AS `lower_3sigma` , ( `avg_value` + ? * `std_dev_value` ) AS `upper_3sigma` FROM `stats` | 4 | 0.0068 | 
| SELECT `net_amount` , NTILE (?) OVER ( ORDER BY `net_amount` ) AS `quartile` , NTILE (?) OVER ( ORDER BY `net_amount` ) AS `median_group` FROM `transactions` | 1 | 0.0037 | 
| SELECT `id` , `transactionable_id` , `transactionable_type` , `amount` , `fee` , `tax` , `status` , `net_amount` , LEFT ( `description` , ? ) , `discount` , `payment_gateway` , `payment_reference_id` , `customer_reference` , `created_at` , `updated_at` FROM `clinic` . `transactions` LIMIT ? | 3 | 0.0037 | 
| SHOW CREATE TABLE `clinic` . `transactions` | 3 | 0.0026 | 


### **High row‑examining queries**

```sql
select digest_text, sum_rows_examined, sum_rows_sent
from performance_schema.events_statements_summary_by_digest
where sum_rows_examined > 10000
order by sum_rows_examined desc
limit 10;
```
---
| digest_text | sum_rows_examined | sum_rows_sent | 
| --- | ---: | ---: | 
| SELECT `id` , `event_id` , LEFT ( `box` , ? ) , LEFT ( `details` , ? ) , `action_type` , `detection_type_id` , `created_at` , `updated_at` FROM `swcc-aware` . `event_detections` ORDER BY `details` ASC LIMIT ? | 44814 | 11000 | 
| SELECT `id` , `event_id` , LEFT ( `box` , ? ) , LEFT ( `details` , ? ) , `action_type` , `detection_type_id` , `created_at` , `updated_at` FROM `swcc-aware` . `event_detections` ORDER BY `details` DESC LIMIT ? | 40740 | 10000 | 
| SELECT `id` , `event_id` , LEFT ( `box` , ? ) , LEFT ( `details` , ? ) , `action_type` , `detection_type_id` , `created_at` , `updated_at` FROM `swcc-aware` . `event_detections` ORDER BY `id` DESC , `details` ASC LIMIT ? | 32592 | 8000 | 
| SELECT `id` , `image` , LEFT ( `heatmap` , ? ) , `notice_time` , `date` , `addition_type` , `location_id` , LEFT ( `meta_geo` , ? ) , `created_by` , `deleted_at` , `created_at` , `updated_at` FROM `swcc-aware` . `events` ORDER BY `meta_geo` DESC LIMIT ? | 27636 | 12000 | 
| SELECT `id` , `event_id` , LEFT ( `box` , ? ) , LEFT ( `details` , ? ) , `action_type` , `detection_type_id` , `created_at` , `updated_at` FROM `swcc-aware` . `event_detections` ORDER BY `id` DESC , `details` DESC LIMIT ? | 24444 | 6000 | 
| SELECT `id` , `image` , LEFT ( `heatmap` , ? ) , `notice_time` , `date` , `addition_type` , `location_id` , LEFT ( `meta_geo` , ? ) , `created_by` , `deleted_at` , `created_at` , `updated_at` FROM `swcc-aware` . `events` ORDER BY `meta_geo` ASC LIMIT ? | 20727 | 9000 | 
| SELECT * FROM `information_schema` . `REFERENTIAL_CONSTRAINTS` WHERE CONSTRAINT_SCHEMA = ? AND TABLE_NAME = ? AND `REFERENCED_TABLE_NAME` IS NOT NULL | 20594 | 172 | 
| SHOW TABLE STATUS FROM `clinic` | 19042 | 4745 | 
| SELECT `id` , `event_id` , LEFT ( `box` , ? ) , LEFT ( `details` , ? ) , `action_type` , `detection_type_id` , `created_at` , `updated_at` FROM `swcc-aware` . `event_detections` LIMIT ? | 17000 | 17000 | 
| SELECT `id` , `image` , LEFT ( `heatmap` , ? ) , `notice_time` , `date` , `addition_type` , `location_id` , LEFT ( `meta_geo` , ? ) , `created_by` , `deleted_at` , `created_at` , `updated_at` FROM `swcc-aware` . `events` LIMIT ? | 13000 | 13000 | 


---

## 📌 **4. Slow Query & Runtime Diagnostics**

### **Top 10 slowest queries**

```sql
select
    left(digest_text, 100) as query_preview,
    count_star as runs,
    round(sum_timer_wait/1000000000000,2) as total_sec,
    round(avg_timer_wait/1000000000,2) as avg_ms,
    sum_rows_examined as rows_read,
    sum_rows_sent as rows_returned
from performance_schema.events_statements_summary_by_digest
where digest_text is not null
order by sum_timer_wait desc
limit 10;
```

### **Running queries longer than 5 seconds**

```sql
select id, user, host, db, command, time, state, left(info, 100) as query
from information_schema.processlist
where time > 5 and command != 'Sleep'
order by time desc;
```

---

## 📌 **5. Database Size & Index Analysis**

### **Database size & table count**

```sql
select table_schema as db,
       round(sum(data_length + index_length)/1024/1024, 2) as size_mb,
       count(*) as table_count
from information_schema.tables
group by table_schema
order by size_mb desc;
```

### **Unused and redundant indexes**

```sql
select * from sys.schema_unused_indexes;
select * from sys.schema_redundant_indexes;
```

---

## 📌 **6. Detailed Appointment & Patient Lookup**

### **Active recent patients with appointment data**

```sql
select p.id, p.email, p.first_name,
       a.id as appointment_id, a.total_price, a.status as appointment_status,
       s.start_time, sc.day_of_week
from patients p
inner join appointments a on p.id = a.patient_id and a.status in ('Pending','Scheduled','Completed')
inner join slots s on a.slot_id = s.id
inner join schedules sc on s.schedule_id = sc.id
where p.created_at > '2024-01-01'
  and p.is_active = true
  and p.id > 0
order by p.id asc
limit 50;
```

---

## 📌 **7. Global System Status Checks**

### **InnoDB & connection metrics**

```sql
show global status like 'Innodb_row_lock%';
show global status like 'Threads_connected';
show global status like 'Aborted_connects%';
```

---

## 📌 **8. Comprehensive Health Check Block**

### **Single query for system performance overview**

```sql
select
    (select count(*) from information_schema.processlist) as active_connections,
    (select variable_value from performance_schema.global_status where variable_name = 'Threads_connected') as current_connections,

    (select round(sum(data_length + index_length)/1024/1024, 2)
        from information_schema.tables where table_schema = database()) as db_size_mb,

    (select count(*) from mysql.slow_log where start_time > now() - interval 1 hour) as slow_query_last_hour,

    (select count(*) from performance_schema.data_locks) as current_locks,
    (select count(*) from performance_schema.data_lock_waits) as all_blocked_transaction,

    (select variable_value from performance_schema.global_status where variable_name = 'Innodb_buffer_pool_read_requests') as buffer_read_requests,
    (select variable_value from performance_schema.global_status where variable_name = 'Innodb_buffer_pool_reads') as buffer_disk_reads;
```

---

## 📌 **9. Deadlock Audit**

```sql
select 'Deadlock Information' as info,
       variable_value as deadlocks_since_start
from performance_schema.global_status
where variable_name = 'innodb_deadlocks';
```

---

## 📌 **10. Buffer Pool Efficiency Calculation**

```sql
select
    (select variable_value from performance_schema.global_status where variable_name = 'Innodb_buffer_pool_read_requests') as read_requests,
    (select variable_value from performance_schema.global_status where variable_name = 'Innodb_buffer_pool_reads') as disk_reads,
    round(
        (1 - (select variable_value from performance_schema.global_status where variable_name = 'Innodb_buffer_pool_reads') /
             (select variable_value from performance_schema.global_status where variable_name = 'Innodb_buffer_pool_read_requests')
        ) * 100, 2
    ) as hit_rate_percent;
```

---

## ✅ Summary

This README consolidates essential SQL scripts to:

* Diagnose performance bottlenecks
* Monitor MySQL engine status
* Detect data integrity issues
* Analyze index efficiency
* Audit slow queries & locking
* Evaluate buffer pool performance

You can directly copy/paste any block into your MySQL client or automation scripts.

If you'd like, I can also generate:

* A version with explanations for each query
* A shorter “quick reference” version
* A shell script that runs all diagnostics automatically
