
## T-SQL Data Warehouse Development

This section contains the database codes executed inside Microsoft SQL Server (SSMS) to create the clean relational schema and run the dynamic data merge.

### 1. Database Schema Generation (`create_fact_tables.sql`)

This code creates the final unified Fact tables with matching forced data types to store the 10-year historical trade data.

```sql
USE KSA_Trade_Warehouse;
GO

-- 1. Create the final unified Fact table for Imports
CREATE TABLE Fact_Imports (
    Transaction_Date DATE,
    Year_Int INT,
    Quarter_Str VARCHAR(10),
    Chapter_Code VARCHAR(2),
    HS_Code_4d VARCHAR(4),
    Country_Name NVARCHAR(100),
    Weight_KG NUMERIC(18, 4),
    Value_SAR NUMERIC(18, 4)
);

-- 2. Create the final unified Fact table for Exports
CREATE TABLE Fact_Exports (
    Transaction_Date DATE,
    Year_Int INT,
    Quarter_Str VARCHAR(10),
    Chapter_Code VARCHAR(2),
    HS_Code_4d VARCHAR(4),
    Country_Name NVARCHAR(100),
    Weight_KG NUMERIC(18, 4),
    Value_SAR NUMERIC(18, 4)
);
GO
```

### 2. The Dynamic SQL ETL Ingestion Script (`run_dynamic_etl.sql`)

This advanced code uses a T-SQL Cursor to automatically loop through all 33 staging tables, dynamically detect column names using keyword scanning, convert string quarters into true Date formats, and safely compress everything into the target Fact tables.

```sql
USE KSA_Trade_Warehouse;
GO

-- Empty the destination Fact tables first to avoid data duplication
TRUNCATE TABLE Fact_Imports;
TRUNCATE TABLE Fact_Exports;
GO

-- ===================================================================
-- PART 1: Dynamically Aggregate Import Staging Tables
-- ===================================================================
DECLARE @table_name NVARCHAR(256), @import_sql NVARCHAR(MAX) = '';
DECLARE import_cursor CURSOR FOR 
SELECT name FROM sys.tables WHERE name LIKE 'stg_Imports%';

OPEN import_cursor;
FETCH NEXT FROM import_cursor INTO @table_name;

WHILE @@FETCH_STATUS = 0
BEGIN
    DECLARE @col_val NVARCHAR(128), @col_wgt NVARCHAR(128), @col_hs NVARCHAR(128), @col_yr NVARCHAR(128), @col_qtr NVARCHAR(128), @col_cnt NVARCHAR(128);

    -- Smart column detection based on keyword matching
    SELECT @col_val = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name) AND name LIKE '%قيمة%';
    SELECT @col_wgt = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name) AND name LIKE '%الوزن%';
    SELECT @col_hs  = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name) AND name LIKE '%تعرفة%';
    SELECT @col_yr  = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name) AND (name LIKE '%السنة%' OR name LIKE '%سنة%');
    SELECT @col_qtr = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name) AND (name LIKE '%الربع%' OR name LIKE '%ربع%');
    SELECT @col_cnt = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name) AND (name LIKE '%الدولة%' OR name LIKE '%دولة%');

    -- Build and generate the SQL insert script dynamically for the current loop table
    SET @import_sql = '
    INSERT INTO Fact_Imports (Transaction_Date, Year_Int, Quarter_Str, Chapter_Code, HS_Code_4d, Country_Name, Weight_KG, Value_SAR)
    SELECT 
        CASE 
            WHEN CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(50)) LIKE ''%Q1%'' THEN CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-01-01'' AS DATE)
            WHEN CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(50)) LIKE ''%Q2%'' THEN CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-04-01'' AS DATE)
            WHEN CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(50)) LIKE ''%Q3%'' THEN CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-07-01'' AS DATE)
            WHEN CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(50)) LIKE ''%Q4%'' THEN CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-10-01'' AS DATE)
            ELSE CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-01-01'' AS DATE)
        END,
        CAST(' + QUOTENAME(@col_yr) + ' AS INT), CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(10)),
        SUBSTRING(RIGHT(''0000'' + REPLACE(CAST(' + QUOTENAME(@col_hs) + ' AS VARCHAR(20)), ''.0'', ''''), 4), 1, 2),
        RIGHT(''0000'' + REPLACE(CAST(' + QUOTENAME(@col_hs) + ' AS VARCHAR(20)), ''.0'', ''''), 4),
        CAST(' + QUOTENAME(@col_cnt) + ' AS NVARCHAR(100)), CAST(' + QUOTENAME(@col_wgt) + ' AS NUMERIC(18,4)), CAST(' + QUOTENAME(@col_val) + ' AS NUMERIC(18,4))
    FROM ' + QUOTENAME(@table_name) + ';';

    EXEC sp_executesql @import_sql;
    FETCH NEXT FROM import_cursor INTO @table_name;
END;
CLOSE import_cursor;
DEALLOCATE import_cursor;
GO

-- ===================================================================
-- PART 2: Dynamically Aggregate Export Staging Tables
-- ===================================================================
DECLARE @table_name_exp NVARCHAR(256), @export_sql NVARCHAR(MAX) = '';
DECLARE export_cursor CURSOR FOR 
SELECT name FROM sys.tables WHERE name LIKE 'stg_Exports%';

OPEN export_cursor;
FETCH NEXT FROM export_cursor INTO @table_name_exp;

WHILE @@FETCH_STATUS = 0
BEGIN
    DECLARE @col_val NVARCHAR(128), @col_wgt NVARCHAR(128), @col_hs NVARCHAR(128), @col_yr NVARCHAR(128), @col_qtr NVARCHAR(128), @col_cnt NVARCHAR(128);

    SELECT @col_val = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name_exp) AND name LIKE '%قيمة%';
    SELECT @col_wgt = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name_exp) AND name LIKE '%الوزن%';
    SELECT @col_hs  = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name_exp) AND name LIKE '%تعرفة%';
    SELECT @col_yr  = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name_exp) AND (name LIKE '%السنة%' OR name LIKE '%سنة%');
    SELECT @col_qtr = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name_exp) AND (name LIKE '%الربع%' OR name LIKE '%ربع%');
    SELECT @col_cnt = name FROM sys.columns WHERE object_id = OBJECT_ID(@table_name_exp) AND (name LIKE '%الدولة%' OR name LIKE '%دولة%');

    SET @export_sql = '
    INSERT INTO Fact_Exports (Transaction_Date, Year_Int, Quarter_Str, Chapter_Code, HS_Code_4d, Country_Name, Weight_KG, Value_SAR)
    SELECT 
        CASE 
            WHEN CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(50)) LIKE ''%Q1%'' THEN CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-01-01'' AS DATE)
            WHEN CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(50)) LIKE ''%Q2%'' THEN CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-04-01'' AS DATE)
            WHEN CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(50)) LIKE ''%Q3%'' THEN CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-07-01'' AS DATE)
            WHEN CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(50)) LIKE ''%Q4%'' THEN CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-10-01'' AS DATE)
            ELSE CAST(CAST(' + QUOTENAME(@col_yr) + ' AS VARCHAR(4)) + ''-01-01'' AS DATE)
        END,
        CAST(' + QUOTENAME(@col_yr) + ' AS INT), CAST(' + QUOTENAME(@col_qtr) + ' AS VARCHAR(10)),
        SUBSTRING(RIGHT(''0000'' + REPLACE(CAST(' + QUOTENAME(@col_hs) + ' AS VARCHAR(20)), ''.0'', ''''), 4), 1, 2),
        RIGHT(''0000'' + REPLACE(CAST(' + QUOTENAME(@col_hs) + ' AS VARCHAR(20)), ''.0'', ''''), 4),
        CAST(' + QUOTENAME(@col_cnt) + ' AS NVARCHAR(100)), CAST(' + QUOTENAME(@col_wgt) + ' AS NUMERIC(18,4)), CAST(' + QUOTENAME(@col_val) + ' AS NUMERIC(18,4))
    FROM ' + QUOTENAME(@table_name_exp) + ';';

    EXEC sp_executesql @export_sql;
    FETCH NEXT FROM export_cursor INTO @table_name_exp;
END;
CLOSE export_cursor;
DEALLOCATE export_cursor;
GO
```
