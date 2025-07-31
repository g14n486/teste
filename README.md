pip install investpy pandas pyarrow```

Now, you can use the following Python code:

```python
import pandas as pd
import investpy

def get_financial_data(ticker, country='brazil'):
    """
    Retrieves fundamental financial data for a given company.

    Args:
        ticker (str): The company's ticker symbol (e.g., 'WEGE3').
        country (str): The country where the company is listed.

    Returns:
        pandas.DataFrame: A DataFrame containing the financial data,
                          or None if the data cannot be retrieved.
    """
    try:
        print(f"Fetching financial data for {ticker} in {country}...")
        # Fetch income statement and balance sheet data
        income_statement = investpy.get_stock_financial_summary(
            stock=ticker,
            country=country,
            summary_type='income_statement',
            period='annual'
        )

        balance_sheet = investpy.get_stock_financial_summary(
            stock=ticker,
            country=country,
            summary_type='balance_sheet',
            period='annual'
        )

        # Extract relevant data
        total_revenue = income_statement.loc[income_statement['Metric'] == 'Total Revenue'].iloc[:, 1:]
        ebitda = income_statement.loc[income_statement['Metric'] == 'EBITDA'].iloc[:, 1:]
        net_income = income_statement.loc[income_statement['Metric'] == 'Net Income'].iloc[:, 1:]
        total_assets = balance_sheet.loc[balance_sheet['Metric'] == 'Total Assets'].iloc[:, 1:]
        total_liabilities = balance_sheet.loc[balance_sheet['Metric'] == 'Total Liabilities'].iloc[:, 1:]

        # Create a new DataFrame for the selected metrics
        financial_data = pd.concat([total_revenue, ebitda, net_income, total_assets, total_liabilities])
        financial_data.index = ['Total Revenue', 'EBITDA', 'Net Income', 'Total Assets', 'Total Liabilities']

        # Convert data to numeric, coercing errors to NaN
        for col in financial_data.columns:
            financial_data[col] = pd.to_numeric(financial_data[col], errors='coerce')

        # Calculate margins and other indicators
        financial_data.loc['EBITDA Margin (%)'] = (financial_data.loc['EBITDA'] / financial_data.loc['Total Revenue']) * 100
        financial_data.loc['Net Margin (%)'] = (financial_data.loc['Net Income'] / financial_data.loc['Total Revenue']) * 100
        financial_data.loc['Debt to Assets'] = financial_data.loc['Total Liabilities'] / financial_data.loc['Total Assets']


        return financial_data.transpose()

    except Exception as e:
        print(f"An error occurred: {e}")
        return None

def save_to_parquet(data, filename="financial_data.parquet"):
    """
    Saves a Pandas DataFrame to a Parquet file.

    Args:
        data (pandas.DataFrame): The DataFrame to save.
        filename (str): The name of the output Parquet file.
    """
    if data is not None and not data.empty:
        try:
            data.to_parquet(filename) # Use pandas.DataFrame.to_parquet to save the data. [1, 2, 3, 4, 5]
            print(f"Data successfully saved to {filename}")
        except Exception as e:
            print(f"Could not save data to Parquet file: {e}")
    else:
        print("No data to save.")

if __name__ == '__main__':
    # Get user input for company ticker and filename
    company_ticker = input("Enter the company ticker (e.g., WEGE3): ")
    output_filename = input("Enter the desired output filename (without extension): ")

    # Get the financial data
    data = get_financial_data(company_ticker)

    # Save the data to a .parquet file
    if data is not None:
        save_to_parquet(data, f"{output_filename}.parquet")
