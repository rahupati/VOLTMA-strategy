# Import necessary libraries
from kiteconnect import KiteConnect
import pandas as pd
from datetime import datetime, time

# Initialize Kite Connect with your API Key and Secret
api_key = "your_api_key"
api_secret = "your_api_secret"
kite = KiteConnect(api_key=api_key)

# Request token you obtained earlier (replace with fresh request token)
request_token = "your_request_token"

# Generate session and get access token
data = kite.generate_session(request_token, api_secret=api_secret)
access_token = data["access_token"]
print(f"Access Token: {access_token}")

# Set the access token
kite.set_access_token(access_token)

# Total capital for trading (including 5x leverage)
capital = 20000 * 5  # Applying 5x leverage to your base capital of ₹20,000

# Define Risk-Reward Ratio (RRR)
initial_rrr = 1 / 3  # Initial RRR 1:3
adjusted_rrr = 1 / 2  # Adjusted RRR 1:2 after a stop-loss or target hit

# Function to calculate stop loss and target price based on entry price
def calculate_sl_tp(entry_price, rrr):
    stop_loss = entry_price * 0.99  # Assuming a 1% risk
    target_price = entry_price + (entry_price - stop_loss) * (1 / rrr)  # Target based on RRR
    return stop_loss, target_price

# Function to calculate ATR (Average True Range)
def calculate_atr(data):
    data['ATR'] = data['high'] - data['low']
    return data['ATR'].mean()

# Dynamic stock selection based on real-time liquidity and volatility, limited to 5 stocks
def dynamic_stock_selection():
    instruments = kite.instruments("NSE")
    selected_stocks = []
    
    for stock in instruments:
        try:
            if stock['instrument_type'] != 'EQ':  # Only include stocks (equities)
                continue
            
            ltp = kite.ltp(f"NSE:{stock['tradingsymbol']}")['NSE:' + stock['tradingsymbol']]['last_price']
            current_date = datetime.now().strftime("%Y-%m-%d")
            from_date = f"{current_date} 09:15:00"
            to_date = f"{current_date} 15:30:00"
            
            data = kite.historical_data(stock['instrument_token'], 
                                        from_date=from_date, 
                                        to_date=to_date, 
                                        interval="5minute")
            atr = calculate_atr(pd.DataFrame(data))
            
            # Select stocks with high ATR and LTP > 100
            if atr > 2 and ltp > 100:
                selected_stocks.append((stock['tradingsymbol'], atr))
            
            # Stop after selecting 5 stocks (limit the number)
            if len(selected_stocks) >= 5:
                break
        except Exception as e:
            print(f"Error processing stock {stock['tradingsymbol']}: {e}")
    
    return selected_stocks

# Capital distribution based on inverse ATR (higher ATR gets lower capital)
def distribute_capital(selected_stocks, total_capital):
    # Calculate inverse of ATR for allocation
    total_inverse_atr = sum(1 / stock[1] for stock in selected_stocks)
    stock_allocations = []
    
    for stock, atr in selected_stocks:
        capital_allocated = (1 / atr) / total_inverse_atr * total_capital
        stock_allocations.append((stock, capital_allocated))
    
    return stock_allocations

# Calculate quantity based on allocated capital for each stock
def calculate_quantity(stock, allocated_capital):
    try:
        ltp = kite.ltp(f"NSE:{stock}")['NSE:' + stock]['last_price']
        quantity = allocated_capital // ltp  # Floor division for integer quantity
        return int(quantity)
    except Exception as e:
        print(f"Error calculating quantity for {stock}: {e}")
        return 1  # Default to 1 if error occurs

# Apply the Low-Risk MA Intraday Strategy on each selected stock with SL and TP
def apply_strategy(stock, allocated_capital, rrr):
    try:
        current_date = datetime.now().strftime("%Y-%m-%d")
        from_date = f"{current_date} 09:15:00"
        to_date = f"{current_date} 15:30:00"
        
        # Fetch historical data for the stock
        data = pd.DataFrame(kite.historical_data(kite.ltp(f"NSE:{stock}")['NSE:' + stock]['instrument_token'],
                                                 from_date=from_date, 
                                                 to_date=to_date, 
                                                 interval="5minute"))
        data['MA20'] = data['close'].rolling(window=20).mean()
        last_candle = data.iloc[-1]

        entry_price = last_candle['close']
        stop_loss, target_price = calculate_sl_tp(entry_price, rrr)

        # Determine buy/sell based on price and 20-period moving average
        if last_candle['close'] > last_candle['MA20']:  # Buy signal
            quantity = calculate_quantity(stock, allocated_capital)
            place_order_with_sl_tp(stock, "BUY", entry_price, stop_loss, target_price, quantity)
        elif last_candle['close'] < last_candle['MA20']:  # Sell signal (Short)
            quantity = calculate_quantity(stock, allocated_capital)
            place_order_with_sl_tp(stock, "SELL", entry_price, stop_loss, target_price, quantity)
    except Exception as e:
        print(f"Error applying strategy for stock {stock}: {e}")

# Place an order (Buy/Sell) with Stop Loss and Target Price
def place_order_with_sl_tp(stock, direction, entry_price, stop_loss, target_price, quantity):
    try:
        # Place the primary order (Market order)
        kite.place_order(tradingsymbol=stock,
                         exchange="NSE",
                         transaction_type=direction,
                         quantity=quantity,
                         order_type="MARKET",
                         product="MIS")  # Intraday order
        print(f"{direction} order placed for {stock} with quantity {quantity}")

        # Place Stop Loss and Target Price orders after the main order is executed
        kite.place_order(tradingsymbol=stock,
                         exchange="NSE",
                         transaction_type="SELL" if direction == "BUY" else "BUY",
                         quantity=quantity,
                         price=target_price,  # Target price order
                         order_type="LIMIT",
                         product="MIS")  # Intraday target
        print(f"Target price set for {stock} at {target_price}")
        
        kite.place_order(tradingsymbol=stock,
                         exchange="NSE",
                         transaction_type="SELL" if direction == "BUY" else "BUY",
                         quantity=quantity,
                         trigger_price=stop_loss,  # Stop loss order
                         order_type="SL-M",
                         product="MIS")  # Intraday stop loss
        print(f"Stop loss set for {stock} at {stop_loss}")
    except Exception as e:
        print(f"Error placing SL/TP orders for {stock}: {e}")

# Managing Time-Based Rules
def manage_trades():
    stop_time = time(14, 30)  # No new orders after 2:30 PM
    square_off_time = time(15, 10)  # Square off all positions at 3:10 PM
    current_time = datetime.now().time()

    if current_time >= stop_time and current_time < square_off_time:
        print("No new orders after 2:30 PM")
        return

    if current_time >= square_off_time:
        print("Squaring off all positions")
        kite.exit_orders()

# Select stocks dynamically based on real-time data
selected_stocks = dynamic_stock_selection()
print(f"Selected Stocks: {selected_stocks}")

# Distribute capital among selected stocks
stock_allocations = distribute_capital(selected_stocks, capital)
print(f"Stock Allocations: {stock_allocations}")

# Apply the strategy to selected stocks with initial RRR (1:3)
for stock, allocated_capital in stock_allocations:
    apply_strategy(stock, allocated_capital, initial_rrr)

# Manage the time-based rules
manage_trades()
