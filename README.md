Got it. You want to create a DataFrame using only the ibov.csv file for now, because the SQLite data is not ready. The goal is to get a DataFrame with date and ticker columns.

Important Clarification: A standard ibov.csv file contains daily data for the IBOVESPA index. A single row represents a day for the entire market index, so it has columns like date, open, close, etc., but it does not have a column for an individual company ticker.

Therefore, the correct approach is to use ibov.csv to create a list of all valid trading dates. We can then create a DataFrame with this date column and add a placeholder column for the ticker, which you can replace later when your CVM data is ready.

This script will:

Load only the 'date' column from ibov.csv for efficiency.

Clean and format these dates.

Create a new column named ticker and fill it with a placeholder value (e.g., 'IBOV').

Save this new, simple DataFrame to a Parquet file.

Expected Folder Structure

The script assumes your project is set up like this:

Generated code
your_project/
├── data/
│   └── processed/
│       └── ibov.csv  <-- The only input file we need
│
├── output/           <-- The script will create this folder
│
└── scripts/
    └── 03_create_market_dates_index.py <-- Place the Python code here

Python Script: 03_create_market_dates_index.py

This script is designed to be clear and focused on your current requirement.

Generated python
# -*- coding: utf-8 -*-

# ==============================================================================
# Step 0: Import Libraries
# ==============================================================================
import pandas as pd
from pathlib import Path
import os

print("Starting script to create an index file from IBOV dates...")

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
IBOV_CSV_FILE = DATA_PATH / 'ibov.csv'

# --- Output File ---
OUTPUT_FILE = OUTPUT_PATH / 'market_trading_dates.parquet' 

# --- Column Names Configuration ---
# The name of the date column in your ibov.csv file
IBOV_DATE_COL = 'date' 
# Placeholder value for the ticker column since it's not in the source file
PLACEHOLDER_TICKER = 'IBOV' 

# Desired column names in the final output file
OUTPUT_TICKER_COL = 'ticker'
OUTPUT_DATE_COL = 'date'


# ==============================================================================
# Step 2: Create DataFrame from IBOV Dates
# ==============================================================================
try:
    print(f"Loading dates from: {IBOV_CSV_FILE}")
    
    # Efficiently load *only* the date column from the CSV
    df = pd.read_csv(IBOV_CSV_FILE, usecols=[IBOV_DATE_COL])
    
    print(f"Successfully loaded {df.shape[0]} date entries.")

except FileNotFoundError:
    print(f"\n❌ FILE NOT FOUND: The file was not found at {IBOV_CSV_FILE}")
    exit()
except ValueError as e:
    print(f"\n❌ COLUMN NOT FOUND: Make sure a column named '{IBOV_DATE_COL}' exists in the CSV. Error: {e}")
    exit()
except Exception as e:
    print(f"\n❌ An unexpected error occurred during file loading: {e}")
    exit()

# ==============================================================================
# Step 3: Process and Structure the DataFrame
# ==============================================================================
if not df.empty:
    print("\nProcessing and structuring the DataFrame...")

    # --- Standardize the date column ---
    # Rename the column to our standard name
    df.rename(columns={IBOV_DATE_COL: OUTPUT_DATE_COL}, inplace=True)
    
    # Convert the column to datetime objects
    df[OUTPUT_DATE_COL] = pd.to_datetime(df[OUTPUT_DATE_COL], errors='coerce')
    
    # Drop any rows that couldn't be converted to a valid date
    df.dropna(subset=[OUTPUT_DATE_COL], inplace=True)
    print("Standardized date column.")
    
    # --- Remove duplicate dates to get a unique list of trading days ---
    original_rows = len(df)
    df.drop_duplicates(inplace=True)
    print(f"Removed {original_rows - len(df)} duplicate dates.")

    # --- Add the placeholder ticker column ---
    df[OUTPUT_TICKER_COL] = PLACEHOLDER_TICKER
    print(f"Added placeholder ticker column with value '{PLACEHOLDER_TICKER}'.")

    # --- Reorder columns to the desired format [ticker, date] ---
    df = df[[OUTPUT_TICKER_COL, OUTPUT_DATE_COL]]
    
    # --- Sort the final DataFrame ---
    df.sort_values(by=OUTPUT_DATE_COL, ascending=True, inplace=True)
    print("DataFrame sorted by date.")

    # ==============================================================================
    # Step 4: Save the Final DataFrame
    # ==============================================================================
    try:
        df.to_parquet(OUTPUT_FILE, index=False)
        print(f"\n✅ Success! The market dates file has been saved to:\n{OUTPUT_FILE}")
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

Check Configuration: Open the script and look at Step 1. Ensure IBOV_DATE_COL matches the name of the date column in your ibov.csv file. You can also change the PLACEHOLDER_TICKER if you wish.

Save: Save the code as 03_create_market_dates_index.py inside your scripts/ folder.

Run from Terminal:

Navigate to your project's root directory (your_project/).

Activate your Python virtual environment.

Execute the script:

Generated bash
python scripts/03_create_market_dates_index.py
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

The result will be a new file, market_trading_dates.parquet, in your output folder. It will contain a clean, sorted list of all unique trading days found in your IBOV file, perfect for use as a base for future data merging.
