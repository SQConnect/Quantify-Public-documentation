# Candle Manual

## Table of Contents
1. [Introduction](#introduction)
2. [Candle Basics](#candle-basics)
3. [Event System](#event-system)
4. [Integrated Candle Management](#integrated-candle-management)
5. [Candle Consolidation](#candle-consolidation)
6. [Technical Analysis](#technical-analysis)
7. [Best Practices](#best-practices)
8. [Examples](#examples)

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

The framework provides integrated candle management through the `CandleDataManager` class, which is automatically available in all strategies through the `BaseStrategy` class.

### Accessing Candle Data in Strategies

```python
class YourStrategy(BaseStrategy):
    async def on_new_candle(self, symbol: str, timeframe: str, candle_data: CandleData) -> None:
        # Get latest candle
        latest = self.get_latest_candle(symbol, timeframe)
        
        # Get previous candle
        previous = self.get_previous_candle(symbol, timeframe)
        
        # Get last 100 candles
        candles = self.get_candles(symbol, timeframe, lookback=100)
        
        # Get as pandas DataFrame
        df = self.get_dataframe(symbol, timeframe, lookback=100)
```

### Candle Data Structure

```python
@dataclass
class CandleData:
    timestamp: datetime
    open: float
    high: float
    low: float
    close: float
    volume: float
    timeframe: str
    symbol: str
    vwap: float
    trades: int
    additional_data: dict
```

## Candle Consolidation

The framework provides a flexible `CandleConsolidator` class for combining candles into different timeframes. This is useful for:
- Converting higher frequency data to lower frequency (e.g., 1-minute to 5-minute candles)
- Creating custom timeframes not directly available from the data source
- Aggregating historical data for analysis

### Using the CandleConsolidator

```python
from src.data.candle_consolidator import CandleConsolidator, TimeFrame

# Create a 5-minute consolidator from 1-minute candles
consolidator = CandleConsolidator(
    target_timeframe=5,  # 5-minute candles
    base_timeframe=TimeFrame.MINUTE,  # Input is 1-minute candles
    alignment="left"  # Align to start of period
)

# Add individual candles
consolidated = consolidator.add_candle(candle)
if consolidated:
    # Period is complete, process the consolidated candle
    process_candle(consolidated)

# Add multiple candles at once
consolidated_candles = consolidator.add_candles_batch(candles)

# Force consolidate any pending candles
final_candles = consolidator.force_consolidate_pending()

# Get consolidated candles for a symbol
candles = consolidator.get_consolidated_candles("BTC/USD")

# Convert to pandas DataFrame
df = consolidator.to_pandas("BTC/USD")
```

### Supported Timeframes

The consolidator supports various timeframe units:
- Seconds (S)
- Minutes (M)
- Hours (H)
- Days (D)
- Weeks (W)

You can specify timeframes in two ways:
1. Integer value with base timeframe:
```python
consolidator = CandleConsolidator(5, TimeFrame.MINUTE)  # 5-minute candles
```

2. String format:
```python
consolidator = CandleConsolidator("5M")  # 5-minute candles
consolidator = CandleConsolidator("1H")  # 1-hour candles
consolidator = CandleConsolidator("1D")  # 1-day candles
```

### Helper Functions

The module provides convenient helper functions for common consolidations:

```python
from src.data.candle_consolidator import (
    create_5min_consolidator,
    create_1hour_consolidator,
    create_daily_consolidator
)

# Create common consolidators
five_min = create_5min_consolidator()
one_hour = create_1hour_consolidator()
daily = create_daily_consolidator()
```

## Technical Analysis

### Using Technical Indicators

The framework provides a comprehensive set of technical indicators that work seamlessly with our candle system. All indicators are designed to work directly with our `CandleData` structure.

#### Available Indicators

1. **Relative Strength Index (RSI)**
```python
from src.indicators.candle_indicators import CandleRSI

# Initialize RSI
rsi = CandleRSI(period=14)

# Update with new candle
rsi.update(candle_data)

# Get current value
rsi_value = rsi.get_value()
```

2. **Moving Average Convergence Divergence (MACD)**
```python
from src.indicators.candle_indicators import CandleMACD

# Initialize MACD
macd = CandleMACD(fast_period=12, slow_period=26, signal_period=9)

# Update with new candle
macd.update(candle_data)

# Get current values
macd_line = macd.get_macd()
signal_line = macd.get_signal()
histogram = macd.get_histogram()
```

## Best Practices

1. **Memory Management**
   - Clear history when no longer needed: `consolidator.clear_history()`
   - Clear specific symbol: `consolidator.clear_history("BTC/USD")`
   - Monitor memory usage through heartbeats

2. **Alignment**
   - "left": Align to start of period (default)
   - "right": Align to end of period
   - "center": Align to middle of period

3. **Batch Processing**
   - Use `add_candles_batch()` for better performance when processing multiple candles
   - Sort candles by timestamp before adding

4. **Error Handling**
   - Check for None returns from `add_candle()`
   - Handle potential ValueError for invalid OHLC data
   - Implement proper error handling in event handlers
   - Monitor system health through heartbeats

5. **Performance Considerations**
   - The system maintains memory limits for ticks and candles
   - Regular cleanup of old data
   - Efficient event processing
   - Optimized candle generation
   - Regular health checks and monitoring

6. **Monitoring and Maintenance**
   - Monitor system health through heartbeats
   - Check for data consistency
   - Monitor memory usage
   - Track tick and candle counts
   - Monitor active symbols and timeframes

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