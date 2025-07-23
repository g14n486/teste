Of course. This is a common and useful task: creating a simplified "index" or "lookup" file from a more complex dataset.

Based on your request, it seems the ibov.csv file is not actually needed for this specific goal, because the necessary information (the company ticker and the date of the event) should both be present in your cvm_data.sqlite file.

The plan is to:

Load the data directly from your cvm_data.sqlite file.

Identify the specific columns that contain the company ticker and the event date.

Extract only these two columns into a new DataFrame.

Clean and format the data (ensure dates are correct, remove duplicates).

Save this new, lightweight DataFrame to a Parquet file.

This approach is more efficient as it avoids loading the large ibov.csv file unnecessarily.

Expected Folder Structure

The script assumes your project is set up like this:

Generated code
your_project/
├── data/
│   └── processed/
│       └── cvm_data.sqlite  <-- The only input file we need
│
├── output/                  <-- The script will create this folder
│
└── scripts/
    └── 02_create_ticker_date_index.py <-- Place the Python code here

Python Script: 02_create_ticker_date_index.py

This script is designed to be easily configurable. You can adjust the column names in the "Configuration" section to match your exact file schema.

Generated python
# -*- coding: utf-8 -*-

# ==============================================================================
# Step 0: Import Libraries
# ==============================================================================
import pandas as pd
import sqlite3
from pathlib import Path
import os

print("Starting script to create a ticker-date index file...")

# ==============================================================================
# Step 1: Configuration
# ==============================================================================
# --- Define the Project Root Path ---
ROOT_PATH = Path(__file__).resolve().parent.parent

# --- Define Paths for Input and Output ---
DATA_PATH = ROOT_PATH / 'data' / 'processed'
OUTPUT_PATH = ROOT_PATH / 'output'

# Ensure the output directory exists
os.makedirs(OUTPUT_PATH, exist_ok=True)

# --- Input File ---
CVM_SQLITE_FILE = DATA_PATH / 'cvm_data.sqlite'

# --- Output File ---
# Parquet is recommended for efficiency and type preservation
OUTPUT_FILE = OUTPUT_PATH / 'company_event_dates.parquet' 

# --- Column Names (IMPORTANT: ADJUST THESE TO MATCH YOUR SQLITE TABLE) ---
# The name of the column in your SQLite table that contains the company ticker
# This might be 'ticker', 'CD_CVM', 'DENOM_CIA', etc.
CVM_TICKER_COL = 'CD_CVM' 
# The name of the column in your SQLite table that contains the event date
CVM_DATE_COL = 'DT_COMPTC'

# --- Desired Column Names in the Output File ---
OUTPUT_TICKER_COL = 'ticker'
OUTPUT_DATE_COL = 'date'


# ==============================================================================
# Step 2: Load Required Data from SQLite
# ==============================================================================
try:
    print(f"Connecting to database: {CVM_SQLITE_FILE}")
    conn = sqlite3.connect(CVM_SQLITE_FILE)

    # Automatically find the first table name in the database
    table_name_query = "SELECT name FROM sqlite_master WHERE type='table';"
    table_name = pd.read_sql_query(table_name_query, conn).iloc[0, 0]
    
    print(f"Reading columns '{CVM_TICKER_COL}' and '{CVM_DATE_COL}' from table '{table_name}'...")
    
    # Efficiently select only the columns we need
    query = f'SELECT "{CVM_TICKER_COL}", "{CVM_DATE_COL}" FROM {table_name}'
    
    df = pd.read_sql_query(query, conn)
    
    conn.close()

except sqlite3.Error as e:
    print(f"\n❌ SQL ERROR: Could not read the database. Error: {e}")
    exit()
except FileNotFoundError:
    print(f"\n❌ FILE NOT FOUND: The database file was not found at {CVM_SQLITE_FILE}")
    exit()
except Exception as e:
    print(f"\n❌ An unexpected error occurred during data loading: {e}")
    exit()

print(f"Successfully loaded {df.shape[0]} rows.")

# ==============================================================================
# Step 3: Process and Clean the Extracted Data
# ==============================================================================
if not df.empty:
    print("\nProcessing data...")

    # --- Rename columns to the desired output names ---
    df.rename(columns={
        CVM_TICKER_COL: OUTPUT_TICKER_COL,
        CVM_DATE_COL: OUTPUT_DATE_COL
    }, inplace=True)
    print(f"Renamed columns to '{OUTPUT_TICKER_COL}' and '{OUTPUT_DATE_COL}'.")

    # --- Clean the data ---
    # Drop rows where either ticker or date is missing
    original_rows = len(df)
    df.dropna(subset=[OUTPUT_TICKER_COL, OUTPUT_DATE_COL], inplace=True)
    print(f"Dropped {original_rows - len(df)} rows with missing data.")
    
    # Convert date column to datetime objects
    df[OUTPUT_DATE_COL] = pd.to_datetime(df[OUTPUT_DATE_COL], errors='coerce')
    df.dropna(subset=[OUTPUT_DATE_COL], inplace=True) # Drop rows where date conversion failed
    print("Standardized date column to datetime format.")

    # Remove any fully duplicate rows
    original_rows = len(df)
    df.drop_duplicates(inplace=True)
    print(f"Dropped {original_rows - len(df)} duplicate rows.")

    # ==============================================================================
    # Step 4: Sort the Final DataFrame
    # ==============================================================================
    print("Sorting data by ticker and date...")
    df.sort_values(by=[OUTPUT_TICKER_COL, OUTPUT_DATE_COL], ascending=[True, True], inplace=True)

    # ==============================================================================
    # Step 5: Save the Final DataFrame
    # ==============================================================================
    try:
        df.to_parquet(OUTPUT_FILE, index=False)
        print(f"\n✅ Success! The index file has been saved to:\n{OUTPUT_FILE}")
        print("\nFinal data preview:")
        print(df.head())

    except Exception as e:
        print(f"\n❌ ERROR: Could not save the Parquet file. Reason: {e}")

else:
    print("\n⚠️ The loaded DataFrame is empty. No output file was created.")
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Python
IGNORE_WHEN_COPYING_END
How to Use

Configure: The most important step is to open the script and edit the variables in "Step 1: Configuration". Make sure CVM_TICKER_COL and CVM_DATE_COL exactly match the column names in your cvm_data.sqlite table.

Save: Save the code as 02_create_ticker_date_index.py inside your scripts/ folder.

Run from Terminal:

Navigate to your project's root directory (your_project/).

Activate your Python virtual environment (e.g., source .venv/bin/activate).

Execute the script:

Generated bash
python scripts/02_create_ticker_date_index.py
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

After running, the output folder will contain the file company_event_dates.parquet. This file will have just two columns, ticker and date, and will be clean, sorted, and ready for use in your projects.
