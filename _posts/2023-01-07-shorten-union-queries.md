---
title: "How to shorten large union queries in Snowflake"
date: 2023-01-07T13:18:30-04:00
categories:
  - blog
tags:
  - Jekyll
  - update
---

Recently I was trying to reduce the size of a large union query in Snowflake where the only thing that changed between the queries was the schema name. If you work with SQL professionally, you've probably encountered something like this:

```sql
select 
    'COMPANY_NAME' as Company
  , GL.ACTNUM as Account_Number
  , ACT.DESCRIPTION as Account_Name
  from COMPANY_NAME1.General_Ledger_Table GL 

  inner join COMPANY_NAME1.Account_Name_Table ACT
      on ACT.ID = GL.ID

union all

...

select 
    'COMPANY_NAME' as Company
  , GL.ACTNUM as Account_Number
  , ACT.DESCRIPTION as Account_Name
  from COMPANY_NAME17.General_Ledger_Table GL 

  inner join COMPANY_NAME17.Account_Name_Table ACT
      on ACT.ID = GL.ID
```
The query that inspired this post was originally a SQL Server query, but with Fivetran, we've now pushed all of our data up to Snowflake. This felt like the perfect time for a redesign - turns out there is a much better way to do this in Snowflake! 

Here is a basic example that accomplishes the same as above. The code loops over a list of company names (the results of the 'organization' cursor in the below example), unions the results together, and then returns the final results using a resultset:

```sql
set ro = 'sysadmin';
set wh = 'my_wh';
set db = 'dev';
set sch = 'my_schema';

use role identifier($ro);
use warehouse identifier($wh);
use database identifier($db);
use schema identifier($sch);

declare
    sql varchar;
    final_sql varchar;
    organization cursor for (select COMPANY_NAME from MY_TABLE);
    my_results resultset;
begin
    final_sql := '';
    
    for company in organization do
        sql := $$select 'COMPANY_NAME' as Company
            , GL.ACTNUM as Account_Number
            , ACT.DESCRIPTION as Account_Name
            from COMPANY_NAME.General_Ledger_Table GL 

            inner join COMPANY_NAME.Account_Name_Table ACT
                on ACT.ID = GL.ID
            $$;
        sql := replace(sql, 'COMPANY_NAME', company.COMPANY_NAME);

        if(final_sql != '')then 
          final_sql := final_sql || ' union all ';
        end if;

        final_sql := final_sql || sql;
    end for;
    
    my_results := (execute immediate :final_sql);
    return table(my_results);
end;
```
Note that this is a simplified example for the purposes of this blog. Using similar logic I was able to reduce a 300 line production query to ~40 lines of code! 