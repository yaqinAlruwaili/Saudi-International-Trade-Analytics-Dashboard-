
# Python Ingestion Pipeline

This section automates the ingestion of raw Saudi international trade data and reference datasets into SQL Server staging area using Python, Pandas, and SQLAlchemy.

## 1. Global Configuration & Database Connection

This section configures the SQL Server connection and defines relative file paths for the raw trade datasets, HS codes reference file, and country CSV files. Using relative paths makes the project easier to run on different machines.

```python
import os
from pathlib import Path

import glob
import pandas as pd
import sqlalchemy as sa
from sqlalchemy import types

# ===================================================================
# GLOBAL CONFIGURATION & DATABASE CONNECTION
# ===================================================================
SERVER_NAME = "localhost"
DATABASE_NAME = "KSA_Trade_Warehouse"

CONNECTION_STRING = (
    f"mssql+pyodbc://@{SERVER_NAME}/{DATABASE_NAME}"
    "?driver=ODBC+Driver+17+for+SQL+Server&trusted_connection=yes"
)

engine = sa.create_engine(CONNECTION_STRING)

# Project directory (change this path if needed)
PROJECT_DIR = Path(__file__).resolve().parent

# Data file paths (relative to the project directory)
DATA_FOLDER = PROJECT_DIR / "Data"
TRADE_DATA_FOLDER = DATA_FOLDER / "exports_imports_data"
HS_CODE_FILE = DATA_FOLDER / "Dim_HS_Code.xlsx"
COUNTRY_FILE_1 = DATA_FOLDER / "countries1.csv"
COUNTRY_FILE_2 = DATA_FOLDER / "countries2.csv"
```

## 2. Part 1: Ingest Raw Trade Data (33 Excel Files)

This section automatically scans the source directory for Excel files containing raw import and export data. It enforces consistent SQL data types, preserves customs codes as strings to prevent leading-zero loss, and uploads each file into an individual SQL Server staging table.

```python
# ===================================================================
# PART 1: INGEST RAW TRADE DATA (33 EXCEL FILES)
# ===================================================================
print("Starting Part 1: Raw Trade Data Ingestion...")

trade_data_types = {
    "قيمة - ريال": types.Numeric(precision=18, scale=4),
    "الوزن الصافي بالكيلو": types.Numeric(precision=18, scale=4),
    "رمز التعرفة 4 خانات": types.VARCHAR(length=10),
    "السنة": types.INTEGER(),
    "الربع": types.VARCHAR(length=20),
    "الدولة": types.NVARCHAR(length=100),
}

excel_files = sorted(TRADE_DATA_FOLDER.glob("*.xlsx"))

if not TRADE_DATA_FOLDER.is_dir():
    print(f"Error: Trade data directory not found: {TRADE_DATA_FOLDER}")

elif not excel_files:
    print("Error: No Excel files (.xlsx) found in the trade data directory.")

else:
    print(f"Found {len(excel_files)} Excel files. Starting batch upload...")

    uploaded_files = 0

    for file_path in excel_files:
        file_name = file_path.name
        base_name = file_path.stem

        table_name = (
            base_name
            if base_name.startswith("stg_")
            else "stg_" + base_name
        )

        print(f"Uploading {file_name} -> {table_name} ...")

        try:
            df_trade = pd.read_excel(
                file_path,
                dtype={"رمز التعرفة 4 خانات": str},
            )
            df_trade.columns = df_trade.columns.str.strip()

            df_trade.to_sql(
                name=table_name,
                con=engine,
                if_exists="replace",
                index=False,
                dtype=trade_data_types,
            )

            uploaded_files += 1
            print(f"Success: {table_name} uploaded.")

        except Exception as error:
            print(f"Error uploading {file_name}: {error}")
            raise

    if uploaded_files == len(excel_files):
        print(
            f"Part 1 Completed: All {uploaded_files} files "
            "uploaded successfully.\n"
        )
    else:
        print(
            f"Warning: Only {uploaded_files} of "
            f"{len(excel_files)} files were uploaded.\n"
        )
```

### What It Does

- Automatically discovers Excel files in the raw trade data directory.
- Reads customs codes as strings to preserve their original format.
- Removes leading and trailing whitespace from column names.
- Applies predefined SQL data types to standardize the staging tables.
- Uploads each Excel file to a separate SQL Server staging table.
- Tracks successful uploads and raises an error if a file fails to upload.

## 3. Part 2: Ingest HS Codes Reference Table

This section loads the HS codes reference dataset, containing 97 Harmonized System (HS) chapters, into SQL Server as `Dim_HS_Code`. The table supports product classification and access to Arabic and English chapter descriptions.

```python
# ===================================================================
# PART 2: INGEST HS CODES REFERENCE TABLE
# ===================================================================
print("Starting Part 2: HS Codes Dimension Ingestion...")

if not HS_CODE_FILE.is_file():
    print(f"Error: HS Code master file not found: {HS_CODE_FILE}")

else:
    df_hs = pd.read_excel(HS_CODE_FILE)
    df_hs.columns = df_hs.columns.str.strip()

    hs_data_types = {
        "Chapter": types.VARCHAR(),
        "Arabic Description": types.NVARCHAR(length=255),
        "English Description": types.VARCHAR(length=255),
    }

    df_hs.to_sql(
        name="Dim_HS_Code",
        con=engine,
        if_exists="replace",
        index=False,
        dtype=hs_data_types,
    )

    print("Part 2 Completed: Dim_HS_Code uploaded successfully.\n")
```

### What It Does

- Checks whether the HS codes reference file exists.
- Reads the Excel reference dataset using Pandas.
- Standardizes column names by trimming whitespace.
- Defines SQL data types for chapter codes and descriptions.
- Uploads the reference data to the `Dim_HS_Code` table.

## 4. Part 3: Preprocess & Ingest Unique Countries Reference Table

This section combines country records from two CSV files, removes missing and duplicate values, trims whitespace, filters out blank entries, and sorts the remaining country names alphabetically. The cleaned dataset is uploaded to SQL Server as `Dim_Unique_Countries`.

```python
# ===================================================================
# PART 3: PREPROCESS & INGEST UNIQUE COUNTRIES REFERENCE TABLE
# ===================================================================
print("Starting Part 3: Unique Countries Preprocessing and Ingestion...")

if not COUNTRY_FILE_1.is_file() or not COUNTRY_FILE_2.is_file():
    print("Error: One or both country CSV files were not found.")

else:
    # 1. Read the two raw source files
    df1 = pd.read_csv(COUNTRY_FILE_1)
    df2 = pd.read_csv(COUNTRY_FILE_2)

    # 2. Combine country columns, clean values, remove duplicates, and sort
    all_countries = pd.concat(
        [df1["Country_Name"], df2["Country_Name"]],
        ignore_index=True,
    )

    unique_countries = (
        all_countries
        .dropna()
        .str.strip()
    )

    unique_countries = (
        unique_countries[unique_countries.ne("")]
        .drop_duplicates()
        .sort_values()
    )

    # 3. Create a clean DataFrame with standardized naming
    df_countries = pd.DataFrame({
        "Country_Name_Ar": unique_countries
    })

    country_data_types = {
        "Country_Name_Ar": types.NVARCHAR(length=100)
    }

    # 4. Upload the cleaned country reference table
    df_countries.to_sql(
        name="Dim_Unique_Countries",
        con=engine,
        if_exists="replace",
        index=False,
        dtype=country_data_types,
    )

    print(
        "Part 3 Completed: Unique countries cleaned and "
        "Dim_Unique_Countries uploaded successfully.\n"
    )

print("Pipeline execution finished.")
```

### What It Does

- Reads country data from two CSV files.
- Combines the country-name columns into a single series.
- Removes missing values, whitespace-only entries, and duplicates.
- Sorts the cleaned country names alphabetically.
- Creates a standardized `Country_Name_Ar` column.
- Uploads the cleaned country reference dataset to `Dim_Unique_Countries`.

## 5. Summary

The Python ingestion pipeline prepares and loads three groups of datasets into SQL Server.

| Source Dataset | Destination Table(s) | Purpose |
|---|---|---|
| 33 raw Excel files | Individual `stg_` staging tables | Store raw import and export data |
| HS codes Excel file | `Dim_HS_Code` | Provide HS chapter classifications and descriptions |
| Two country CSV files | `Dim_Unique_Countries` | Provide a consolidated country reference list |

These tables provide the raw trade data and reference datasets required for the subsequent SQL Server ETL process and Power BI dashboard.

