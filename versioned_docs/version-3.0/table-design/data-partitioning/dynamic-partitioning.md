---
{
    "title": "Dynamic partitioning",
    "language": "en"
}
---

<!-- 
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->


Dynamic partitioning is designed to manage the lifecycle of partitions, reducing the burden on users.

Dynamic partitioning only supports RANGE partitioning based on `DATE` or `DATETIME` columns. It is applicable for cases where time data in partition columns grows synchronously with the real world. In such scenarios, data can be partitioned flexibly based on time data, and can be automatically stored with the cold-hot tiering strategy or recycled according to settings. 

For partitioning method that can be more widely applicable, see [Auto partitioning](https://doris.apache.org/docs/table-design/data-partitioning/auto-partitioning).

:::caution Warning

This feature will be disabled when synchronized by CCR. If this table is copied by CCR, that is, PROPERTIES contains `is_being_synced = true`, it will be displayed as enabled in show create table, but will not actually take effect. When `is_being_synced` is set to `false`, these features will resume working, but the `is_being_synced` property is for CCR peripheral modules only and should not be manually set during CCR synchronization.

:::

## How to use

The rules for dynamic partitioning can be specified when the table is created or modified at runtime.

Currently, dynamic partition rules can only be set for partition tables with single partition columns.    

- Specified when creating table

```sql
CREATE TABLE tbl1
(...)
PROPERTIES
(
    "dynamic_partition.prop1" = "value1",
    "dynamic_partition.prop2" = "value2",
    ...
)
```

- Modify at runtime

```sql
ALTER TABLE tbl1 SET
(
    "dynamic_partition.prop1" = "value1",
    "dynamic_partition.prop2" = "value2",
    ...
)
```

## Rule parameters

The rules of dynamic partition are prefixed with `dynamic_partition.`:

- `dynamic_partition.enable`

  Whether to enable the dynamic partition feature. Can be specified as `TRUE` or` FALSE`. If not filled, the default is `TRUE`. If it is `FALSE`, Doris will ignore the dynamic partitioning rules of the table.

- `dynamic_partition.time_unit`(required parameters)

  The unit for dynamic partition scheduling. Can be specified as `HOUR`,`DAY`,` WEEK`, `MONTH` and `YEAR`, means to create or delete partitions by hour, day, week, month and year, respectively.

  When specified as `HOUR`, the suffix format of the dynamically created partition name is `yyyyMMddHH`, for example, `2020032501`. *When the time unit is HOUR, the data type of partition column cannot be DATE.*

  When specified as `DAY`, the suffix format of the dynamically created partition name is `yyyyMMdd`, for example, `20200325`.

  When specified as `WEEK`, the suffix format of the dynamically created partition name is `yyyy_ww`. That is, the week of the year of current date. For example, the suffix of the partition created for `2020-03-25` is `2020_13`, indicating that it is currently the 13th week of 2020.

  When specified as `MONTH`, the suffix format of the dynamically created partition name is `yyyyMM`, for example, `202003`.

  When specified as `YEAR`, the suffix format of the dynamically created partition name is `yyyy`, for example, `2020`.

- `dynamic_partition.time_zone`

  The time zone of the dynamic partition, if not filled in, defaults to the time zone of the current machine's system, such as `Asia/Shanghai`, if you want to know the supported TimeZone, you can found in [Timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

- `dynamic_partition.start`

  The starting offset of the dynamic partition, usually a negative number. Depending on the `time_unit` attribute, based on the current day (week / month), the partitions with a partition range before this offset will be deleted. If not filled, the default is `-2147483648`, that is, the history partition will not be  deleted.

:::caution
Note that if the user sets history_partition_num (> 0), the starting partition for creating dynamic partitions will use max(start, -history_partition_num), and when deleting historical partitions, the range up to start will still be retained, where start < 0.
:::

- `dynamic_partition.end`(required parameters)

  The end offset of the dynamic partition, usually a positive number. According to the difference of the `time_unit` attribute, the partition of the corresponding range is created in advance based on the current day (week / month).


- `dynamic_partition.prefix`(required parameters)

  The dynamically created partition name prefix.


- `dynamic_partition.buckets`

    The number of buckets corresponding to the dynamically created partitions.

- `dynamic_partition.replication_num`

  The replication number of dynamic partition.If not filled in, defaults to the number of table's replication number.    

- `dynamic_partition.start_day_of_week`

  When `time_unit` is` WEEK`, this parameter is used to specify the starting point of the week. The value ranges from 1 to 7. Where 1 is Monday and 7 is Sunday. The default is 1, which means that every week starts on Monday.    


- `dynamic_partition.start_day_of_month`

  When `time_unit` is` MONTH`, this parameter is used to specify the start date of each month. The value ranges from 1 to 28. 1 means the 1st of every month, and 28 means the 28th of every month. The default is 1, which means that every month starts at 1st. The 29, 30 and 31 are not supported at the moment to avoid ambiguity caused by lunar years or months.

- Doris supports multi-level storage with SSD and HDD tiers. For more details, please refer to [tiered-storage](../tiered-storage/tiered-ssd-hdd.md)

- `dynamic_partition.create_history_partition`

  The default is false. When set to true, Doris will automatically create all partitions, as described in the creation rules below. At the same time, the parameter `max_dynamic_partition_num` of FE will limit the total number of partitions to avoid creating too many partitions at once. When the number of partitions expected to be created is greater than `max_dynamic_partition_num`, the operation will fail.

  When the `start` attribute is not specified, this parameter has no effect.

- `dynamic_partition.history_partition_num`

  When `create_history_partition` is `true`, this parameter is used to specify the number of history partitions. The default value is -1, which means it is not set.

- `dynamic_partition.reserved_history_periods`

  The range of reserved history periods. It should be in the form of `[yyyy-MM-dd,yyyy-MM-dd],[...,...]` while the `dynamic_partition.time_unit` is "DAY, WEEK, MONTH and YEAR". And it should be in the form of `[yyyy-MM-dd HH:mm:ss,yyyy-MM-dd HH:mm:ss],[...,...]` while the dynamic_partition.time_unit` is "HOUR". And no more spaces expected. The default value is `"NULL"`, which means it is not set.

  Let us give an example. Suppose today is 2021-09-06, partitioned by day, and the properties of dynamic partition are set to: 

  ```time_unit="DAY/WEEK/MONTH/YEAR", end=3, start=-3, reserved_history_periods="[2020-06-01,2020-06-20],[2020-10-31,2020-11-15]"```.

  The system will automatically reserve following partitions in following period :

  ```sql
  ["2020-06-01","2020-06-20"],
  ["2020-10-31","2020-11-15"]
  ```

  or

  ```time_unit="HOUR", end=3, start=-3, reserved_history_periods="[2020-06-01 00:00:00,2020-06-01 03:00:00]"```.

  The system will automatically reserve following partitions in following period :

  ```
  ["2020-06-01 00:00:00","2020-06-01 03:00:00"]
  ```

  Otherwise, every `[...,...]` in `reserved_history_periods` is a couple of properties, and they should be set at the same time. And the first date can't be larger than the second one.


## Create history partition rules

When `create_history_partition` is `true`, i.e. history partition creation is enabled, Doris determines the number of history partitions to be created based on `dynamic_partition.start` and `dynamic_partition.history_partition_num`. 

Assuming the number of history partitions to be created is `expect_create_partition_num`, the number is as follows according to different settings.

- `create_history_partition` = `true`  
  
  - `dynamic_partition.history_partition_num` is not set, i.e. -1.  
    `expect_create_partition_num` = `end` - `start`; 
  
  - `dynamic_partition.history_partition_num` is set   
    `expect_create_partition_num` = `end` - max(`start`, `-history_partition_num`);

- `create_history_partition` = `false`  

No history partition will be created, `expect_create_partition_num` = `end` - 0;

When `expect_create_partition_num` is greater than `max_dynamic_partition_num` (default 500), creating too many partitions is prohibited.

**Examples:** 

Suppose today is 2021-05-20, partition by day, and the attributes of dynamic partition are set to `create_history_partition=true, end=3, start=-3, history_partition_num=1`, then the system will automatically create the following partitions.

```
p20210519
p20210520
p20210521
p20210522
p20210523
```

`history_partition_num=5` and keep the rest attributes as in 1, then the system will automatically create the following partitions.

```
p20210517
p20210518
p20210519
p20210520
p20210521
p20210522
p20210523
```

`history_partition_num=-1` i.e., if you do not set the number of history partitions and keep the rest of the attributes as in 1, the system will automatically create the following partitions.

```
p20210517
p20210518
p20210519
p20210520
p20210521
p20210522
p20210523
```

## Example

1. Table `tbl1` partition column k1, type is DATE, create a dynamic partition rule. By day partition, only the partitions of the last 7 days are kept, and the partitions of the next 3 days are created in advance.

   ```sql
   CREATE TABLE tbl1
   (
       k1 DATE,
       ...
   )
   PARTITION BY RANGE(k1) ()
   DISTRIBUTED BY HASH(k1)
   PROPERTIES
   (
       "dynamic_partition.enable" = "true",
       "dynamic_partition.time_unit" = "DAY",
       "dynamic_partition.start" = "-7",
       "dynamic_partition.end" = "3",
       "dynamic_partition.prefix" = "p",
       "dynamic_partition.buckets" = "32"
   );
   ```

   Suppose the current date is 2020-05-29. According to the above rules, tbl1 will produce the following partitions:

   ```
   p20200529: ["2020-05-29", "2020-05-30")
   p20200530: ["2020-05-30", "2020-05-31")
   p20200531: ["2020-05-31", "2020-06-01")
   p20200601: ["2020-06-01", "2020-06-02")
   ```

   On the next day, 2020-05-30, a new partition will be created `p20200602: [" 2020-06-02 "," 2020-06-03 ")`

   On 2020-06-06, because `dynamic_partition.start` is set to 7, the partition 7 days ago will be deleted, that is, the partition `p20200529` will be deleted.

2. Table tbl1 partition column k1, type is DATETIME, create a dynamic partition rule. Partition by week, only keep the partition of the last 2 weeks, and create the partition of the next 2 weeks in advance.

   ```sql
   CREATE TABLE tbl1
   (
       k1 DATETIME,
       ...
   )
   PARTITION BY RANGE(k1) ()
   DISTRIBUTED BY HASH(k1)
   PROPERTIES
   (
       "dynamic_partition.enable" = "true",
       "dynamic_partition.time_unit" = "WEEK",
       "dynamic_partition.start" = "-2",
       "dynamic_partition.end" = "2",
       "dynamic_partition.prefix" = "p",
       "dynamic_partition.buckets" = "8"
   );
   ```

   Suppose the current date is 2020-05-29, which is the 22nd week of 2020. The default week starts on Monday. Based on the above rules, tbl1 will produce the following partitions:

   ```
   p2020_22: ["2020-05-25 00:00:00", "2020-06-01 00:00:00")
   p2020_23: ["2020-06-01 00:00:00", "2020-06-08 00:00:00")
   p2020_24: ["2020-06-08 00:00:00", "2020-06-15 00:00:00")
   ```

   The start date of each partition is Monday of the week. At the same time, because the type of the partition column k1 is DATETIME, the partition value will fill the hour, minute and second fields, and all are 0.

   On 2020-06-15, the 25th week, the partition 2 weeks ago will be deleted, ie `p2020_22` will be deleted.

   In the above example, suppose the user specified the start day of the week as `"dynamic_partition.start_day_of_week" = "3"`, that is, set Wednesday as the start of week. The partition is as follows:

   ```
   p2020_22: ["2020-05-27 00:00:00", "2020-06-03 00:00:00")
   p2020_23: ["2020-06-03 00:00:00", "2020-06-10 00:00:00")
   p2020_24: ["2020-06-10 00:00:00", "2020-06-17 00:00:00")
   ```

   That is, the partition ranges from Wednesday of the current week to Tuesday of the next week.

   :::tip

   2019-12-31 and 2020-01-01 are in same week, if the starting date of the partition is 2019-12-31, the partition name is `p2019_53`, if the starting date of the partition is 2020-01 -01, the partition name is `p2020_01`.

   :::

3. Table tbl1 partition column k1, type is DATE, create a dynamic partition rule. Partition by month without deleting historical partitions, and create partitions for the next 2 months in advance. At the same time, set the starting date on the 3rd of each month.

   ```sql
   CREATE TABLE tbl1
   (
       k1 DATE,
       ...
   )
   PARTITION BY RANGE(k1) ()
   DISTRIBUTED BY HASH(k1)
   PROPERTIES
   (
       "dynamic_partition.enable" = "true",
       "dynamic_partition.time_unit" = "MONTH",
       "dynamic_partition.end" = "2",
       "dynamic_partition.prefix" = "p",
       "dynamic_partition.buckets" = "8",
       "dynamic_partition.start_day_of_month" = "3"
   );
   ```

   Suppose the current date is 2020-05-29. Based on the above rules, tbl1 will produce the following partitions:

   ```
   p202005: ["2020-05-03", "2020-06-03")
   p202006: ["2020-06-03", "2020-07-03")
   p202007: ["2020-07-03", "2020-08-03")
   ```

   Because `dynamic_partition.start` is not set, the historical partition will not be deleted.

   Assuming that today is 2020-05-20, and set 28th as the start of each month, the partition range is:

   ```
   p202004: ["2020-04-28", "2020-05-28")
   p202005: ["2020-05-28", "2020-06-28")
   p202006: ["2020-06-28", "2020-07-28")
   ```

## Principle and control behavior

Apache Doris sets a fixed FE control thread that continuously checks the table based on dynamic partitioning at specific time intervals (specified by the `dynamic_partition_check_interval_seconds` field) to perform the necessary partition creation and deletion operations.

Specifically, when dynamic partitioning is executed, the following checks and operations are performed (refer to the start time of the partition as `START` and the end time as `END`, and omit `dynamic_partition.`):

- All partitions before `START` are deleted. 
- If `create_history_partition` is `false`, create all partitions between the current time and `END`; if `create_history_partition` is `true`, not only all partitions between the current time and `END` are created, but also all partitions between `START` and current time are created. If `history_partition_num` is specified, the number of created partitions before current time cannot exceed the value of `history_partition_num`. 

Note that:

- If the partition time range intersects with the `[START, END]` range, it is considered to belong to the current dynamic partition time range.
- If the newly created partition conflicts with an existing partition, the current partition is retained, and the new partition is not created. If the conflict occurs when the table is created, DDL will occur an error.

Therefore, after the automatic maintenance of the partition table, the state presented is as follows:

- **No partitions are included** before the `START` time except for those specified in `reserved_history_periods`.
- **All manually created partitions** after the `END` time are retained. 
- Apart from manually deleted or accidentally lost partitions, the table contains all partitions within a specific range:
  - If `create_history_partition` is `true`, 
    - if `history_partition_num` is specified, the specific range is `[max(START, current time) - history_partition_num * time_unit), END]`;
    - if `history_partition_num` is not specified, the specific range is `[START, END]`.
  - If `dynamic_partition.create_history_partition` is `false`, the specific range is `[current time, END]`, also including existing partitions in `[START, current time)`.
  The entire specific range is divided into multiple partition ranges based on `time_unit`. If a range intersects with an existing partition `X`, `X` is preserved; otherwise, the range will be completely covered by a partition created by dynamic partition. 
- Unless the number of partitions is about to exceed `max_dynamic_partition_num`, the creation will fail.

## Modify properties

You can modify the properties of the dynamic partitioning with the following command:

```sql
ALTER TABLE tbl1 SET
(
    "dynamic_partition.prop1" = "value1",
    ...
);
```

The modification of certain attributes may cause conflicts. Assume that the partition granularity was DAY and the following partitions have been created:

```
p20200519: ["2020-05-19", "2020-05-20")
p20200520: ["2020-05-20", "2020-05-21")
p20200521: ["2020-05-21", "2020-05-22")
```

If the partition granularity is changed to MONTH at this time, the system will try to create a partition with the range `["2020-05-01", "2020-06-01")`, and this range conflicts with the existing partition. So it cannot be created. And the partition with the range `["2020-06-01", "2020-07-01")` can be created normally. Therefore, the partition between 2020-05-22 and 2020-05-30 needs to be filled manually.

## Check table scheduling status

You can further view the scheduling of dynamic partitioned tables by using the following command:

```sql
mysql> SHOW DYNAMIC PARTITION TABLES;
+-----------+--------+----------+-------------+------+--------+---------+-----------+----------------+---------------------+--------+------------------------+----------------------+-------------------------+
| TableName | Enable | TimeUnit | Start       | End  | Prefix | Buckets | StartOf   | LastUpdateTime | LastSchedulerTime   | State  | LastCreatePartitionMsg | LastDropPartitionMsg | ReservedHistoryPeriods  |
+-----------+--------+----------+-------------+------+--------+---------+-----------+----------------+---------------------+--------+------------------------+----------------------+-------------------------+
| d3        | true   | WEEK     | -3          | 3    | p      | 1       | MONDAY    | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | [2021-12-01,2021-12-31] |
| d5        | true   | DAY      | -7          | 3    | p      | 32      | N/A       | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    |
| d4        | true   | WEEK     | -3          | 3    | p      | 1       | WEDNESDAY | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    |
| d6        | true   | MONTH    | -2147483648 | 2    | p      | 8       | 3rd       | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    |
| d2        | true   | DAY      | -3          | 3    | p      | 32      | N/A       | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    |
| d7        | true   | MONTH    | -2147483648 | 5    | p      | 8       | 24th      | N/A            | 2020-05-25 14:29:24 | NORMAL | N/A                    | N/A                  | NULL                    |
+-----------+--------+----------+-------------+------+--------+---------+-----------+----------------+---------------------+--------+------------------------+----------------------+-------------------------+
7 rows in set (0.02 sec)
```

- `LastUpdateTime`: The last time of modifying dynamic partition properties 
- `LastSchedulerTime`: The last time of performing dynamic partition scheduling
- `State`: The state of the last execution of dynamic partition scheduling
- `LastCreatePartitionMsg`: Error message of the last time to dynamically add partition scheduling
- `LastDropPartitionMsg`: Error message of the last execution of dynamic deletion partition scheduling

## Advanced operations

**Modify FE configuration items**

- `dynamic\_partition\_enable`

  Whether to enable Doris's dynamic partition feature. The default value is `false`, which is off. This parameter only affects the partitioning operation of dynamic partition tables, not normal tables. You can modify the parameters in `fe.conf` and restart FE to take effect. You can also execute the following commands at runtime to take effect:

  ```sql
  MySQL protocol:

  `ADMIN SET FRONTEND CONFIG ("dynamic_partition_enable" = "true")`

  HTTP protocol:

  `curl --location-trusted -u username:password -XGET http://fe_host:fe_http_port/api/_set_config?dynamic_partition_enable=true`
  ```

  To turn off dynamic partitioning globally, set this parameter to `false`.

- `dynamic\_partition\_check\_interval\_seconds`

  The execution frequency of dynamic partition threads defaults to 600 (10 minutes), that is, scheduling is performed every 10 minutes. You can modify the parameters in `fe.conf` and restart FE to take effect. You can also modify the following commands at runtime:

  ```sql
  MySQL protocol:
  
  `ADMIN SET FRONTEND CONFIG ("dynamic_partition_check_interval_seconds" = "7200")`
  
  HTTP protocol:
  
  `curl --location-trusted -u username:password -XGET http://fe_host:fe_http_port/api/_set_config?dynamic_partition_check_interval_seconds=432000`
  ```

**Switching between dynamic partitioning and manual partitioning**

You can switch a table between dynamic and manual partitioning, but a table cannot be partitioned simultaneously by dynamic and manual partitioning.

By running the `ALTER TABLE tbl_name SET ("dynamic_partition.enable" = "<true/false>")` command, you can turn on and off dynamic partitioning.

When dynamic partitioning is turned off, Apache Doris will no longer manage partitions automatically, and users need to create or delete partitions manually by using `ALTER TABLE`; when dynamic partitioning is turned on, redundant partitions will be deleted according to the rules of dynamic partitioning.
