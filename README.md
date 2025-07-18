Building a Market Reaction Model with LSTM: A Sound Approach

Yes, building a Long Short-Term Memory (LSTM) model to predict market reactions to company press releases is a very good idea. LSTMs are a type of recurrent neural network (RNN) particularly well-suited for analyzing time-series data, which is central to your project.[1][2] Your model's success will hinge on its ability to process and find patterns in sequential data, and LSTMs are designed to remember long-term dependencies, a key feature when dealing with financial markets.[3][4]

Your proposed multi-faceted approach, incorporating sentiment analysis, financial data (both realized and expected), and macroeconomic variables, is comprehensive. Combining sentiment data from news and social media with financial metrics has been shown to improve stock market prediction accuracy.[5][6][7] This is because market movements are influenced by a variety of factors, including investor sentiment, which can be captured through the nuanced analysis of text.[5][8]
Analysis of the Provided Code

The Python code you've provided is a solid starting point for time-series forecasting. It demonstrates the use of LSTMs, as well as other models like CNNs and GRUs, for predicting weather data. The code effectively covers key steps such as:

    Data Loading and Preprocessing: Reading a CSV file, parsing date-time information, and creating a time-series dataframe.

    Feature Engineering: Creating cyclical features from timestamps (e.g., 'Day sin', 'Day cos').

    Data Structuring for LSTMs: Transforming the data into sequences of a specified window size.

    Model Building and Training: Defining, compiling, and fitting LSTM models using TensorFlow and Keras.

    Evaluation: Using metrics like Mean Squared Error and plotting predictions against actual values.

    Multivariate Forecasting: Extending the model to predict multiple variables simultaneously.

Demo for Your Use Case: Predicting Market Reaction

Here's a conceptual demonstration of how you can adapt the provided code for your specific use case. This demo will outline the necessary steps and provide a corresponding Python code snippet.

Conceptual Steps:

    Data Acquisition and Merging:

        Gather your data: press releases, historical stock prices, macroeconomic data (e.g., interest rates, inflation), and financial data (realized and expected earnings).

        Align the data by date. You'll need to associate each press release with the corresponding financial and market data for that day.

    Sentiment Analysis:

        For each press release, perform sentiment analysis on two levels:

            Content Sentiment: The sentiment of the factual information presented (e.g., positive for increased profits, negative for a lawsuit).

            Wording Sentiment: The sentiment conveyed by the language used, considering nuances and tone.

        You can use pre-trained models like FinBERT for financial text sentiment analysis or train your own.[9][10] The output should be numerical sentiment scores.

    Feature Engineering and Preprocessing:

        Combine all your features into a single DataFrame indexed by date. Your features will include:

            Content Sentiment Score

            Wording Sentiment Score

            Realized Financial Data (e.g., revenue, EPS)

            Expected Financial Data

            Macroeconomic Variables

            Historical Stock Data (e.g., opening price, closing price, volume)

        Normalize or scale all numerical features to be within a similar range, which is crucial for the performance of neural networks.[11][12]

    Model Architecture and Training:

        Structure your data into sequences (windows) as demonstrated in the provided code. Each sequence will contain the features for a set number of past days, and the target variable will be the market reaction on the following day (e.g., percentage change in stock price).

        Build a multivariate LSTM model. You can start with a simple architecture similar to model5 in your code and experiment with more complex structures, like stacked LSTMs (model6), as needed.[13]

        The Keras Functional API is particularly useful for building models with multiple, potentially different, types of inputs.[14]
        
        
import tensorflow as tf
import os
import pandas as pd
import numpy as np
from sklearn.preprocessing import MinMaxScaler
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import LSTM, Dense, InputLayer
from tensorflow.keras.optimizers import Adam
from tensorflow.keras.losses import MeanSquaredError

# --- 1. Data Acquisition and Merging (Conceptual) ---
# Assume you have a DataFrame `merged_df` with the following columns:
# 'Date', 'Stock_Price_Change', 'Content_Sentiment', 'Wording_Sentiment',
# 'Realized_EPS', 'Expected_EPS', 'Interest_Rate', 'Inflation_Rate'

# For demonstration, let's create a dummy DataFrame
data = {
    'Date': pd.to_datetime(pd.date_range(start='2023-01-01', periods=100)),
    'Stock_Price_Change': np.random.randn(100) * 0.02,
    'Content_Sentiment': np.random.rand(100) * 2 - 1,
    'Wording_Sentiment': np.random.rand(100) * 2 - 1,
    'Realized_EPS': np.random.randn(100) + 2,
    'Expected_EPS': np.random.randn(100) + 2.1,
    'Interest_Rate': np.random.rand(100) * 0.05,
    'Inflation_Rate': np.random.rand(100) * 0.03
}
merged_df = pd.DataFrame(data)
merged_df.set_index('Date', inplace=True)


# --- 2. Feature Engineering and Preprocessing ---
# The target variable is the stock price change
target_col = 'Stock_Price_Change'

# Normalize the features
scaler = MinMaxScaler()
scaled_features = scaler.fit_transform(merged_df.drop(columns=[target_col]))
scaled_df = pd.DataFrame(scaled_features, columns=merged_df.columns.drop(target_col), index=merged_df.index)

# Combine scaled features with the target
scaled_df[target_col] = merged_df[target_col]

# --- 3. Data Structuring for LSTM ---
def df_to_X_y(df, window_size=5, target_col='Stock_Price_Change'):
  df_as_np = df.to_numpy()
  X = []
  y = []
  for i in range(len(df_as_np) - window_size):
    # The features are all columns except the target
    row = df_as_np[i:i+window_size, :-1]
    X.append(row)
    # The label is the target column of the next day
    label = df_as_np[i+window_size, -1]
    y.append(label)
  return np.array(X), np.array(y)

WINDOW_SIZE = 10
X, y = df_to_X_y(scaled_df, WINDOW_SIZE, target_col)

# Split into training, validation, and test sets
X_train, y_train = X[:70], y[:70]
X_val, y_val = X[70:85], y[70:85]
X_test, y_test = X[85:], y[85:]


# --- 4. Model Architecture and Training ---
model = Sequential()
# The input shape is (window_size, number_of_features)
model.add(InputLayer((X_train.shape[1], X_train.shape[2])))
model.add(LSTM(128, return_sequences=True))
model.add(LSTM(64))
model.add(Dense(32, 'relu'))
model.add(Dense(1, 'linear')) # Predicting a single continuous value

model.summary()

model.compile(loss=MeanSquaredError(), optimizer=Adam(learning_rate=0.001), metrics=['mean_absolute_error'])

model.fit(X_train, y_train, validation_data=(X_val, y_val), epochs=50)

# --- 5. Evaluation ---
test_predictions = model.predict(X_test).flatten()

results = pd.DataFrame(data={'Test Predictions': test_predictions, 'Actuals': y_test})
print(results)

# You can then plot the results as shown in your original code
import matplotlib.pyplot as plt
plt.plot(results['Test Predictions'])
plt.plot(results['Actuals'])
plt.show()
