# MySQL-Snowflake-ETL-pipeline

# Python ETL for Airbnb Data (MySQL to Snowflake) 💾➡️❄️

This project implements a simple Extract, Transform, Load (ETL) pipeline written in Python using a Jupyter Notebook. It extracts Airbnb listing data from a **MySQL** database, performs data cleaning and type conversion using **Pandas**, and then loads the cleaned data into a **Snowflake** data warehouse.

---

## 🚀 Overview

The pipeline executes the following steps:

1.  **Extract:** Connects to a local MySQL database (`my_db_etl`) and extracts the data from the `airbnb` table into a Pandas DataFrame.
2.  **Transform:**
    * Selects a specific subset of columns.
    * Converts all column names to uppercase.
    * Removes currency symbols (`$`, `,`) and spaces from the `PRICE` and `SERVICE_FEE` columns.
    * Converts key columns (`CONSTRUCTION_YEAR`, `PRICE`, `SERVICE_FEE`, `MINIMUM_NIGHTS`, `NUMBER_OF_REVIEWS`, `LAT`, `LONG`) to appropriate numerical or datetime data types, handling empty strings by converting them to `NaN` (nulls).
    * Checks for missing values and duplicates (though no explicit handling is shown beyond checking).
3.  **Load:** Connects to a Snowflake warehouse (`COMPUTE_WH` in `DEMO_DB.PUBLIC`) and loads the processed DataFrame into a new or replaced table named `AIRBNB_LISTINGS`.

---

## 🛠️ Setup and Installation

### Prerequisites

* Python (3.x recommended)
* A running **MySQL** server with the `my_db_etl` database and `airbnb` table.
* A **Snowflake** account with credentials and necessary database/schema privileges.
* The `python_etl.ipynb` file.

### Installation

The project relies on several Python libraries. Run the following command (or the first code cell in the notebook) to install the dependencies:

```bash
pip install mysql-connector-python pandas snowflake-connector-python python-dotenv
Configuration
The MySQL and Snowflake connection parameters are hardcoded in the notebook but would typically be managed via environment variables (e.g., in a .env file) for security, as suggested by the from dotenv import load_dotenv import.

MySQL Connection (Extraction):

Python

connection = mysql.connector.connect(
    host="localhost",
    user="root",
    password="<YOUR_MYSQL_PASSWORD>", 
    database="my_db_etl",
    port=3306
)
Snowflake Connection (Loading):

Python

conn = snowflake.connector.connect(
    user="<YOUR_SNOWFLAKE_USER>", 
    password="<YOUR_SNOWFLAKE_PASSWORD>", 
    account="<YOUR_SNOWFLAKE_ACCOUNT>",
    warehouse="COMPUTE_WH",
    database="DEMO_DB",
    schema="PUBLIC"
)
Note: Replace the placeholder values (e.g., <YOUR_MYSQL_PASSWORD>) with your actual credentials.

💻 Core ETL Code Logic
The following code snippets represent the key stages of the ETL process.

1. Extract and Initial Load
Loads data from MySQL into a Pandas DataFrame.

Python

import pandas as pd
import mysql.connector
import snowflake.connector
import os
# from dotenv import load_dotenv # Suggested for secure credential handling

# MySQL Connection setup...

query = "SELECT * FROM airbnb;"
df = pd.read_sql(query, connection)
2. Transform (Cleaning and Structuring)
Selects columns, standardizes column names, and cleans price/fee columns.

Python

# Select and rename columns
columns_to_keep = [ 'name', 'host_id', 'host_identity_verified', 'host_name',
                   'neighbourhood_group', 'neighbourhood', 'lat', 'long', 'country',
                   'country_code', 'instant_bookable', 'cancellation_policy', 'room_type',
                   'construction_year', 'price', 'service_fee', 'minimum_nights',
                   'number_of_reviews', 'last_review']

df = df[columns_to_keep]
df.columns = [x.upper() for x in df.columns]

# Clean PRICE and SERVICE_FEE columns
df['PRICE'] = df['PRICE'].str.replace('$', '').str.replace(',', '').str.replace(' ', '')
df['SERVICE_FEE'] = df['SERVICE_FEE'].str.replace('$', '').str.replace(',', '').str.replace(' ', '')

# Convert data types
import numpy as np
df_to_int = ['CONSTRUCTION_YEAR', 'PRICE', 'SERVICE_FEE', 'MINIMUM_NIGHTS', 'NUMBER_OF_REVIEWS']
for col in df_to_int:
    df[col] = df[col].replace('', np.nan)
    df[col] = pd.to_numeric(df[col], errors='coerce').astype('Int64')

df['LAST_REVIEW'] = df['LAST_REVIEW'].replace(" ", "")
df['LAST_REVIEW'] = pd.to_datetime(df['LAST_REVIEW'] , format='%m/%d/%Y')
df[['LAT', 'LONG']] = df[['LAT', 'LONG']].apply(pd.to_numeric)
3. Load
Creates/replaces the target table in Snowflake and loads the data using write_pandas.

Python

from snowflake.connector.pandas_tools import write_pandas

# Snowflake Connection setup...
cs = conn.cursor()

# Define and create/replace target table (Note: LAST_REVIEW is loaded as a STRING here)
cs.execute("""
CREATE OR REPLACE TABLE AIRBNB_LISTINGS (
    NAME STRING,
    HOST_ID STRING,
    HOST_IDENTITY_VERIFIED STRING,
    HOST_NAME STRING,
    NEIGHBOURHOOD_GROUP STRING,
    NEIGHBOURHOOD STRING,
    LAT FLOAT,
    LONG FLOAT,
    COUNTRY STRING,
    COUNTRY_CODE STRING,
    INSTANT_BOOKABLE STRING,
    CANCELLATION_POLICY STRING,
    ROOM_TYPE STRING,
    CONSTRUCTION_YEAR NUMBER(4,0),
    PRICE NUMBER(10,2),
    SERVICE_FEE NUMBER(10,2),
    MINIMUM_NIGHTS NUMBER(10,0),
    NUMBER_OF_REVIEWS NUMBER(10,0),
    LAST_REVIEW STRING
);""")

# Load DataFrame into Snowflake
success, nchunks, nrows, _ = write_pandas(conn, df[['NAME', 'HOST_ID', 'HOST_IDENTITY_VERIFIED', 'HOST_NAME',
       'NEIGHBOURHOOD_GROUP', 'NEIGHBOURHOOD', 'LAT', 'LONG', 'COUNTRY',
       'COUNTRY_CODE', 'INSTANT_BOOKABLE', 'CANCELLATION_POLICY', 'ROOM_TYPE',
       'CONSTRUCTION_YEAR', 'PRICE', 'SERVICE_FEE', 'MINIMUM_NIGHTS',
       'NUMBER_OF_REVIEWS', 'LAST_REVIEW']], 'AIRBNB_LISTINGS')

print(f"✅ Loaded {nrows} rows into Snowflake table 'AIRBNB_LISTINGS'.")
    
cs.close()
conn.close()
