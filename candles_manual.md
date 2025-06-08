# Candles Manual

## Overview
The candle system is responsible for managing and processing market data into OHLCV (Open, High, Low, Close, Volume) candles across multiple timeframes. It provides real-time candle generation, event notifications, and data management capabilities.

## Event Types
The candle system supports several types of events that can be subscribed to:

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

## Heartbeat System
The heartbeat system provides regular status updates about the candle manager's state. Heartbeats are emitted:

1. **Per Symbol/Timeframe**: Individual heartbeats for each active symbol and timeframe combination
2. **Global**: A system-wide heartbeat with overall status

### Heartbeat Data Structure
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

### Subscribing to Heartbeats
```python
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

### Heartbeat Handler Example
```python
async def handle_heartbeat(event: CandleEvent):
    status = event.additional_data
    print(f"System Status at {status['timestamp']}:")
    print(f"Active Symbols: {status['active_symbols']}")
    print(f"Active Timeframes: {status['active_timeframes']}")
    print(f"Total Ticks: {status['tick_count']}")
    print(f"Total Candles: {status['candle_count']}")
```

## Event Subscription
To subscribe to candle events:

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
```

## Event Handler Example
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
```

## Best Practices
1. Always handle exceptions in event handlers
2. Use appropriate event types for your use case
3. Subscribe to heartbeats for system monitoring
4. Implement proper error handling for event processing
5. Monitor system health through heartbeat events

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