# Event System Documentation

## Overview
The trading system uses an event-driven architecture to handle various types of events that occur during trading. This document provides a comprehensive overview of all available events and their usage.

## Event Types

### Market Data Events

#### 1. TICK Event
```python
MarketDataEventType.TICK
```
- **Purpose**: Real-time price and volume updates
- **Data Fields**:
  - `price`: Current price
  - `volume`: Trading volume
  - `bid`: Best bid price
  - `ask`: Best ask price
- **Usage**: 
  - Price monitoring
  - Volume analysis
  - Real-time trading decisions
  - Performance tracking

#### 2. QUOTE Event
```python
MarketDataEventType.QUOTE
```
- **Purpose**: Order book updates
- **Data Fields**:
  - `bid`: Best bid price
  - `ask`: Best ask price
  - `bid_size`: Size at best bid
  - `ask_size`: Size at best ask
- **Usage**:
  - Spread analysis
  - Liquidity monitoring
  - Order book imbalance detection

#### 3. DEPTH Event
```python
MarketDataEventType.DEPTH
```
- **Purpose**: Full order book depth
- **Data Fields**:
  - `bids`: List of bid prices and sizes
  - `asks`: List of ask prices and sizes
- **Usage**:
  - Market depth analysis
  - Large order impact assessment
  - Liquidity analysis

#### 4. GREEKS Event
```python
MarketDataEventType.GREEKS
```
- **Purpose**: Options Greeks updates
- **Data Fields**:
  - `delta`: Price sensitivity to underlying
  - `gamma`: Delta sensitivity to underlying
  - `theta`: Time decay
  - `vega`: Volatility sensitivity
  - `rho`: Interest rate sensitivity
- **Usage**:
  - Options trading
  - Risk management
  - Portfolio hedging

### Trading Day Events

#### 1. START_OF_DAY Event
```python
MarketDataEventType.START_OF_DAY
```
- **Purpose**: Trading day initialization
- **Trigger**: 30 minutes after market open
- **Data Fields**:
  - `time`: Event trigger time
- **Usage**:
  - Reset daily statistics
  - Initialize trading parameters
  - Clear previous day's data
  - Log trading day start

#### 2. END_OF_DAY Event
```python
MarketDataEventType.END_OF_DAY
```
- **Purpose**: Trading day conclusion
- **Trigger**: At market close
- **Data Fields**:
  - `time`: Event trigger time
- **Usage**:
  - Save daily metrics
  - Generate daily reports
  - Close positions if needed
  - Log daily summary

### Order Events

#### 1. ORDER_CREATED Event
```python
OrderEventType.CREATED
```
- **Purpose**: New order notification
- **Data Fields**:
  - `order_id`: Unique order identifier
  - `symbol`: Trading symbol
  - `side`: Buy/Sell
  - `quantity`: Order size
  - `price`: Order price
- **Usage**:
  - Order tracking
  - Position management
  - Risk monitoring

#### 2. ORDER_FILLED Event
```python
OrderEventType.FILLED
```
- **Purpose**: Order execution notification
- **Data Fields**:
  - `order_id`: Order identifier
  - `fill_price`: Execution price
  - `fill_quantity`: Executed quantity
  - `commission`: Trading fees
- **Usage**:
  - Trade execution tracking
  - P&L calculation
  - Performance monitoring

#### 3. ORDER_CANCELLED Event
```python
OrderEventType.CANCELLED
```
- **Purpose**: Order cancellation notification
- **Data Fields**:
  - `order_id`: Order identifier
  - `reason`: Cancellation reason
- **Usage**:
  - Order management
  - Error tracking
  - Strategy adjustment

### Position Events

#### 1. POSITION_OPENED Event
```python
PositionEventType.OPENED
```
- **Purpose**: New position notification
- **Data Fields**:
  - `symbol`: Trading symbol
  - `quantity`: Position size
  - `entry_price`: Average entry price
- **Usage**:
  - Position tracking
  - Risk management
  - Portfolio monitoring

#### 2. POSITION_CLOSED Event
```python
PositionEventType.CLOSED
```
- **Purpose**: Position closure notification
- **Data Fields**:
  - `symbol`: Trading symbol
  - `quantity`: Closed quantity
  - `exit_price`: Average exit price
  - `pnl`: Realized profit/loss
- **Usage**:
  - P&L tracking
  - Performance analysis
  - Trade history

## Event Handling

### Subscribing to Events
```python
# Subscribe to market data events
event_manager.subscribe(MarketDataEventType.TICK, callback_function)

# Subscribe to order events
event_manager.subscribe(OrderEventType.FILLED, callback_function)
```

### Unsubscribing from Events
```python
# Unsubscribe from events
event_manager.unsubscribe(MarketDataEventType.TICK, callback_function)
```

### Publishing Events
```python
# Publish a market data event
event = MarketDataEvent(
    event_type=MarketDataEventType.TICK,
    symbol="BTCUSDT",
    timestamp=datetime.now(),
    data={"price": 50000, "volume": 1.5}
)
await event_manager.publish(event)
```

## Best Practices

1. **Event Handling**
   - Keep event handlers lightweight
   - Use async/await for non-blocking operations
   - Handle exceptions in event callbacks

2. **Performance**
   - Subscribe only to needed events
   - Unsubscribe when no longer needed
   - Use appropriate event types for data

3. **Error Handling**
   - Implement error handling in callbacks
   - Log event processing errors
   - Implement retry mechanisms for critical events

4. **Testing**
   - Test event handlers in isolation
   - Verify event data integrity
   - Monitor event processing performance

## Example Usage

```python
class TradingStrategy:
    def __init__(self):
        # Subscribe to market data
        event_manager.subscribe(MarketDataEventType.TICK, self.on_tick)
        event_manager.subscribe(MarketDataEventType.END_OF_DAY, self.on_end_of_day)
        
    async def on_tick(self, event: MarketDataEvent):
        # Handle tick data
        price = event.data['price']
        volume = event.data['volume']
        # Process market data...
        
    async def on_end_of_day(self, event: MarketDataEvent):
        # Handle end of day
        # Save metrics
        # Generate reports
        # Close positions if needed
```

## Event Flow Diagram
```
Market Data → TICK/QUOTE/DEPTH Events → Strategy Processing
    ↓
Order Events → Position Events → Performance Tracking
    ↓
Trading Day Events → Daily Summary & Reports
```

## Troubleshooting

1. **Missing Events**
   - Verify event subscriptions
   - Check event publishing
   - Monitor event manager logs

2. **Event Processing Delays**
   - Check handler performance
   - Monitor system resources
   - Optimize event processing

3. **Data Inconsistencies**
   - Validate event data
   - Implement data checks
   - Monitor data quality

## Additional Resources

- Event Manager Implementation: `src/core/events.py`
- Event Type Definitions: `src/core/event_types.py`
- Example Strategies: `src/strategy_framework/strategy_types/` 