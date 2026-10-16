
## Python Ingestion section

This section contains the automation codes used to scan directories and ingest the raw dataset into the SQL Server staging area.

### 1- Trade Data Ingestion Code (`ingest_trade_data_to_sql.py`)

```python
import os
import glob
import pandas as pd
import sqlalchemy as sa
from sqlalchemy import types

# 1. Define the directory path containing the 33 raw Excel files
FOLDER_PATH = r"C:\Users\your_username\Desktop\Saudi_Trade_Project\Data\Raw_Files"

# 2. Configure the SQL Server database connection string
server_name = 'localhost'
database_name = 'KSA_Trade_Warehouse'
connection_string = f"mssql+pyodbc://@{server_name}/{database_name}?driver=ODBC+Driver+17+for+SQL+Server&trusted_connection=yes"
engine = sa.create_engine(connection_string)

# 3. Define strict data types to ensure a seamless SQL UNION later
sql_data_types = {
    'قيمة - ريال': types.Numeric(precision=18, scale=4),
    'الوزن الصافي بالكيلو': types.Numeric(precision=18, scale=4),
    'رمز التعرفة 4 خانات': types.VARCHAR(length=10), # Read as text to protect leading zeros
    'السنة': types.INTEGER(),
    'الربع': types.VARCHAR(length=20),
    'الدولة': types.NVARCHAR(length=100) # Supports Arabic character encoding
}

# 4. Automatically discover all Excel (.xlsx) files within the target folder
excel_files = glob.glob(os.path.join(FOLDER_PATH, "*.xlsx"))

if not excel_files:
    print("❌ No Excel files (.xlsx) found in the specified directory. Please check the path!")
else:
    print(f" Found {len(excel_files)} files. Starting strict pipeline ingestion...")

    # 5. Loop through each discovered file and upload it dynamically
    for file_path in excel_files:
        # Extract the file name without the extension to use as the base SQL table name
        file_name = os.path.basename(file_path)
        table_name = os.path.splitext(file_name)[0]

        # Prepend 'stg_' prefix to the table name for staging area organization
        if not table_name.startswith('stg_'):
            table_name = 'stg_' + table_name

        print(f"Uploading: {file_name} To SQL Table: {table_name} ...")

        # Read Excel sheet and explicitly force customs codes to strings to protect leading zeros
        df = pd.read_excel(file_path, dtype={'رمز التعرفة 4 خانات': str})

        # Clean up column headers by trimming any accidental hidden leading/trailing spaces
        df.columns = df.columns.str.strip()

        # Ingest dataframe into the warehouse with the predefined strict schemas
        df.to_sql(
            name=table_name,
            con=engine,
            if_exists='replace',  # Overwrite previous staging tables if they exist
            index=False,
            dtype=sql_data_types
        )
        print(f"Table {table_name} has been successfully uploaded with standardized data types!")

    print("\nAll files have been uploaded successfully!")
```

**What it does:** Automatically loops through all 33 raw Excel files in the data folder, cleans the text columns, enforces strict data types, and uploads them as staging tables into SQL Server while protecting leading zeros in customs codes.

### 2- HS Codes Dimension Code (`ingest_hs_codes_to_sql.py`)

```python
import pandas as pd
import sqlalchemy as sa
from sqlalchemy import types

# 1. Configure the SQL Server database connection string
server_name = 'localhost'
database_name = 'KSA_Trade_Warehouse'
connection_string = f"mssql+pyodbc://@{server_name}/{database_name}?driver=ODBC+Driver+17+for+SQL+Server&trusted_connection=yes"
engine = sa.create_engine(connection_string)

# 2. Define the exact path to the Master HS Code reference table
file_path = r"C:\Users\your_username\Desktop\Saudi_Trade_Project\Data\Dim_HS_Code.xlsx"

# 3. Load the reference master table into the ingestion memory
df = pd.read_excel(file_path)

# Clean up column headers by trimming any hidden white spaces
df.columns = df.columns.str.strip()

# 4. Enforce clean data dimension types for the SQL catalog
dim_data_types = {
    'Chapter': types.VARCHAR(), # Stores codes as text strings without length restrictions
    'Arabic Description': types.NVARCHAR(length=255), # Safely stores full Arabic text strings
    'English Description': types.VARCHAR(length=255)
}

# 5. Ingest the data into the main database environment under 'Dim_HS_Code'
df.to_sql(
    name='Dim_HS_Code',
    con=engine,
    if_exists='replace',
    index=False,
    dtype=dim_data_types
)

print("All reference tables have been uploaded successfully!")
```

**What it does:** Uploads the master lookup table containing the 97 global customs chapters into SQL Server to act as the primary product reference dimension for filtering the trade database by English and Arabic descriptions.
