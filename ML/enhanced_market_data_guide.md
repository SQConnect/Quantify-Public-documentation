# Enhanced Market Data Guide

## Overview

The strategy framework now supports comprehensive real-time market data including tick, quote, and depth events across all supported brokers. This guide explains how to use these enhanced capabilities.

## Available Market Data Types

### 1. Tick Events
Real-time price and volume updates for individual trades.

```python
@dataclass
class TickEvent(MarketDataEvent):
    price: Decimal          # Current trade price
    volume: Decimal         # Trade volume
    bid: Optional[Decimal]  # Best bid price
    ask: Optional[Decimal]  # Best ask price
```

**Use Cases:**
- Real-time price monitoring
- Volume analysis
- High-frequency trading
- Performance tracking

### 2. Quote Events
Bid/ask updates with size information.

```python
@dataclass
class QuoteEvent(MarketDataEvent):
    bid: Decimal      # Best bid price
    ask: Decimal      # Best ask price
    bid_size: Decimal # Size at best bid
    ask_size: Decimal # Size at best ask
```

**Use Cases:**
- Spread analysis
- Liquidity monitoring
- Order book imbalance detection
- Market making strategies

### 3. Depth Events
Full order book depth with multiple price levels.

```python
@dataclass
class DepthEvent(MarketDataEvent):
    bids: List[Dict[str, Decimal]]  # List of {price: Decimal, size: Decimal}
    asks: List[Dict[str, Decimal]]  # List of {price: Decimal, size: Decimal}
```

**Use Cases:**
- Market depth analysis
- Large order impact assessment
- Order flow analysis
- Institutional activity detection

### 4. OHLC Events
Candle data (already available).

```python
@dataclass
class OHLCEvent(MarketDataEvent):
    timeframe: str
    open: Decimal
    high: Decimal
    low: Decimal
    close: Decimal
    volume: Decimal
```

## Broker Support

| Broker | OHLC | TICK | QUOTE | DEPTH | Notes |
|--------|------|------|-------|-------|-------|
| Kraken | ✅ | ✅ | ✅ | ✅ | Full support via WebSocket v2 |
| Saxo | ✅ | ✅ | ✅ | ❌ | Missing depth data |
| IB | ✅ | ✅ | ✅ | ✅ | Full support |
| Binance | ✅ | ❌ | ❌ | ❌ | OHLC only |
| Mock | ✅ | ✅ | ✅ | ✅ | For testing |

## Quick Start

### 1. Basic Market Data Subscription

```python
from src.broker_interface.kraken_broker import KrakenBroker
from src.core.events import EventManager

# Create event manager and broker
event_manager = EventManager()
await event_manager.start()

broker = KrakenBroker(event_manager, config)

# Connect to broker
await broker.connect()

# Subscribe to all market data types
success = await broker.subscribe_to_market_data("BTC/USD", timeframe="1m")
```

### 2. Event Handling

```python
from src.core.events import TickEvent, QuoteEvent, DepthEvent

async def on_tick(event: TickEvent):
    print(f"Tick: {event.symbol} @ {event.price}")

async def on_quote(event: QuoteEvent):
    spread = event.ask - event.bid
    print(f"Quote: {event.symbol} - Spread: {spread}")

async def on_depth(event: DepthEvent):
    total_bids = sum(bid['size'] for bid in event.bids)
    total_asks = sum(ask['size'] for ask in event.asks)
    print(f"Depth: {event.symbol} - Bid size: {total_bids}, Ask size: {total_asks}")

# Subscribe to events
await event_manager.subscribe('tick', on_tick)
await event_manager.subscribe('quote', on_quote)
await event_manager.subscribe('depth', on_depth)
```

### 3. Strategy Integration

```python
from src.strategy_framework.base_strategy import BaseStrategy

class MyStrategy(BaseStrategy):
    async def initialize(self) -> bool:
        # Initialize base strategy
        if not await super().initialize():
            return False
        
        # Subscribe to market data events
        await self.event_manager.subscribe('tick', self.on_tick)
        await self.event_manager.subscribe('quote', self.on_quote)
        await self.event_manager.subscribe('depth', self.on_depth)
        
        return True
    
    async def on_tick(self, event: TickEvent):
        # Handle tick events
        pass
    
    async def on_quote(self, event: QuoteEvent):
        # Handle quote events
        pass
    
    async def on_depth(self, event: DepthEvent):
        # Handle depth events
        pass
```

## Advanced Usage

### 1. Order Flow Analysis

```python
class OrderFlowStrategy(BaseStrategy):
    def __init__(self, config):
        super().__init__(config)
        self.large_order_threshold = 1000
        self.order_events = []
    
    async def on_depth(self, event: DepthEvent):
        # Detect large orders
        large_bids = [bid for bid in event.bids if bid['size'] > self.large_order_threshold]
        large_asks = [ask for ask in event.asks if ask['size'] > self.large_order_threshold]
        
        if large_bids or large_asks:
            # Large orders detected - potential institutional activity
            await self._analyze_order_flow(event.symbol, large_bids, large_asks)
    
    async def _analyze_order_flow(self, symbol: str, large_bids: list, large_asks: list):
        # Implement order flow analysis logic
        pass
```

### 2. Market Microstructure Analysis

```python
class MicrostructureStrategy(BaseStrategy):
    async def on_quote(self, event: QuoteEvent):
        # Calculate spread
        spread = event.ask - event.bid
        spread_pct = spread / event.bid
        
        # Analyze bid/ask size imbalance
        size_imbalance = event.bid_size / event.ask_size if event.ask_size > 0 else 1.0
        
        if size_imbalance > 2.0:
            # Bids dominate - potential upward pressure
            await self._consider_buy_signal(event.symbol, event.ask)
        elif size_imbalance < 0.5:
            # Asks dominate - potential downward pressure
            await self._consider_sell_signal(event.symbol, event.bid)
```

### 3. Multi-Timeframe Analysis

```python
class MultiTimeframeStrategy(BaseStrategy):
    def __init__(self, config):
        super().__init__(config)
        self.tick_data = {}
        self.ohlc_data = {}
    
    async def on_tick(self, event: TickEvent):
        # Store tick data for analysis
        if event.symbol not in self.tick_data:
            self.tick_data[event.symbol] = []
        
        self.tick_data[event.symbol].append({
            'timestamp': event.timestamp,
            'price': event.price,
            'volume': event.volume
        })
        
        # Keep only recent data
        if len(self.tick_data[event.symbol]) > 1000:
            self.tick_data[event.symbol] = self.tick_data[event.symbol][-1000:]
    
    async def on_ohlc(self, event: OHLCEvent):
        # Combine tick and OHLC data for analysis
        await self._analyze_multi_timeframe(event.symbol)
```

## Configuration

### Broker Configuration

```yaml
broker:
  kraken:
    api_key: "${KRAKEN_API_KEY}"
    api_secret: "${KRAKEN_API_SECRET}"
    ws_url: "wss://ws.kraken.com/v2"
    market_data:
      default_channels: ["tick", "quote", "depth", "ohlc"]
      default_timeframe: "1m"
      max_depth_levels: 10
      subscription_timeout: 10.0
```

### Strategy Configuration

```yaml
strategy:
  my_strategy:
    symbols: ["BTC/USD", "ETH/USD"]
    timeframes: ["1m", "5m"]
    market_data_channels: ["tick", "quote", "depth"]
    parameters:
      large_order_threshold: 1000
      spread_threshold: 0.001
      imbalance_threshold: 2.0
```

## Best Practices

### 1. Event Handling
- Keep event handlers lightweight and fast
- Use async/await properly
- Handle exceptions gracefully
- Avoid blocking operations in event handlers

### 2. Data Management
- Implement proper data cleanup
- Use appropriate data structures for your use case
- Consider memory usage with high-frequency data
- Implement data validation

### 3. Performance
- Monitor event processing latency
- Use efficient data structures
- Implement proper error handling
- Consider using background tasks for heavy processing

### 4. Testing
- Test with real market data
- Implement unit tests for event handlers
- Use mock data for development
- Test error scenarios

## Troubleshooting

### Common Issues

1. **No events received**
   - Check broker connection
   - Verify subscription success
   - Check symbol format
   - Review broker logs

2. **High latency**
   - Monitor event processing time
   - Check for blocking operations
   - Review system resources
   - Optimize event handlers

3. **Memory issues**
   - Implement data cleanup
   - Limit data storage
   - Use efficient data structures
   - Monitor memory usage

### Debug Mode

Enable debug logging to troubleshoot issues:

```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

## Examples

See the following example files:
- `examples/market_data_strategy_example.py` - Complete strategy example
- `tests/test_kraken_market_data.py` - Test script for verification

## Next Steps

1. **Test the enhanced Kraken broker** with real data
2. **Implement strategies** using the new market data types
3. **Enhance other brokers** to support all data types
4. **Add advanced features** like data aggregation and filtering

## Support

For issues or questions:
1. Check the troubleshooting section
2. Review broker-specific documentation
3. Test with the provided examples
4. Check the framework logs for errors 