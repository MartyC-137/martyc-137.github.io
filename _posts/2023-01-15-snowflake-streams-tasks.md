---
title: "Streams and Tasks workflow in Snowflake"
date: 2023-01-15T13:18:30-04:00
categories:
  - blog
tags:
  - Snowflake
  - SQL
---

Snowflake [streams](https://docs.snowflake.com/en/user-guide/streams-intro.html) and [tasks](https://docs.snowflake.com/en/user-guide/tasks-intro.html) provide a mechanism for automatically updating one table, as soon as data is loaded into another table.A Snowflake stream tracks all data manipulation language (DML) changes to a table, and by wrapping a stored procedure inside of a Snowflake task, we can update another table based on the DML changes of another table.

Something I commonly see in a production environment is views created from another view, simply to move the dataset to a different database or schema:

```sql
create or replace my_2nd_view
as
select
...
from my_first_view
```
[Nested views negatively affect performance](https://bornsql.ca/blog/nested-views-bad/), as views are computed at runtime. If you simply need to move a dataset from one location to another within Snowflake (for example, from staging to prod, or the databse you expose to end users), streams and tasks is a much better alternative.

To start, lets set a few session variables and initialize our session: 
```sql
/* Set session variables */
set role_name = 'sysadmin';
set wh = 'my_wh';
set db = 'my_db';
set sch = 'my_schema';
set dest_table = 'my_table';
set stream_name = 'my_stream';
set source_table = 'source_db.source_schema.source_table';
set proc_name = 'my_procedure';
set task_name = 'push_my_table';

/* Initialize Environment */
use role identifier($role_name);
use warehouse identifier($wh)
```
Next, lets create our database, table, and stream objects:
```sql
/* Create database, table and stream objects */
create database if not exists identifier($db);
create schema if not exists identifier($sch);

use database identifier($db);
use schema identifier($sch);

create table if not exists identifier($dest_table)
comment='JSON data from API, streaming from the source database'
clone identifier($source_table);

create stream if not exists identifier($stream_name) on table identifier($source_table)
comment = 'CDC stream from source table to prod table';

/* quick diagnostic check */
show streams;
select * from identifier($stream_name);
```
A Snowflake stream is a schema level object that contains the same fields as your source table, but also includes the following fields:

* `METADATA$ACTION`: Specifies whether the action is an `INSERT` or a `DELETE`
* `METADATA$ISUPDATE`: Specifies whether the `INSERT` or `DELETE` is part of an `UPDATE`
* `METADATA$ROW_ID`: A unique row ID. This ID can be used to track changes to specific rows over time

So, in our basic example above, here is how the results of `select * from identifier($stream_name);` would look:

| NAME | AMOUNT | METADATA\$ACTION | METADATA\$ISUPDATE | METADATA\$ROW_ID |
| ---- | ------ | ---------------- | ------------------ | ---------------- |
| John | 5 | INSERT | FALSE | 3d5ttsht47wssy8bv |

CODE THREE
```sql
/* Create stored procedure */
create or replace procedure identifier($proc_name)()
returns varchar
language sql
execute as owner
as
$$
begin
merge into my_table DEST using (
    select * from my_stream
      qualify row_number() over (
      partition by json_data:ID order by insert_date) = 1
      ) SOURCE
        on DEST.json_data:ID = SOURCE.json_data:ID
when matched and metadata$action = 'INSERT' then 
update set DEST.json_data = SOURCE.json_data,
            DEST.insert_date = current_timestamp()
when not matched and metadata$action = 'INSERT' then 
insert (DEST.json_data, DEST.insert_date)
        values(SOURCE.json_data, current_timestamp());
return 'CDC records successfully inserted';
end;
$$;

/* Create task */
create or replace task identifier($task_name)
warehouse = LOAD_WH
schedule = '1 minute'
comment = 'Change data capture task that pulls over new data once a minute'
when system$stream_has_data('my_stream')
as
call my_procedure();

/* grant execute task priveleges to role sysadmin */
use role accountadmin;
grant execute task on account to role identifier($role_name);

/* tasks are created in a suspended state by default, you must 'resume' them to schedule them */
use role identifier($role_name);
alter task identifier($task_name) resume;

select * from identifier($my_table);

show tasks;
select * from table(information_schema.task_history()) order by scheduled_time;
```