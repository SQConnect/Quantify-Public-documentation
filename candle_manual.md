# Candle Manual

## Table of Contents
1. [Introduction](#introduction)
2. [Candle Basics](#candle-basics)
3. [Data Ingestion](#data-ingestion)
4. [Event System](#event-system)
5. [Integrated Candle Management](#integrated-candle-management)
6. [Candle Consolidation and Resampling](#candle-consolidation-and-resampling)
7. [Strategy Warm-Up Period](#strategy-warm-up-period)
8. [Technical Analysis](#technical-analysis)
9. [Best Practices](#best-practices)
10. [Examples](#examples)

## Introduction

This manual provides a comprehensive guide to working with candles in our trading system. Candles are fundamental building blocks for technical analysis and trading strategies, representing price action over specific time periods.

## Candle Basics

A candle represents price action over a specific timeframe and contains:
- Open price
- High price
- Low price
- Close price
- Volume
- Timestamp
- VWAP (Volume Weighted Average Price)
- Number of trades
- Additional metadata

### Standard Timeframes
```python
valid_timeframes = ['1m', '5m', '15m', '30m', '1h', '4h', '1d']
```

## Data Ingestion

The framework has moved away from a generic ZeroMQ-based subscription model towards a more robust, broker-specific data handling approach. Data is now primarily ingested through the active broker's WebSocket or REST API endpoints. This change ensures higher fidelity data and better integration with the broker's ecosystem.

When a strategy is initialized, it specifies the required symbols and timeframes. The `BaseStrategy` and the underlying broker implementation handle the subscription and data fetching automatically. Historical data is pre-fetched to warm up the strategy, and live data is streamed subsequently.

### Configuration

Data requirements are defined directly in the strategy's configuration file. The `StrategyFactory` uses this configuration to set up the necessary data streams via the designated broker.

```yaml
# Example from a strategy's config.yaml
strategy_name: MyAwesomeStrategy
strategy_class: MyAwesomeStrategy
base_timeframe: '1m'
resample_timeframes: ['5m', '15m']
symbols:
  - 'XBT/USD'
  - 'ETH/USD'
warm_up_period_candles:
  '1m': 200
  '5m': 60
```

## Event System

The candle system supports several types of events that can be subscribed to:

### Event Types

1. **NEW_CANDLE**: Triggered when a new candle is created
   - Contains the initial candle data
   - Includes timestamp, OHLCV data, and additional metadata

2. **CANDLE_CLOSED**: Triggered when a candle period is complete
   - Contains the final candle data
   - Includes all OHLCV data and statistics

3. **PRICE_ALERT**: Triggered when price crosses specified levels
   - Contains the price level and condition that triggered the alert
   - Includes the current candle data

4. **VOLUME_ALERT**: Triggered when volume exceeds thresholds
   - Contains the volume threshold that was exceeded
   - Includes the current candle data

5. **PATTERN_DETECTED**: Triggered when technical patterns are identified
   - Contains the pattern name and details
   - Includes the relevant candle data

6. **HEARTBEAT**: Regular system status updates
   - Contains comprehensive system status information
   - Includes:
     - Current timestamp
     - Tick and candle counts
     - Active symbols and timeframes
     - Last tick times for each symbol
     - System health metrics

### Heartbeat System

The heartbeat system provides regular status updates about the candle manager's state. Heartbeats are emitted:

1. **Per Symbol/Timeframe**: Individual heartbeats for each active symbol and timeframe combination
2. **Global**: A system-wide heartbeat with overall status

#### Heartbeat Data Structure
```python
{
    'timestamp': datetime,  # Current system time
    'tick_count': int,      # Total number of ticks processed
    'candle_count': int,    # Total number of candles generated
    'active_symbols': List[str],  # List of symbols with active data
    'active_timeframes': List[str],  # List of registered timeframes
    'last_tick_times': {  # Last tick time for each symbol
        'symbol': datetime,
        ...
    }
}
```

### Event Subscription Examples

```python
# Subscribe to new candles
candle_manager.subscribe_to_events(
    CandleEventType.NEW_CANDLE,
    symbol='BTC/USD',
    timeframe='1m',
    handler=your_handler
)

# Subscribe to candle closures
candle_manager.subscribe_to_events(
    CandleEventType.CANDLE_CLOSED,
    symbol='BTC/USD',
    timeframe='1m',
    handler=your_handler
)

# Subscribe to symbol-specific heartbeats
candle_manager.subscribe_to_events(
    CandleEventType.HEARTBEAT,
    symbol='BTC/USD',
    timeframe='1m',
    handler=your_handler
)

# Subscribe to global heartbeats
candle_manager.subscribe_to_events(
    CandleEventType.HEARTBEAT,
    symbol='*',
    timeframe='*',
    handler=your_handler
)
```

### Event Handler Examples

```python
async def handle_candle_event(event: CandleEvent):
    if event.event_type == CandleEventType.NEW_CANDLE:
        print(f"New candle for {event.symbol} on {event.timeframe}")
        print(f"Open: {event.candle_data['open']}")
        print(f"High: {event.candle_data['high']}")
        print(f"Low: {event.candle_data['low']}")
        print(f"Close: {event.candle_data['close']}")
        print(f"Volume: {event.candle_data['volume']}")
    elif event.event_type == CandleEventType.CANDLE_CLOSED:
        print(f"Candle closed for {event.symbol} on {event.timeframe}")
        # Process closed candle data

async def handle_heartbeat(event: CandleEvent):
    status = event.additional_data
    print(f"System Status at {status['timestamp']}:")
    print(f"Active Symbols: {status['active_symbols']}")
    print(f"Active Timeframes: {status['active_timeframes']}")
    print(f"Total Ticks: {status['tick_count']}")
    print(f"Total Candles: {status['candle_count']}")
```

## Integrated Candle Management

The framework provides integrated candle management through a singleton `EventManager` and a `CandleManager` instance that is injected into each strategy. This ensures that all parts of the application, from data providers to strategies, work from a unified event queue and consistent candle data.

### Accessing the Candle Manager

Within a strategy that inherits from `BaseStrategy`, the `CandleManager` is available as `self.candle_manager`. It is automatically initialized and configured by the `StrategyFactory` during the strategy's creation.

```python
# No manual instantiation is needed.
# self.candle_manager is available in your strategy methods.

class YourStrategy(BaseStrategy):
    async def initialize(self):
        # The candle manager is ready to be used.
        self.candle_manager.get_candles(...)
```

### Accessing Candle Data in Strategies

The `BaseStrategy` provides convenient methods to access candle data managed by the `CandleManager`.

```python
class YourStrategy(BaseStrategy):
    async def on_live_candle(self, candle: Candle) -> None:
        # Get latest candle for a specific symbol and timeframe
        latest = self.candle_manager.get_latest_candle(candle.symbol, candle.timeframe)
        
        # Get previous candle
        previous = self.candle_manager.get_previous_candle(candle.symbol, candle.timeframe)
        
        # Get last 100 candles
        candles = self.candle_manager.get_candles(candle.symbol, candle.timeframe, lookback=100)
        
        # Get as pandas DataFrame
        df = self.candle_manager.get_candles_as_df(candle.symbol, candle.timeframe, lookback=100)
```

## Candle Consolidation and Resampling

A key feature of the `CandleManager` is its ability to resample a base timeframe (e.g., 1-minute candles) into multiple larger timeframes (e.g., 5-minute, 15-minute). This is done efficiently in real-time as new data arrives.

The resampling process is now handled through a robust three-pass system designed to correctly process historical data snapshots and then transition smoothly to live data.

### Three-Pass System for Historical Data

When a strategy starts, it needs a "warm-up" period with historical data. The `CandleManager` uses a special process to ensure this data is loaded and resampled correctly before live trading begins.

1.  **Pass 1: Load Base Candles**: The `CandleManager` is first populated with a historical snapshot of the *base timeframe* candles (e.g., 1-minute data).
2.  **Pass 2: Resample and Backfill**: It then iterates over this historical data to build all the required larger timeframe candles (e.g., creating 5-minute candles from the 1-minute data). During this pass, candles for the larger timeframes are created and backfilled.
3.  **Pass 3: Flush and Publish**: Finally, the manager performs a "flush," iterating through all completed historical candles (both base and resampled) and publishes them sequentially via `CANDLE_CLOSED` events. This allows the strategy to process them as if they were arriving in real-time, ensuring indicators are calculated correctly.

This system guarantees that by the time the first live candle arrives, the strategy's state is fully synchronized with a complete and consistent set of historical data across all required timeframes.

## Strategy Warm-Up Period

To ensure that strategies have sufficient data to make informed trading decisions from the moment they go live, the framework implements a mandatory warm-up period. This period is handled by the `BaseStrategy` and is configured in the strategy's YAML file.

During the warm-up phase, the `BaseStrategy` subscribes to historical candle data from the `CandleManager`. It provides two distinct abstract methods that developers must implement to handle the flow of historical and live data separately.

### `on_historical_candle(candle: Candle)`

This method is called for each candle processed during the initial warm-up phase. This includes both the base timeframe candles and any resampled, consolidated candles. Use this method to populate indicators and data structures with historical context without triggering trading logic.

### `on_live_candle(candle: Candle)`

This method is called only *after* the warm-up period is complete for all required timeframes. Live market data, whether from a real-time feed or a backtest, will trigger this method. All trading logic, such as signal generation and order placement, should be placed here.

The `BaseStrategy` tracks the number of historical candles received for each timeframe against the counts specified in the `warm_up_period_candles` configuration. It will only begin calling `on_live_candle` once every timeframe has received its required number of historical candles.

```python
class MyElliottWaveStrategy(BaseStrategy):

    async def initialize(self):
        # Initialization logic, e.g., setting up indicators
        self.ema_50 = EMA(period=50)

    async def on_historical_candle(self, candle: Candle) -> None:
        """
        Called for each historical candle during the warm-up period.
        """
        self.logger.info(f"Received historical {candle.timeframe} candle for {candle.symbol}: {candle.close_price}")
        # Update indicators with historical data
        if candle.timeframe == '5m':
            self.ema_50.add_value(candle.close_price)

    async def on_live_candle(self, candle: Candle) -> None:
        """
        Called for each new live candle after the warm-up is complete.
        """
        self.logger.info(f"Received live {candle.timeframe} candle for {candle.symbol}: {candle.close_price}")
        # Update indicators with live data
        if candle.timeframe == '5m':
            self.ema_50.add_value(candle.close_price)
            
            # Implement trading logic
            if candle.close_price > self.ema_50.get():
                self.logger.info("Price crossed above 50-period EMA. Considering a long position.")
                # Place buy order, etc.

```

## Technical Analysis

The framework provides a rich set of tools for performing technical analysis on candle data.
These tools are available as standalone utilities or can be integrated directly into your strategies.

## Best Practices

1. **Error Handling**: Always implement proper error handling for ZMQ connection issues
2. **Cleanup**: Ensure the subscriber is properly stopped in your strategy's cleanup method
3. **Configuration**: Use configuration files to manage ZMQ addresses and topics
4. **Monitoring**: Monitor the subscriber's status through the CandleManager's heartbeat events
5. **Logging**: Use the built-in logger (`self.logger`) to record important events, decisions, and errors within your strategy. This is invaluable for debugging and performance analysis.
6. **Configuration**: Keep strategy parameters in configuration files rather than hardcoding them. This makes it easier to tune, backtest, and manage your strategies.
7. **State Management**: Be mindful of strategy state. Use the provided methods for warm-up (`on_historical_candle`) and live trading (`on_live_candle`) to ensure a clean separation of concerns.

## Error Handling

The candle system includes comprehensive error handling:
1. Validates all incoming tick data
2. Checks for data consistency
3. Handles gaps in data
4. Recovers from errors automatically
5. Provides detailed error logging

## Performance Considerations

1. The system maintains memory limits for ticks and candles
2. Regular cleanup of old data
3. Efficient event processing
4. Optimized candle generation
5. Regular health checks and monitoring

## Monitoring and Maintenance

1. Monitor system health through heartbeats
2. Check for data consistency
3. Monitor memory usage
4. Track tick and candle counts
5. Monitor active symbols and timeframes

## Examples

### Basic Strategy Example
```python
class SimpleStrategy(BaseStrategy):
    def __init__(self, config_path: str):
        super().__init__(config_path)
        
        # Register timeframes
        self.candle_manager.register_timeframe('14m', 14, 'm')
        
        # Subscribe to events
        self.candle_manager.subscribe_to_events(
            CandleEventType.NEW_CANDLE,
            'BTC/USD',
            '14m',
            self.on_candle_event
        )
    
    async def on_candle_event(self, event: CandleEvent):
        if event.event_type == CandleEventType.NEW_CANDLE:
            # Update indicators
            await self._update_indicators(event.candle_data)
            
            # Generate signals
            signals = await self.generate_signals()
            for signal in signals:
                await self.process_signal(signal)
        
        elif event.event_type == CandleEventType.CANDLE_CLOSED:
            # Check for trading signals
            previous_candle = self.get_previous_candle(event.symbol, event.timeframe)
            if previous_candle and event.candle_data.close > previous_candle.high:
                await self.enter_position(event.symbol, 'long', event.candle_data.close)
```

### Advanced Strategy Example
```python
class AdvancedStrategy(BaseStrategy):
    def __init__(self, config_path: str):
        super().__init__(config_path)
        
        # Initialize indicators
        self.rsi = CandleRSI(period=14)
        self.macd = CandleMACD()
        
        # Subscribe to multiple events
        self.candle_manager.subscribe_to_events(
            CandleEventType.NEW_CANDLE,
            'BTC/USD',
            '1h',
            self.on_candle_event
        )
        self.candle_manager.subscribe_to_events(
            CandleEventType.PRICE_ALERT,
            'BTC/USD',
            '1h',
            self.on_price_alert
        )
    
    async def on_candle_event(self, event: CandleEvent):
        # Update indicators
        self.rsi.update(event.candle_data)
        self.macd.update(event.candle_data)
        
        # Generate signals based on multiple indicators
        if (self.rsi.get_value() < 30 and 
            self.macd.get_histogram() > 0):
            await self.enter_position(event.symbol, 'long', event.candle_data.close)
    
    async def on_price_alert(self, event: CandleEvent):
        price_level = event.additional_data['price_level']
        condition = event.additional_data['condition']
        
        if event.symbol in self.positions:
            position = self.positions[event.symbol]
            if (position['side'] == 'long' and condition == 'below' and
                event.candle_data.close < float(price_level)):
                await self.exit_position(event.symbol, event.candle_data.close)
```

This manual provides a comprehensive guide to working with candles in our trading system. The new integrated candle management system makes it easier to work with candle data while maintaining good performance and memory management. 