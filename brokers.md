# Broker Documentation

This document provides comprehensive documentation for all brokers available in our library.

## Table of Contents
1. [Overview](#overview)
2. [Broker Factory](#broker-factory)
3. [Kraken Broker](#kraken-broker)
4. [Saxo Broker](#saxo-broker)
5. [Interactive Brokers](#interactive-brokers)
6. [Binance Broker](#binance-broker)
7. [Mock Broker](#mock-broker)
8. [Common Operations](#common-operations)
9. [Best Practices](#best-practices)

## Overview

Our broker system provides a unified interface for interacting with different cryptocurrency exchanges. Each broker implements the same interface, making it easy to switch between different exchanges or use multiple exchanges simultaneously.

## Broker Factory

The `BrokerFactory` is used to create broker instances. It handles the configuration and initialization of different broker types.

### Usage
```python
from src.broker_interface.broker_factory import BrokerFactory

# Create broker configuration
broker_config = {
    'config': {
        'api_key': 'your_api_key',
        'api_secret': 'your_api_secret',
        'testnet': True,  # Use testnet/paper trading
        'paper_trading': True,  # Enable paper trading
        'base_url': 'https://api.kraken.com',
        'ws_url': 'wss://ws.kraken.com'
    }
}

# Create broker instance
broker = BrokerFactory.create_broker('kraken', broker_config)
```

## Kraken Broker

The Kraken broker provides access to the Kraken cryptocurrency exchange.

### Configuration
```python
kraken_config = {
    'config': {
        'api_key': 'your_api_key',
        'api_secret': 'your_api_secret',
        'testnet': True,  # Use testnet
        'paper_trading': True,  # Enable paper trading
        'base_url': 'https://api.kraken.com',
        'ws_url': 'wss://ws.kraken.com'
    }
}
```

### Connection
```python
# Connect to Kraken
await broker.connect()

# Check connection status
is_connected = broker.is_connected()
```

### Market Data
```python
# Subscribe to market data
async def on_market_data(event):
    print(f"Received market data: {event.data}")

await broker.subscribe_to_market_data("XBT/USD", on_market_data)

# Get current price
price = await broker.get_current_price("XBT/USD")

# Get order book
order_book = await broker.get_order_book("XBT/USD")
```

### Trading
```python
# Place a market order
order = await broker.place_market_order(
    symbol="XBT/USD",
    side="buy",
    quantity=0.1
)

# Place a limit order
order = await broker.place_limit_order(
    symbol="XBT/USD",
    side="sell",
    quantity=0.1,
    price=50000.0
)

# Cancel an order
await broker.cancel_order(order_id="order_123")

# Get order status
order_status = await broker.get_order_status(order_id="order_123")
```

### Account Management
```python
# Get account balance
balance = await broker.get_balance()

# Get positions
positions = await broker.get_positions()

# Get account summary
summary = await broker.get_account_summary()
```

## Saxo Broker

The Saxo broker provides access to Saxo Bank's trading platform, supporting stocks, forex, commodities, and more.

### Configuration
```python
saxo_config = {
    'config': {
        'api_key': 'your_api_key',
        'api_secret': 'your_api_secret',
        'environment': 'demo',  # 'demo' or 'live'
        'base_url': 'https://gateway.saxobank.com/sim/openapi',
        'ws_url': 'wss://streaming.saxobank.com/sim/openapi/streamingws/connect'
    }
}
```

### Connection
```python
# Connect to Saxo
await broker.connect()

# Check connection status
is_connected = broker.is_connected()
```

### Market Data
```python
# Subscribe to market data
async def on_market_data(event):
    print(f"Received market data: {event.data}")

await broker.subscribe_to_market_data("EURUSD", on_market_data)

# Get current price
price = await broker.get_current_price("EURUSD")

# Get order book
order_book = await broker.get_order_book("EURUSD")
```

### Trading
```python
# Place a market order
order = await broker.place_market_order(
    symbol="EURUSD",
    side="buy",
    quantity=1000,
    asset_type="forex"
)

# Place a limit order
order = await broker.place_limit_order(
    symbol="EURUSD",
    side="sell",
    quantity=1000,
    price=1.1000,
    asset_type="forex"
)

# Cancel an order
await broker.cancel_order(order_id="order_123")
```

### Account Management
```python
# Get account balance
balance = await broker.get_balance()

# Get positions
positions = await broker.get_positions()

# Get account summary
summary = await broker.get_account_summary()
```

## Interactive Brokers

The Interactive Brokers (IB) broker provides access to IB's trading platform, supporting stocks, options, futures, and more.

### Configuration
```python
ib_config = {
    'config': {
        'host': '127.0.0.1',
        'port': 7497,  # 7496 for TWS, 7497 for IB Gateway
        'client_id': 1,
        'paper_trading': True,
        'timeout': 20
    }
}
```

### Connection
```python
# Connect to Interactive Brokers
await broker.connect()

# Check connection status
is_connected = broker.is_connected()
```

### Market Data
```python
# Subscribe to market data
async def on_market_data(event):
    print(f"Received market data: {event.data}")

await broker.subscribe_to_market_data("AAPL", on_market_data)

# Get current price
price = await broker.get_current_price("AAPL")

# Get order book
order_book = await broker.get_order_book("AAPL")
```

### Trading
```python
# Place a market order
order = await broker.place_market_order(
    symbol="AAPL",
    side="buy",
    quantity=100,
    asset_type="stock"
)

# Place a limit order
order = await broker.place_limit_order(
    symbol="AAPL",
    side="sell",
    quantity=100,
    price=150.0,
    asset_type="stock"
)

# Place an option order
option_order = await broker.place_option_order(
    symbol="AAPL",
    side="buy",
    quantity=1,
    strike=150.0,
    expiration="2024-12-20",
    option_type="call"
)

# Cancel an order
await broker.cancel_order(order_id="order_123")
```

### Account Management
```python
# Get account balance
balance = await broker.get_balance()

# Get positions
positions = await broker.get_positions()

# Get account summary
summary = await broker.get_account_summary()
```

## Binance Broker

The Binance broker provides access to the Binance cryptocurrency exchange.

### Configuration
```python
binance_config = {
    'config': {
        'api_key': 'your_api_key',
        'api_secret': 'your_api_secret',
        'testnet': True,  # Use testnet
        'paper_trading': True,  # Enable paper trading
        'base_url': 'https://api.binance.com',
        'ws_url': 'wss://stream.binance.com:9443/ws'
    }
}
```

### Connection
```python
# Connect to Binance
await broker.connect()

# Check connection status
is_connected = broker.is_connected()
```

### Market Data
```python
# Subscribe to market data
async def on_market_data(event):
    print(f"Received market data: {event.data}")

await broker.subscribe_to_market_data("BTCUSDT", on_market_data)

# Get current price
price = await broker.get_current_price("BTCUSDT")

# Get order book
order_book = await broker.get_order_book("BTCUSDT")

# Get klines/candlestick data
klines = await broker.get_klines(
    symbol="BTCUSDT",
    interval="1h",
    limit=100
)
```

### Trading
```python
# Place a market order
order = await broker.place_market_order(
    symbol="BTCUSDT",
    side="buy",
    quantity=0.1
)

# Place a limit order
order = await broker.place_limit_order(
    symbol="BTCUSDT",
    side="sell",
    quantity=0.1,
    price=50000.0
)

# Place a stop-limit order
order = await broker.place_stop_limit_order(
    symbol="BTCUSDT",
    side="sell",
    quantity=0.1,
    price=45000.0,
    stop_price=46000.0
)

# Cancel an order
await broker.cancel_order(order_id="order_123")

# Cancel all open orders
await broker.cancel_all_orders(symbol="BTCUSDT")
```

### Account Management
```python
# Get account balance
balance = await broker.get_balance()

# Get positions
positions = await broker.get_positions()

# Get account summary
summary = await broker.get_account_summary()

# Get trading fees
fees = await broker.get_trading_fees()
```

### Special Features

1. **Futures Trading**
```python
# Switch to futures trading
await broker.use_futures()

# Place a futures order
futures_order = await broker.place_futures_order(
    symbol="BTCUSDT",
    side="buy",
    quantity=0.1,
    leverage=10
)
```

2. **Margin Trading**
```python
# Enable margin trading
await broker.enable_margin()

# Place a margin order
margin_order = await broker.place_margin_order(
    symbol="BTCUSDT",
    side="buy",
    quantity=0.1,
    is_isolated=True
)
```

3. **WebSocket Streams**
```python
# Subscribe to multiple streams
await broker.subscribe_to_streams([
    "btcusdt@trade",
    "btcusdt@depth",
    "btcusdt@kline_1m"
], on_market_data)
```

## Mock Broker

The Mock Broker is used for testing and paper trading. It simulates exchange behavior without making actual trades.

### Configuration
```python
mock_config = {
    'config': {
        'initial_balance': 100000.0,  # Initial paper trading balance
        'paper_trading': True
    }
}
```

### Usage
```python
# Create mock broker
broker = BrokerFactory.create_broker('mock', mock_config)

# Connect (no actual connection needed)
await broker.connect()

# Use the same interface as real brokers
await broker.place_market_order(...)
await broker.get_balance()
```

## Common Operations

### Market Data Subscription
```python
async def on_market_data(event):
    """Handle market data events"""
    if event.event_type == MarketDataEventType.TICK:
        print(f"Price: {event.data['price']}")
    elif event.event_type == MarketDataEventType.TRADE:
        print(f"Trade: {event.data}")

# Subscribe to market data
await broker.subscribe_to_market_data("XBT/USD", on_market_data)
```

### Order Management
```python
# Place different types of orders
market_order = await broker.place_market_order(
    symbol="XBT/USD",
    side="buy",
    quantity=0.1
)

limit_order = await broker.place_limit_order(
    symbol="XBT/USD",
    side="sell",
    quantity=0.1,
    price=50000.0
)

stop_order = await broker.place_stop_order(
    symbol="XBT/USD",
    side="sell",
    quantity=0.1,
    stop_price=45000.0
)

# Modify an order
modified_order = await broker.modify_order(
    order_id="order_123",
    new_quantity=0.2,
    new_price=51000.0
)

# Cancel an order
await broker.cancel_order(order_id="order_123")
```

### Position Management
```python
# Get current positions
positions = await broker.get_positions()

# Close a position
await broker.close_position(symbol="XBT/USD")

# Get position details
position = await broker.get_position(symbol="XBT/USD")
```

## Best Practices

1. **Error Handling**
   ```python
   try:
       await broker.place_market_order(...)
   except BrokerError as e:
       logger.error(f"Order placement failed: {e}")
   ```

2. **Connection Management**
   ```python
   # Always check connection before operations
   if not broker.is_connected():
       await broker.connect()
   
   # Handle disconnections
   async def on_disconnect():
       logger.warning("Broker disconnected, attempting to reconnect...")
       await broker.connect()
   ```

3. **Rate Limiting**
   ```python
   # Implement rate limiting for API calls
   async def place_order_with_retry(*args, **kwargs):
       max_retries = 3
       for attempt in range(max_retries):
           try:
               return await broker.place_market_order(*args, **kwargs)
           except RateLimitError:
               await asyncio.sleep(1 * (attempt + 1))
   ```

4. **Order Validation**
   ```python
   # Validate orders before placement
   def validate_order(symbol, side, quantity, price=None):
       if quantity <= 0:
           raise ValueError("Quantity must be positive")
       if price is not None and price <= 0:
           raise ValueError("Price must be positive")
       # Add more validation as needed
   ```

5. **Paper Trading**
   ```python
   # Always test strategies in paper trading first
   broker_config = {
       'config': {
           'paper_trading': True,
           'initial_balance': 100000.0
       }
   }
   ```

6. **Logging and Monitoring**
   ```python
   # Log all important operations
   logger.info(f"Placing order: {order_details}")
   logger.info(f"Order placed: {order_id}")
   logger.info(f"Order filled: {fill_details}")
   ```

## Example Strategy Integration

Here's an example of how to integrate a broker with a trading strategy:

```python
class TradingStrategy:
    def __init__(self, broker_config):
        self.broker = BrokerFactory.create_broker('kraken', broker_config)
        
    async def initialize(self):
        await self.broker.connect()
        await self.broker.subscribe_to_market_data("XBT/USD", self.on_market_data)
        
    async def on_market_data(self, event):
        if event.event_type == MarketDataEventType.TICK:
            # Implement your trading logic here
            if self.should_buy(event.data['price']):
                await self.place_buy_order()
                
    async def place_buy_order(self):
        try:
            order = await self.broker.place_market_order(
                symbol="XBT/USD",
                side="buy",
                quantity=0.1
            )
            logger.info(f"Buy order placed: {order}")
        except BrokerError as e:
            logger.error(f"Failed to place buy order: {e}")
            
    def should_buy(self, price):
        # Implement your buy signal logic
        pass
```

## Error Handling

Common broker errors and how to handle them:

1. **Connection Errors**
   ```python
   try:
       await broker.connect()
   except ConnectionError as e:
       logger.error(f"Failed to connect: {e}")
       # Implement reconnection logic
   ```

2. **Order Errors**
   ```python
   try:
       await broker.place_market_order(...)
   except InsufficientFundsError:
       logger.error("Insufficient funds for order")
   except InvalidOrderError:
       logger.error("Invalid order parameters")
   except RateLimitError:
       logger.error("Rate limit exceeded")
   ```

3. **Market Data Errors**
   ```python
   try:
       await broker.subscribe_to_market_data(...)
   except SubscriptionError as e:
       logger.error(f"Failed to subscribe: {e}")
   ```

## Performance Considerations

1. **WebSocket Usage**
   - Use WebSocket connections for real-time data
   - Implement heartbeat mechanisms
   - Handle reconnection automatically

2. **API Rate Limits**
   - Implement rate limiting
   - Use batch operations when possible
   - Cache frequently accessed data

3. **Memory Management**
   - Clear old market data
   - Limit order history
   - Implement proper cleanup

4. **Error Recovery**
   - Implement automatic reconnection
   - Retry failed operations
   - Maintain state consistency 