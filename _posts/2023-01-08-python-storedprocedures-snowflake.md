---
title: "Creating Python stored procedures in Snowflake"
date: 2023-01-08T20:08:30-04:00
categories:
  - blog
tags:
  - Snowflake
  - Python
---

A really great feature of Snowflake is the ability to write stored procedures and functions in Python!
You can then use these stored procedures or functions to operate on your Snowflake objects.

One thing I frequently do is compare one field to another, to determine if something exists in one dataset but not another. Does one table contain sales orders, pallet numbers, or report ID's that the other table 
does not? This question comes up frequently for projects, troubleshooting etc. and it dawned on me that it
would be great to have the ability to streamline this entirely within Snowflake.
 
I'll be using the Snowflake UI, as well as Snowpark for Python in this example. To set up your Snowpark environment, please refer to the [official Snowflake documentation](https://docs.snowflake.com/en/developer-guide/snowpark/python/setup.html) on the topic.

Below is a minimum reproducible example that will run in Snowflake to proof out this functionality. Although this is a rudimentary example, you can see how this quickly becomes a powerful tool. In a production environment, perhaps a sales order or item is missing from one report but present in another dataset.

In this example, we create two tables containing fruits and the amount of that fruit. We then create a stored procedure that we can call to determine which fruits exist in one table, but not the other:

```sql
set ro = 'sysadmin';
set wh = 'my_wh';
set db = 'dev';
set sch = 'my_schema';

use role identifier($ro);
use warehouse identifier($wh);
use database identifier($db);
use schema identifier($sch);

create or replace table mytable(amount number comment 'fake amounts for testing', fruits string comment 'fake types of fruit for testing');
create or replace table mytable2 like mytable;
    
insert into mytable values (1, 'apple'),(2, 'orange'),(5, 'grape'),(7, 'cantelope'),(9, 'pineapple'),(17, 'banana'),(21, 'tangerine');
insert into mytable2 values (1, 'apple'),(3, 'orange'),(5, 'grape'),(7, 'strawberry'),(10, 'pineapple'),(17, 'banana'),(22, 'raspberry');

/* testing */
-- select * from mytable;
-- select * from mytable2;

create or replace procedure print_differences(TABLE1 string, TABLE2 string, FIELD1 string, FIELD2 string)
returns array
language python 
runtime_version = '3.8'
packages = ('snowflake-snowpark-python', 'pandas')
handler = 'print_differences'
as 
$$
import pandas as pd

def print_differences(session, table1: str,table2: str,field1: str,field2: str):

    #read the tables into a snowpark dataframe
    table1 = session.table(table1)
    table2 = session.table(table2)

    #convert to pandas
    df1 = table1.to_pandas()
    df2 = table2.to_pandas()

    # convert the the fields of interest from each table to a list
    list1 = df1[field1].to_list()
    list2 = df2[field2].to_list()

    return [item for item in list1 if item not in list2]
$$
;

call print_differences('MYTABLE2', 'MYTABLE', 'FRUITS', 'FRUITS');

-- output:
-- ["cantelope","tangerine"]
```

You can also do this entirely in Python if you prefer, and the stored procedure will be saved in Snowflake. First, lets import our modules and connect to Snowpark:

```py
import os, snowflake
import pandas as pd

from snowflake.snowpark import Session
from snowflake.snowpark.types import StringType

from dotenv import load_dotenv
load_dotenv()

# establish Snowflake Connection
account = os.getenv('SNOWFLAKE_ACCT')
user = os.getenv('SNOWFLAKE_USER')
password = os.getenv('SNOWFLAKE_PASSWORD')
role = os.getenv('SNOWFLAKE_ROLE')
warehouse = 'my_wh'
database = 'dev'
schema = 'my_schema'

def snowpark_cnxn(account, user, password, role, warehouse, database, schema):
    connection_parameters = {
        "account": account,
        "user": user,
        "password": password,
        "role": role,
        "warehouse": warehouse,
        "database": database,
        "schema": schema
    }
    session = Session.builder.configs(connection_parameters).create()
    return session

session = snowpark_cnxn(account, user, password, role, warehouse, database, schema)

print(session.sql('select current_warehouse(), current_database(), current_schema()').collect())
```

Next, we'll use the `sql` method to execute our `CREATE` and `INSERT` DDL:

```py
session.sql("""create or replace table mytable(amount number comment 'fake amounts for testing', fruits string comment 'fake types of fruit for testing')""").show()
session.sql("""create or replace table mytable2 like mytable""").show()
session.sql("""insert into mytable values (1, 'apple'),(2, 'orange'),(5, 'grape'),(7, 'cantelope'),(9, 'pineapple'),(17, 'banana'),(21, 'tangerine')""").show()
session.sql("""insert into mytable2 values (1, 'apple'),(3, 'orange'),(5, 'grape'),(7, 'strawberry'),(10, 'pineapple'),(17, 'banana'),(22, 'raspberry')""").show()
```

Then, we simply define a function in Python just like we would any other function:

```py
def print_differences(session: snowflake.snowpark.Session, table1: str,table2: str,field1: str,field2: str):
    # read the tables into a snowpark dataframe
    table1 = session.table(table1)
    table2 = session.table(table2)

    # convert to pandas
    df1 = table1.to_pandas()
    df2 = table2.to_pandas()

    # convert the the fields of interest from each table to a list
    list1 = df1[field1].to_list()
    list2 = df2[field2].to_list()

    return ', '.join(item for item in list1 if item not in list2)

```

then, we execute the `sproc.register` method to register the stored procedure in Snowflake:

```py
session.add_packages('snowflake-snowpark-python')

session.sproc.register(
    func = print_differences
  , return_type = StringType()
  , input_types = [StringType(), StringType(), StringType(), StringType()]
  , is_permanent = True
  , name = 'PRINT_DIFFERENCES'
  , replace = True
  , stage_location = '@UDF_STAGE'
)
```

You can return the data in a few ways, I prefer a tabular output so I like the second option to return the data as a Pandas dataframe:

```py
# option 1: You can return the results on one line using the sql() method:
session.sql('''call print_differences('MYTABLE', 'MYTABLE2', 'FRUITS', 'FRUITS')''').show()

# option 2: Call stored procedure, print results as dataframe
x = session.call('print_differences', 'MYTABLE', 'MYTABLE2', 'FRUITS', 'FRUITS')

df = pd.DataFrame({'Differences': x.split(',')})
print(df)
```

Output of option 2:

| Differences |
| ----------- |
| cantelope   |
| tangerine   |

Here is the entire Python script:

```py
#import modules
import os, snowflake
import pandas as pd

from snowflake.snowpark import Session
from snowflake.snowpark.types import StringType

from dotenv import load_dotenv
load_dotenv()

# establish Snowflake Connection
account = os.getenv('SNOWFLAKE_ACCT')
user = os.getenv('SNOWFLAKE_USER')
password = os.getenv('SNOWFLAKE_PASSWORD')
role = os.getenv('SNOWFLAKE_ROLE')
warehouse = 'my_wh'
database = 'dev'
schema = 'my_schema'

def snowpark_cnxn(account, user, password, role, warehouse, database, schema):
    connection_parameters = {
        "account": account,
        "user": user,
        "password": password,
        "role": role,
        "warehouse": warehouse,
        "database": database,
        "schema": schema
    }
    session = Session.builder.configs(connection_parameters).create()
    return session

print('Connecting to Snowpark...\n')
session = snowpark_cnxn(account, user, password, role, warehouse, database, schema)

print(session.sql('select current_warehouse(), current_database(), current_schema()').collect(), '\n')
print('Connected!\n')

session.sql("""create or replace table mytable(amount number comment 'fake amounts for testing', fruits string comment 'fake types of fruit for testing')""").show()
session.sql("""create or replace table mytable2 like mytable""").show()
session.sql("""insert into mytable values (1, 'apple'),(2, 'orange'),(5, 'grape'),(7, 'cantelope'),(9, 'pineapple'),(17, 'banana'),(21, 'tangerine')""").show()
session.sql("""insert into mytable2 values (1, 'apple'),(3, 'orange'),(5, 'grape'),(7, 'strawberry'),(10, 'pineapple'),(17, 'banana'),(22, 'raspberry')""").show()

def print_differences(session: snowflake.snowpark.Session, table1: str,table2: str,field1: str,field2: str):
    # read the tables into a snowpark dataframe
    table1 = session.table(table1)
    table2 = session.table(table2)

    # convert to pandas
    df1 = table1.to_pandas()
    df2 = table2.to_pandas()

    # convert the the fields of interest from each table to a list
    list1 = df1[field1].to_list()
    list2 = df2[field2].to_list()

    return ', '.join(item for item in list1 if item not in list2)

session.add_packages('snowflake-snowpark-python')

print('Registering Stored Procedure with Snowflake...\n')

session.sproc.register(
    func = print_differences
  , return_type = StringType()
  , input_types = [StringType(), StringType(), StringType(), StringType()]
  , is_permanent = True
  , name = 'PRINT_DIFFERENCES'
  , replace = True
  , stage_location = '@UDF_STAGE'
)

print('Stored Procedure registered with Snowflake!\n')

# You can return the results on one line using the sql() method:
# session.sql('''call print_differences('MYTABLE', 'MYTABLE2', 'FRUITS', 'FRUITS')''').show()

# call stored procedure, print results as dataframe
x = session.call('print_differences', 'MYTABLE', 'MYTABLE2', 'FRUITS', 'FRUITS')
print(x, '\n')

df = pd.DataFrame({'Differences': x.split(',')})
print(df)
```