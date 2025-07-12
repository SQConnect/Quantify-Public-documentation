# Broker Documentation

This document provides comprehensive documentation for all brokers available in the Quantify trading system.

## Table of Contents
1. [Overview](#overview)
2. [Broker Factory](#broker-factory)
3. [Saxo Broker](#saxo-broker)
4. [Kraken Broker](#kraken-broker)
5. [Interactive Brokers](#interactive-brokers)
6. [Binance Broker](#binance-broker)
7. [Backtest Broker](#backtest-broker)
8. [Mock Broker](#mock-broker)
9. [Unified Market Data Interface](#unified-market-data-interface)
10. [Common Operations](#common-operations)
11. [Best Practices](#best-practices)

## Overview

Our broker system provides a unified interface for interacting with different financial exchanges and brokers. Each broker implements the same interface, making it easy to switch between different brokers or use multiple brokers simultaneously. All brokers support:

- **Equity Trading**: Market and limit orders for stocks/ETFs
- **Options Trading**: Single leg and multi-leg options strategies  
- **Market Data**: Real-time quotes, historical candles, order book
- **Portfolio Management**: Positions, account summary, holdings
- **Event-Driven Architecture**: Real-time market data and order updates

## Broker Factory

The `BrokerFactory` creates broker instances with proper configuration and initialization.

### Usage
```python
from src.broker_interface.broker_factory import BrokerFactory
from src.core.config.config_manager import ConfigManager

# Load configuration
config_manager = ConfigManager()
broker_config = config_manager.get_broker_config('saxo')

# Create broker instance
broker = BrokerFactory.create_broker('saxo', broker_config)

# Connect and start trading
await broker.connect()
```

## Saxo Broker

The Saxo broker provides comprehensive access to Saxo Bank's trading platform, supporting stocks, options, forex, commodities, and more. **This is our most advanced broker implementation with production-ready features.**

### 🚀 **Key Features**
- ✅ **Auto Token Refresh**: Automatic background token refresh (checks every minute, refreshes 5 minutes before expiry)
- ✅ **WebSocket Auto-Reconnection**: Smart reconnection with exponential backoff and subscription restoration
- ✅ **Comprehensive Options Support**: 25+ strategy types, multi-leg orders, real-time Greeks
- ✅ **Market Data Handling**: Supports both live and indicative prices (market closed scenarios)
- ✅ **Production Error Handling**: Robust error handling, validation, and logging
- ✅ **Portfolio Management**: Complete position tracking, account summaries, holdings

### Configuration
```python
# Environment variables (.env file)
SAXO_API_KEY=your_api_key
SAXO_API_SECRET=your_api_secret
SAXO_REFRESH_TOKEN=your_refresh_token  # Auto-saved after first login

# Broker configuration
saxo_config = {
    'config': {
        'environment': 'demo',  # 'demo' or 'live'
        'base_url': 'https://gateway.saxobank.com/sim/openapi',
        'ws_url': 'wss://streaming.saxobank.com/sim/openapi/streamingws/connect',
        'token_refresh_buffer_minutes': 5,  # Refresh 5 minutes before expiry
        'heartbeat_interval': 30,  # WebSocket heartbeat
        'max_reconnect_attempts': 10
    }
}
```

### Connection & Authentication
```python
# Connect with automatic token management
await broker.connect()

# Connection automatically handles:
# - Token refresh (background monitoring)
# - WebSocket reconnection
# - Subscription restoration
# - Account key caching

# Check connection status
is_connected = broker.is_connected
```

### Equity Trading
```python
from decimal import Decimal

# Place market order
market_order = await broker.place_order(
    symbol="NVDA:xnas",
    order_type="Market",
    side="Buy",
    quantity=Decimal("100")
)

# Place limit order with time in force
limit_order = await broker.place_order(
    symbol="NVDA:xnas",
    order_type="Limit",
    side="Sell", 
    quantity=Decimal("100"),
    price=Decimal("850.00"),
    time_in_force="GTC"  # GTC, DAY, FOK, IOC
)

# Cancel order
cancelled = await broker.cancel_order(order_id="12345")
```

### Options Trading

#### Single Leg Options
```python
from src.options.option_leg import OptionLeg
from src.options.option_types import OptionType

# Get options chain
chain = await broker.get_options_chain("NVDA:xnas", datetime(2024, 12, 20))

# Find specific option
nvda_call = chain.get_leg_by_strike(850.0, OptionType.CALL)

# Place option order
option_order = await broker.place_option_order(
    option_leg=nvda_call,
    side="Buy",
    quantity=1,
    order_type="Limit",
    price=25.50
)
```

#### Multi-Leg Strategy Orders
```python
# Create put spread legs
legs = [
    {"leg": put_sell_leg, "side": "Sell", "quantity": 1},
    {"leg": put_buy_leg, "side": "Buy", "quantity": 1}
]

# Place multi-leg order at net credit
spread_order = await broker.place_multi_leg_order(
    legs=legs,
    order_type="Limit",
    price=2.50  # Net credit
)
```

#### Strategy Factory (25+ Strategies)
```python
# Access comprehensive strategy factory
saxo_options = broker.saxo_options

# Create vertical spread
strategy = await saxo_options.create_option_strategy(
    strategy_type="BullCallSpread",
    underlying="NVDA:xnas",
    expiry=datetime(2024, 12, 20),
    long_strike=800.0,
    short_strike=850.0,
    quantity=1
)

# Available strategies include:
# Vertical Spreads: BullCallSpread, BearPutSpread, etc.
# Volatility: LongStraddle, ShortStrangle, IronCondor
# Calendar: CalendarSpread, DiagonalSpread
# Exotic: ButterflySpread, CondorSpread, JadeeLizard
# And 15+ more...
```

### Market Data
```python
# Get current market data (handles indicative prices)
market_data = await broker.get_market_data("NVDA:xnas")
# Returns: {
#   'price': 845.20,
#   'bid': 845.10,
#   'ask': 845.30,
#   'market_state': 'Open',  # or 'Closed'
#   'is_indicative': False,  # True if market closed
#   'timestamp': datetime(...)
# }

# Subscribe to real-time market data
async def on_tick(payload):
    print(f"Real-time price: {payload}")

await broker.subscribe_to_market_data("NVDA:xnas", on_tick)

# Get historical OHLC data with proper candle integration
candles = await broker.get_ohlc_data(
    symbol="NVDA:xnas",
    timeframe="1h",
    count=100
)
```

### Portfolio Management
```python
# Get comprehensive account summary
account = await broker.get_account_summary()
# Returns: {
#   'account_id': 'ABCD1234',
#   'currency': 'USD', 
#   'cash_balance': 50000.0,
#   'net_liquidation_value': 75000.0,
#   'total_equity': 75000.0,
#   'buying_power': 100000.0,
#   'day_pnl': 1250.0,
#   'total_pnl': 5000.0
# }

# Get all positions with enhanced data
positions = await broker.get_positions()
# Each position includes:
# - symbol, quantity, avg_price, market_price
# - market_value, unrealized_pnl, currency
# - direction, asset_type, exchange

# Get option positions specifically
option_positions = await broker.get_option_positions()

# Get account holdings (detailed breakdown)
holdings = await broker.get_account_holdings()
```

### Advanced Features

#### Real-time Greeks Subscription
```python
from src.core.events import event_manager, GreeksEvent

# Event handler for Greeks updates
async def on_greeks_update(event: GreeksEvent):
    print(f"Greeks for {event.symbol}: Delta={event.delta}, Vega={event.vega}")

# Subscribe to event
event_manager.subscribe(GreeksEvent, on_greeks_update)

# Subscribe to specific option's Greeks
await broker.subscribe_to_greeks(option_leg)
```

#### Order Event Monitoring
```python
from src.core.events import OrderEvent

async def on_order_update(event: OrderEvent):
    order = event.order
    print(f"Order {order.order_id}: {order.status}")

event_manager.subscribe(OrderEvent, on_order_update)
await broker.subscribe_to_orders()
```

#### Connection Monitoring
```python
# Background tasks automatically handle:
# - Token refresh monitoring (every minute)
# - WebSocket heartbeat monitoring  
# - Auto-reconnection with exponential backoff
# - Subscription restoration after reconnection

# Manual connection check
if not broker.is_connected:
    await broker.connect()
```

### Error Handling
```python
from src.core.exceptions import BrokerError, OrderError, MarketDataError

try:
    order = await broker.place_order(...)
except OrderError as e:
    print(f"Order failed: {e}")
except BrokerError as e:
    print(f"Broker error: {e}")
```

## Kraken Broker

The Kraken broker provides access to the Kraken cryptocurrency exchange with both live and paper trading support.

### Configuration
```python
kraken_config = {
    'config': {
        'api_key': 'your_api_key',
        'api_secret': 'your_api_secret',
        'paper_trading': True,  # Enable paper trading
        'base_url': 'https://api.kraken.com',
        'ws_url': 'wss://ws.kraken.com'
    }
}
```

### Trading
```python
# Place orders (same interface as other brokers)
order = await broker.place_order(
    symbol="XBTUSD",
    order_type="Limit",
    side="Buy",
    quantity=Decimal("0.1"),
    price=Decimal("45000.0")
)
```

## Interactive Brokers

The Interactive Brokers (IB) broker provides access to IB's platform via the TWS API.

### Configuration  
```python
ib_config = {
    'config': {
        'host': '127.0.0.1',
        'port': 7497,  # 7496 for TWS, 7497 for IB Gateway
        'client_id': 1,
        'paper_trading': True
    }
}
```

### Usage
```python
# Same unified interface
await broker.connect()
order = await broker.place_order(
    symbol="AAPL",
    order_type="Market",
    side="Buy", 
    quantity=Decimal("100")
)
```

## Binance Broker

The Binance broker provides access to Binance cryptocurrency exchange.

### Configuration
```python
binance_config = {
    'config': {
        'api_key': 'your_api_key',
        'api_secret': 'your_api_secret',
        'testnet': True,
        'base_url': 'https://api.binance.com'
    }
}
```

### Special Features
```python
# Cryptocurrency-specific features
klines = await broker.get_klines(
    symbol="BTCUSDT", 
    interval="1h",
    limit=100
)

# Futures trading
await broker.use_futures()
futures_order = await broker.place_futures_order(...)
```

## Backtest Broker

The Backtest broker simulates trading for backtesting strategies.

### Configuration
```python
backtest_config = {
    'config': {
        'initial_balance': 100000.0,
        'commission': 0.001,  # 0.1% commission
        'slippage': 0.0001   # 0.01% slippage
    }
}
```

### Usage
```python
# Set historical data
broker.set_current_time(datetime(2024, 1, 1))
broker.load_historical_data(data)

# Same trading interface
order = await broker.place_order(...)
```

## Mock Broker

The Mock broker is used for testing without any real trading.

### Configuration
```python
mock_config = {
    'config': {
        'initial_balance': 100000.0,
        'paper_trading': True
    }
}
```

## Unified Market Data Interface

The framework provides a unified market data interface that standardizes how brokers handle real-time market data subscriptions. This interface supports multiple data channels and integrates seamlessly with the strategy framework.

### Market Data Channels

The framework supports four types of market data channels:

```python
from src.broker_interface.market_data_interface import MarketDataChannel

# Available channels
MarketDataChannel.TICK    # Individual trades/ticks
MarketDataChannel.QUOTE   # Best bid/ask quotes  
MarketDataChannel.DEPTH   # Order book depth
MarketDataChannel.OHLC    # Candlestick/OHLC data
```

### Market Data Subscription

Create market data subscriptions using the unified interface:

```python
from src.broker_interface.market_data_interface import MarketDataSubscription, MarketDataChannel

# Subscribe to multiple channels for a symbol
subscription = MarketDataSubscription(
    symbol="ETHUSDT",
    channels=[
        MarketDataChannel.OHLC,
        MarketDataChannel.TICK,
        MarketDataChannel.QUOTE,
        MarketDataChannel.DEPTH
    ],
    timeframe="1m",        # Required for OHLC channel
    depth_levels=10,       # Number of order book levels
    callback=my_callback   # Optional callback function
)

# Subscribe via broker
success = await broker.subscribe_market_data(subscription)
```

### Broker Capabilities

Each broker reports its market data capabilities:

```python
# Get broker capabilities
capabilities = await broker.get_market_data_capabilities()

print(f"Tick support: {capabilities.tick_support}")
print(f"Quote support: {capabilities.quote_support}")
print(f"Depth support: {capabilities.depth_support}")
print(f"OHLC support: {capabilities.ohlc_support}")
print(f"Max depth levels: {capabilities.max_depth_levels}")
print(f"Base timeframe: {capabilities.base_timeframe}")
```

### Broker Support Matrix

| Broker | TICK | QUOTE | DEPTH | OHLC | Base Timeframe |
|--------|------|-------|-------|------|----------------|
| Saxo | ✅ | ✅ | ❌ | ✅ | 1m |
| Kraken | ✅ | ✅ | ✅ | ✅ | 1m |
| IB | ✅ | ✅ | ✅ | ✅ | 1m |
| Binance | ✅ | ✅ | ✅ | ✅ | 1m |
| Mock | ✅ | ✅ | ✅ | ✅ | 1m |

### Event Integration

The unified interface automatically publishes events to the framework's event system:

```python
from src.core.events import event_manager, OHLCEvent, TickEvent, QuoteEvent, DepthEvent

# Subscribe to market data events
async def on_ohlc_data(event: OHLCEvent):
    print(f"OHLC: {event.symbol} - O:{event.open} H:{event.high} L:{event.low} C:{event.close}")

async def on_tick_data(event: TickEvent):
    print(f"Tick: {event.symbol} - Price:{event.price} Volume:{event.volume}")

async def on_quote_data(event: QuoteEvent):
    print(f"Quote: {event.symbol} - Bid:{event.bid} Ask:{event.ask}")

async def on_depth_data(event: DepthEvent):
    print(f"Depth: {event.symbol} - Levels:{len(event.bids)}")

# Register event handlers
await event_manager.subscribe("OHLC_DATA", on_ohlc_data)
await event_manager.subscribe("tick", on_tick_data)
await event_manager.subscribe("quote", on_quote_data)
await event_manager.subscribe("depth", on_depth_data)
```

### Strategy Integration

Strategies automatically use the unified interface through the base strategy class:

```python
class MyStrategy(BaseStrategy):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # Strategy automatically subscribes to OHLC data
        # based on configured symbols and timeframes
    
    async def on_live_candle(self, event: CandleEvent):
        # Handle OHLC data
        pass
    
    async def on_tick(self, event: TickEvent):
        # Handle tick data (if subscribed)
        pass
    
    async def on_quote(self, event: QuoteEvent):
        # Handle quote data (if subscribed)
        pass
    
    async def on_depth(self, event: DepthEvent):
        # Handle depth data (if subscribed)
        pass
```

### Advanced Usage

#### Custom Market Data Subscriptions

```python
# Subscribe to specific channels only
tick_subscription = MarketDataSubscription(
    symbol="BTCUSD",
    channels=[MarketDataChannel.TICK],
    callback=my_tick_handler
)

# Subscribe to order book with custom depth
depth_subscription = MarketDataSubscription(
    symbol="ETHUSD",
    channels=[MarketDataChannel.DEPTH],
    depth_levels=20  # 20 levels instead of default 10
)
```

#### Multiple Symbol Subscriptions

```python
symbols = ["BTCUSD", "ETHUSD", "ADAUSD"]
subscriptions = []

for symbol in symbols:
    subscription = MarketDataSubscription(
        symbol=symbol,
        channels=[MarketDataChannel.OHLC, MarketDataChannel.TICK],
        timeframe="1m"
    )
    subscriptions.append(subscription)
    await broker.subscribe_market_data(subscription)
```

#### Subscription Management

```python
# Get active subscriptions
active_subs = await broker.get_active_subscriptions()
print(f"Active subscriptions: {list(active_subs.keys())}")

# Unsubscribe from specific symbol
await broker.unsubscribe_market_data("BTCUSD")
```

## Common Operations

### Universal Broker Interface

All brokers implement the same interface defined in `BaseBroker`:

```python
# Connection
await broker.connect()
await broker.disconnect()

# Trading
order = await broker.place_order(symbol, order_type, side, quantity, price, time_in_force)
success = await broker.cancel_order(order_id)

# Market Data (Legacy Interface)
data = await broker.get_market_data(symbol)
await broker.subscribe_to_market_data(symbol, callback)

# Market Data (Unified Interface) - Recommended
subscription = MarketDataSubscription(symbol, channels, timeframe)
await broker.subscribe_market_data(subscription)

# Portfolio
positions = await broker.get_positions()
summary = await broker.get_account_summary()
holdings = await broker.get_account_holdings()

# Options (if supported)
chain = await broker.get_options_chain(underlying, expiry)
order = await broker.place_option_order(option_leg, side, quantity, order_type, price)
order = await broker.place_multi_leg_order(legs, order_type, price)
```

### Event System Integration

All brokers integrate with the event system:

```python
from src.core.events import event_manager, MarketDataEvent, OrderEvent

# Subscribe to events
event_manager.subscribe(MarketDataEvent, your_handler)
event_manager.subscribe(OrderEvent, your_order_handler)
```

### Error Handling Best Practices

```python
from src.core.exceptions import BrokerError, ConnectionError, OrderError

try:
    await broker.connect()
    order = await broker.place_order(...)
except ConnectionError:
    print("Failed to connect to broker")
except OrderError as e:
    print(f"Order rejected: {e}")
except BrokerError as e:
    print(f"General broker error: {e}")
```

## Best Practices

### 1. Configuration Management
- Store sensitive credentials in environment variables
- Use the `ConfigManager` for centralized configuration
- Never hardcode API keys in source code

### 2. Connection Management
- Always check `broker.is_connected` before trading
- Use try/finally blocks to ensure proper disconnection
- Let automatic reconnection handle temporary disconnects

### 3. Market Data Subscriptions
- Use the unified `MarketDataInterface` for new implementations
- Check broker capabilities before subscribing to channels
- Subscribe only to channels you actually need to reduce bandwidth
- Use the base timeframe (usually 1m) and let the candle manager create other timeframes
- Handle subscription failures gracefully with retry logic

### 4. Order Management
- Use `Decimal` for all price and quantity values
- Validate orders before submission
- Monitor order events for execution updates
- Implement proper position sizing

### 5. Error Handling
- Catch specific exceptions (OrderError, ConnectionError)
- Implement retry logic for transient errors  
- Log all errors with context for debugging

### 6. Performance
- Cache market data when possible
- Use async/await properly for concurrent operations
- Batch operations when supported by the broker
- Limit the number of simultaneous market data subscriptions

### 7. Testing
- Use Mock broker for unit tests
- Use Backtest broker for strategy validation
- Test with paper trading before live deployment

### 8. Options Trading
- Always validate option chains before trading
- Monitor Greeks for risk management
- Use multi-leg orders for complex strategies
- Implement assignment checking for short options

### 9. Event-Driven Architecture
- Use event handlers for real-time data processing
- Avoid blocking operations in event handlers
- Implement proper error handling in event callbacks
- Use the framework's event system for loose coupling

This unified broker system provides the foundation for all trading operations in the Quantify framework, ensuring consistent behavior across different financial markets and exchanges. 