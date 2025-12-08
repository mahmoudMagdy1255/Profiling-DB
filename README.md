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

---
| query_preview | runs | total_sec | avg_ms | rows_read | rows_returned | 
| --- | ---: | ---: | ---: | ---: | ---: | 
| COMMIT | 89900 | 54.37 | 0.60 | 0 | 0 | 
| COMMIT | 1580 | 16.61 | 10.51 | 0 | 0 | 
| COMMIT | 875 | 9.86 | 11.27 | 0 | 0 | 
| START TRANSACTION | 90052 | 8.86 | 0.10 | 0 | 0 | 
| SHOW TABLE STATUS FROM `clinic` | 35 | 3.39 | 96.98 | 19042 | 4745 | 
| COMMIT | 288 | 2.81 | 9.74 | 0 | 0 | 
| USE `eatizaz-ai` | 12435 | 2.37 | 0.19 | 0 | 0 | 
| SET NAMES ? COLLATE ? , SESSION `sql_mode` = ? | 12443 | 1.97 | 0.16 | 0 | 0 | 
| RENAME TABLE `swcc-aware` . `cache` TO `swcc-aware2` . `cache` , `swcc-aware` . `cache_locks` TO `sw | 1 | 1.53 | 1528.67 | 0 | 0 | 
| USE `clinic` | 6003 | 1.17 | 0.20 | 0 | 0 | 


### **Running queries longer than 5 seconds**

```sql
select id, user, host, db, command, time, state, left(info, 100) as query
from information_schema.processlist
where time > 5 and command != 'Sleep'
order by time desc;
```
---
| id | user | host | db | command | time | state | query | 
| ---: | --- | --- | --- | --- | ---: | --- | --- | 
| 5 | event_scheduler | localhost | \N | Daemon | 2229897 | Waiting on empty queue | \N | 

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

---
| db | size_mb | table_count | 
| --- | ---: | ---: | 
| adahi_dashboard_1 | 481.55 | 42 | 
| adahi_dashboard_2 | 135.97 | 42 | 
| last-eatizaz | 71.05 | 124 | 
| magicai | 23.09 | 127 | 
| clinic | 22.06 | 139 | 
| gadd-ai | 21.64 | 35 | 
| beta-wakeb | 20.69 | 33 | 
| jeddah-lpr | 10.89 | 52 | 
| sarie-crm | 9.16 | 97 | 
| kafd | 8.13 | 125 | 
| chamber | 7.95 | 120 | 
| sharqiah | 7.67 | 114 | 
| mujib-plus | 7.45 | 118 | 
| mysql | 7.25 | 38 | 
| vms-wakeb | 5.14 | 92 | 
| new-aware | 4.59 | 37 | 
| g-tech | 4.28 | 118 | 
| aramco-drone | 3.56 | 41 | 
| aramco-ai | 3.31 | 35 | 
| swcc-aware2 | 3.08 | 33 | 
| wakeb-website | 3.00 | 42 | 
| swcc-aware | 2.86 | 33 | 
| gadd-ai-old | 2.06 | 35 | 
| gaddplatform | 1.75 | 38 | 
| kanban_gadd | 1.59 | 43 | 
| spack | 1.47 | 43 | 
| videopf | 1.28 | 42 | 
| nwc-chatbot | 0.63 | 22 | 
| eatizaz-ai | 0.59 | 19 | 
| moc-ai | 0.55 | 19 | 
| rpa_logic_apps | 0.47 | 18 | 
| rpa_sso | 0.31 | 16 | 
| rpa_auth | 0.09 | 6 | 
| sys | 0.02 | 101 | 
| performance_schema | 0.00 | 111 | 
| information_schema | 0.00 | 79 | 


### **Unused and redundant indexes**

```sql
select * from sys.schema_unused_indexes;
select * from sys.schema_redundant_indexes;
```

schema_unused_indexes
---
| object_schema | object_name | index_name | 
| --- | --- | --- | 
| aramco-drone | roles | roles_created_by_foreign | 
| aramco-drone | steps | steps_created_by_foreign | 
| aramco-drone | users | users_created_by_foreign | 
| aramco-drone | users | users_phone_code_id_foreign | 
| beta-wakeb | wakeb_analytics_pages | wakeb_analytics_pages_date_time_index | 
| beta-wakeb | wakeb_topic_fields | topic_fields_field_id | 
| beta-wakeb | wakeb_webmaster_section_fields | webmaster_section_fields_webmaster_id | 
| clinic | activity_log | subject | 
| clinic | activity_log | causer | 
| clinic | activity_log | activity_log_log_name_index | 
| clinic | admin_complaint_logs | admin_complaint_logs_actorable_type_actorable_id_index | 
| clinic | admin_complaint_logs | admin_complaint_logs_complaint_id_foreign | 
| clinic | admins | admins_created_by_foreign | 
| clinic | advertisements | advertisements_created_by_foreign | 
| clinic | agora_session_participants | agora_session_participants_joined_at_index | 
| clinic | allergies | allergies_allergy_type_id_foreign | 
| clinic | allergies | allergies_allergy_substance_id_foreign | 


schema_redundant_indexes
---
| table_schema | table_name | redundant_index_name | redundant_index_columns | redundant_index_non_unique | dominant_index_name | dominant_index_columns | dominant_index_non_unique | subpart_exists | sql_drop_index | 
| --- | --- | --- | --- | ---: | --- | --- | ---: | ---: | --- | 
| aramco-ai | pulse_entries | pulse_entries_timestamp_index | timestamp | 1 | pulse_entries_timestamp_type_key_hash_value_index | timestamp,type,key_hash,value | 1 | 0 | ALTER TABLE `aramco-ai`.`pulse_entries` DROP INDEX `pulse_entries_timestamp_index` | 
| aramco-ai | pulse_values | pulse_values_type_index | type | 1 | pulse_values_type_key_hash_unique | type,key_hash | 0 | 0 | ALTER TABLE `aramco-ai`.`pulse_values` DROP INDEX `pulse_values_type_index` | 
| aramco-ai | user_locations | user_locations_user_id_index | user_id | 1 | PRIMARY | user_id,location_id | 0 | 0 | ALTER TABLE `aramco-ai`.`user_locations` DROP INDEX `user_locations_user_id_index` | 
| aramco-drone | pulse_entries | pulse_entries_timestamp_index | timestamp | 1 | pulse_entries_timestamp_type_key_hash_value_index | timestamp,type,key_hash,value | 1 | 0 | ALTER TABLE `aramco-drone`.`pulse_entries` DROP INDEX `pulse_entries_timestamp_index` | 
| aramco-drone | pulse_values | pulse_values_type_index | type | 1 | pulse_values_type_key_hash_unique | type,key_hash | 0 | 0 | ALTER TABLE `aramco-drone`.`pulse_values` DROP INDEX `pulse_values_type_index` | 
| aramco-drone | user_locations | user_locations_user_id_index | user_id | 1 | PRIMARY | user_id,location_id | 0 | 0 | ALTER TABLE `aramco-drone`.`user_locations` DROP INDEX `user_locations_user_id_index` | 
| chamber | attribute_categories | attribute_categories_attribute_id_index | attribute_id | 1 | PRIMARY | attribute_id,category_id | 0 | 0 | ALTER TABLE `chamber`.`attribute_categories` DROP INDEX `attribute_categories_attribute_id_index` | 
| chamber | product_attribute_values | product_attribute_values_product_attribute_id_index | product_attribute_id | 1 | PRIMARY | product_attribute_id,attribute_value_id | 0 | 0 | ALTER TABLE `chamber`.`product_attribute_values` DROP INDEX `product_attribute_values_product_attribute_id_index` | 
| chamber | product_options | product_options_product_id_index | product_id | 1 | PRIMARY | product_id,option_id | 0 | 0 | ALTER TABLE `chamber`.`product_options` DROP INDEX `product_options_product_id_index` | 
| clinic | appointment_statuses | appointment_statuses_status_name_index | status_name | 1 | appointment_statuses_status_name_unique | status_name | 0 | 0 | ALTER TABLE `clinic`.`appointment_statuses` DROP INDEX `appointment_statuses_status_name_index` | 
| clinic | appointments | appointments_appointment_date_index | appointment_date | 1 | appointment_doctor_index | appointment_date,doctor_id | 1 | 0 | ALTER TABLE `clinic`.`appointments` DROP INDEX `appointments_appointment_date_index` | 
| clinic | appointments | appointments_appointment_date_index | appointment_date | 1 | appointment_patient_index | appointment_date,patient_id | 1 | 0 | ALTER TABLE `clinic`.`appointments` DROP INDEX `appointments_appointment_date_index` | 
| clinic | patient_blogs | patient_blogs_patient_id_index | patient_id | 1 | PRIMARY | patient_id,blog_id | 0 | 0 | ALTER TABLE `clinic`.`patient_blogs` DROP INDEX `patient_blogs_patient_id_index` | 
| clinic | patient_doctors | patient_doctors_patient_id_index | patient_id | 1 | PRIMARY | patient_id,doctor_id | 0 | 0 | ALTER TABLE `clinic`.`patient_doctors` DROP INDEX `patient_doctors_patient_id_index` | 


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
| id | email | first_name | appointment_id | total_price | appointment_status | start_time | day_of_week | 
| ---: | --- | --- | ---: | ---: | --- | --- | --- | 
| 1 | \N | Mariem | 3 | 150.00 | Completed | 10:15:00 | Monday | 
| 1 | \N | Mariem | 6 | 0.00 | Scheduled | 10:15:00 | Monday | 
| 1 | \N | Mariem | 7 | 0.00 | Scheduled | 12:52:00 | Monday | 
| 1 | \N | Mariem | 15 | 25.00 | Scheduled | 12:55:00 | Sunday | 
| 1 | \N | Mariem | 17 | 0.00 | Scheduled | 17:00:00 | Tuesday | 
| 1 | \N | Mariem | 22 | 2.00 | Completed | 13:00:00 | Monday | 
| 1 | \N | Mariem | 28 | 44.00 | Scheduled | 09:30:00 | Tuesday | 
| 1 | \N | Mariem | 29 | 2.00 | Scheduled | 17:00:00 | Tuesday | 
| 1 | \N | Mariem | 30 | 2.00 | Scheduled | 10:19:00 | Tuesday | 
| 1 | \N | Mariem | 184 | 0.00 | Scheduled | 11:35:00 | Wednesday | 
| 1 | \N | Mariem | 254 | 22.00 | Scheduled | 10:15:00 | Monday | 
| 3 | \N | mohamed | 19 | 25.00 | Scheduled | 13:05:00 | Sunday | 
| 8 | \N | Yousef | 73 | 0.00 | Scheduled | 12:45:00 | Sunday | 
| 8 | \N | Yousef | 75 | 0.00 | Scheduled | 14:35:00 | Sunday | 
| 8 | \N | Yousef | 140 | 300.00 | Scheduled | 16:00:00 | Tuesday | 


---

## 📌 **7. Global System Status Checks**

### **InnoDB & connection metrics**

```sql
show global status like 'Innodb_row_lock%';
show global status like 'Threads_connected';
show global status like 'Aborted_connects%';
```

Innodb_row_lock
---
| Variable_name | Value | 
| --- | --- | 
| Innodb_row_lock_current_waits | 0 | 
| Innodb_row_lock_time | 17818 | 
| Innodb_row_lock_time_avg | 12 | 
| Innodb_row_lock_time_max | 277 | 
| Innodb_row_lock_waits | 1448 | 

Threads_connected
---
| Variable_name | Value | 
| --- | --- | 
| Threads_connected | 1 | 


Aborted_connects
---
| Variable_name | Value | 
| --- | --- | 
| Aborted_connects | 0 | 


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
| active_connections | current_connections | db_size_mb | slow_query_last_hour | current_locks | all_blocked_transaction | buffer_read_requests | buffer_disk_reads | 
| ---: | --- | ---: | ---: | ---: | ---: | --- | --- | 
| 2 | 1 | 22.06 | 0 | 0 | 0 | 39043876 | 5113 | 


---

## 📌 **9. Deadlock Audit**

```sql
select 'Deadlock Information' as info,
       variable_value as deadlocks_since_start
from performance_schema.global_status
where variable_name = 'innodb_deadlocks';
```
+-----------------------+------------------------+
| info                  | deadlocks_since_start |
+-----------------------+------------------------+
| Deadlock Information  | 42                     |
+-----------------------+------------------------+

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
| read_requests | disk_reads | hit_rate_percent | 
| --- | --- | ---: | 
| 39044624 | 5113 | 99.99 | 


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
