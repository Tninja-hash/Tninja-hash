import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import yfinance as yf

# Use the gold ETF symbol
symbol = "GLD"

# Download intraday data with 60-minute intervals
data = yf.download(symbol, period="60d", interval="60m")

# Calculate moving averages with the specified windows
short_window = 9
long_window = 18

data['Short_MA'] = data['Close'].rolling(window=short_window, min_periods=1).mean()
data['Long_MA'] = data['Close'].rolling(window=long_window, min_periods=1).mean()

# Generate signals
data['Signal'] = 0
data['Signal'][short_window:] = np.where(data['Short_MA'][short_window:] > data['Long_MA'][short_window:], 1, 0)
data['Position'] = data['Signal'].diff()

# Parameters for position sizing and risk management
initial_capital = 100.0
risk_per_trade = 0.01  # Risk 1% of capital on each trade
stop_loss_pct = 0.02   # Stop-loss at 2% below the purchase price
portfolio = pd.DataFrame(index=data.index)
portfolio['cash'] = initial_capital
portfolio['holdings'] = 0.0
portfolio['total'] = initial_capital
portfolio['position_size'] = 0  # Track the number of shares
portfolio['stop_loss'] = np.nan

# Backtesting with position sizing and risk management
for i in range(1, len(data)):
    # Check if a stop-loss is hit
    if portfolio['position_size'].iloc[i-1] > 0 and data['Close'].iloc[i] < portfolio['stop_loss'].iloc[i-1]:
        # Stop-loss triggered, sell position
        portfolio['cash'].iloc[i] = portfolio['cash'].iloc[i-1] + (portfolio['position_size'].iloc[i-1] * data['Close'].iloc[i])
        portfolio['holdings'].iloc[i] = 0
        portfolio['position_size'].iloc[i] = 0
        portfolio['stop_loss'].iloc[i] = np.nan
    elif data['Position'].iloc[i] == 1:  # Buy signal
        risk_amount = portfolio['total'].iloc[i-1] * risk_per_trade
        position_size = risk_amount // (data['Close'].iloc[i] * stop_loss_pct)
        portfolio['position_size'].iloc[i] = position_size
        portfolio['cash'].iloc[i] = portfolio['cash'].iloc[i-1] - (position_size * data['Close'].iloc[i])
        portfolio['holdings'].iloc[i] = position_size * data['Close'].iloc[i]
        portfolio['stop_loss'].iloc[i] = data['Close'].iloc[i] * (1 - stop_loss_pct)
    elif data['Position'].iloc[i] == -1:  # Sell signal
        portfolio['cash'].iloc[i] = portfolio['cash'].iloc[i-1] + (portfolio['position_size'].iloc[i-1] * data['Close'].iloc[i])
        portfolio['holdings'].iloc[i] = 0
        portfolio['position_size'].iloc[i] = 0
        portfolio['stop_loss'].iloc[i] = np.nan
    else:
        portfolio['cash'].iloc[i] = portfolio['cash'].iloc[i-1]
        portfolio['holdings'].iloc[i] = portfolio['position_size'].iloc[i-1] * data['Close'].iloc[i]
        portfolio['stop_loss'].iloc[i] = portfolio['stop_loss'].iloc[i-1]

    portfolio['total'].iloc[i] = portfolio['cash'].iloc[i] + portfolio['holdings'].iloc[i]

# Print the final portfolio value and performance
print(f"Final portfolio value: ${portfolio['total'].iloc[-1]:.2f}")
print(f"Total return: {((portfolio['total'].iloc[-1] / initial_capital) - 1) * 100:.2f}%")

# Plot the portfolio value over time
plt.figure(figsize=(12, 6))
plt.plot(portfolio['total'], label='Portfolio Value')
plt.title(f'{symbol} Portfolio Value Over Time with Risk Management')
plt.xlabel('Date')
plt.ylabel('Total Portfolio Value')
plt.legend()
plt.show()
