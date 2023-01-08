---
title: "How to backload JSON data to Snowflake using Snowpark for Python"
date: 2023-01-07T20:13:30-04:00
categories:
  - blog
tags:
  - Snowflake
  - Python
---

Recently I needed to backload a few years of json data from an API to Snowflake. This felt like a great opportunity to test out [Snowpark Python](https://docs.snowflake.com/en/developer-guide/snowpark/python/index.html)!

In the code below, my Snowflake credentials are hidden in a `.env` file, and I load them into the python script using the `python-dotenv` library. 

Since the `.env` file contains your Snowflake secrets, you don't want to add it to your cloud repo! Make sure you add `.env` to your `gitignore` by running the following command in a terminal:

```bash
cd <your_repo>
echo '.env' >> .gitignore
```

The script below loops over a specified date range (in this case, 4 years, incremented by day), calls an API, and inserts a record into a Snowflake `VARIANT` table:

```py

# Import modules
import os, json, requests

from datetime import date, timedelta
from snowflake.snowpark import Session

from dotenv import load_dotenv
load_dotenv()

# Establish Snowflake Connection using Snowpark
account = os.getenv('SNOWFLAKE_ACCT')
user = os.getenv('SNOWFLAKE_USER')
password = os.getenv('SNOWFLAKE_PASSWORD')
role = os.getenv('SNOWFLAKE_ROLE')
role = 'SYSADMIN'
warehouse = 'MY_WH'
database = 'DEV'
schema = 'MY_SCHEMA'
target_table = 'MY_TABLE'

api_key = os.getenv('MY_API_KEY')

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

session = snowpark_cnxn(account,
                        user,
                        password,
                        role,
                        warehouse,
                        database,
                        schema)

print(session.sql('SELECT CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA()').collect())

# API variables
headers = {
  'APIKey': f'{api_key}'
}

# Define a function so we can loop over a date range
def daterange(start_date, end_date):
    for n in range(int((end_date - start_date).days)):
        yield start_date + timedelta(n)
        
start_date = date(2019, 1, 1)
end_date = date(2022, 12, 31)

# Loop through 4 years worth of API data, insert into Snowflake VARIANT table
for date in daterange(start_date, end_date):
    url = f'https://api.mywebsite.com/api/data?&startDate={date}&endDate={date}'
    response = requests.request("GET", url, headers=headers)
    
    formatted_json = json.loads(response.text)
    formatted_json = json.dumps(formatted_json, indent = 4)
    
    # insert to Snowflake
    session.sql(f'''INSERT INTO {target_table} (JSON_DATA, INSERT_DATE)
                    SELECT PARSE_JSON('{formatted_json}'),
                    CURRENT_TIMESTAMP();''').collect()
```

Note that this did take ~15 minutes to run. For performance reasons, I'd probably look into placing the json file in an internal stage next time, and executing the [`COPY INTO`](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table.html) command to speed things up.