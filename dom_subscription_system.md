# DOM Subscription and Normalization System

## Overview

The DOM (Depth of Market) Subscription and Normalization System provides a unified interface for subscribing to order book data across all brokers and normalizing it into a standardized format for order flow analysis. This system enables strategies to analyze order flow consistently across different asset classes (futures, equities, crypto) using the same logic.

## Key Features

- **Unified Interface**: Same subscription method across all brokers
- **Standardized Format**: Consistent orderbook structure regardless of broker
- **Real-time Processing**: Automatic normalization as DOM data arrives
- **Synthetic Orders**: Large aggregated orders split into realistic individual orders
- **Cross-Asset Compatibility**: Same analysis logic for futures, equities, crypto

## Architecture

```
Broker APIs → DOM Subscription → Normalization → Standardized Orderbook → Strategy Analysis
     ↓              ↓                ↓                    ↓                    ↓
  Saxo/Kraken   subscribe_to_    normalize_dom_to_    DOMEvent with      Order Flow
  Binance/IB    dom_data()       orderbook()         synthetic orders    Analysis
```

## Implementation

### 1. Base Broker Interface

All brokers implement the following abstract methods:

```python
@abstractmethod
async def subscribe_to_dom_data(self, symbol: str, callback: Optional[Callable] = None, **kwargs) -> bool:
    """Subscribe to DOM data for a symbol."""

@abstractmethod
async def unsubscribe_from_dom_data(self, symbol: str) -> bool:
    """Unsubscribe from DOM data for a symbol."""

def normalize_dom_to_orderbook(self, dom_data: Dict[str, List[Dict[str, Any]]], symbol: str, timestamp: datetime) -> List[Dict[str, Any]]:
    """Normalize DOM data into standardized orderbook format with synthetic individual orders."""
```

### 2. Normalization Logic

The core normalization algorithm splits large aggregated orders into smaller, realistic individual orders:

```python
# Process bid side (DOM aggregated → individual "virtual" orders)
for price_level in dom_data['bids']:
    price = price_level['price']
    total_size = price_level['size']
    
    # Split large aggregated orders into smaller chunks
    chunk_size = min(total_size, 50)  # Max 50 per "order"
    remaining = total_size
    
    while remaining > 0:
        size = min(chunk_size, remaining)
        orderbook_rows.append({
            'OrderID': f"DOM_{order_id_counter}",
            'Side': 'BUY',
            'Price': price,
            'Size': size,
            'Timestamp': timestamp,
            'Type': 'LMT',
            'Source': 'DOM_SYNTHETIC'
        })
        order_id_counter += 1
        remaining -= size
```

### 3. DOM Event Structure

```python
@dataclass
class DOMEvent:
    """Depth of Market event with normalized orderbook data."""
    event_type: str
    symbol: str
    timestamp: datetime
    dom_data: Dict[str, List[Dict[str, Any]]]  # Raw DOM data
    normalized_orderbook: List[Dict[str, Any]]  # Normalized orderbook rows
    data: Dict[str, Any] = field(default_factory=dict)
```

## Broker Implementations

### Saxo Broker
- Uses existing market data subscription
- Extracts bid/ask from tick data
- Publishes normalized DOM events on tick updates

### Kraken Broker
- Uses existing book (depth) subscription
- Real-time normalization of order book data
- Publishes normalized DOM events on book updates

### Binance Broker
- Uses existing depth subscription
- Real-time normalization of order book data
- Publishes normalized DOM events on depth updates

### IB Broker
- Uses existing depth subscription
- Real-time normalization of DOM data
- Publishes normalized DOM events on depth updates

### Mock Broker
- Simulates realistic order book data
- Real-time normalization of simulated data
- Publishes normalized DOM events for testing

## Usage Examples

### Basic Subscription

```python
# Subscribe to DOM data
success = await broker.subscribe_to_dom_data(
    symbol='BTC/USD',
    callback=my_dom_handler
)

# Handle DOM events
async def my_dom_handler(event: DOMEvent):
    orderbook = event.normalized_orderbook
    
    # Analyze order flow
    buy_volume = sum(order['Size'] for order in orderbook if order['Side'] == 'BUY')
    sell_volume = sum(order['Size'] for order in orderbook if order['Side'] == 'SELL')
    
    imbalance = buy_volume / sell_volume if sell_volume > 0 else float('inf')
    print(f"Order book imbalance: {imbalance:.2f}")
```

### Strategy Integration

```python
class OrderFlowStrategy(BaseStrategy):
    async def initialize(self) -> bool:
        await super().initialize()
        
        # Subscribe to DOM data for each symbol
        for symbol in self.symbols:
            await self.broker.subscribe_to_dom_data(
                symbol=symbol,
                callback=self.on_dom_event
            )
        
        return True
    
    async def on_dom_event(self, event: DOMEvent):
        """Handle normalized DOM events."""
        # Same analysis logic works for any broker!
        await self.analyze_order_flow(event.normalized_orderbook)
```

### Order Flow Analysis

```python
async def analyze_order_flow(self, orderbook: List[Dict[str, Any]]):
    """Analyze order flow using standardized orderbook format."""
    
    # Calculate metrics
    buy_orders = [order for order in orderbook if order['Side'] == 'BUY']
    sell_orders = [order for order in orderbook if order['Side'] == 'SELL']
    
    total_buy_volume = sum(order['Size'] for order in buy_orders)
    total_sell_volume = sum(order['Size'] for order in sell_orders)
    
    # Detect large orders
    large_orders = [order for order in orderbook if order['Size'] > 100]
    
    # Calculate imbalance
    imbalance = total_buy_volume / total_sell_volume if total_sell_volume > 0 else float('inf')
    
    # Generate signals
    if imbalance > 2.0:
        await self.place_buy_order(event.symbol, ...)
    elif imbalance < 0.5:
        await self.place_sell_order(event.symbol, ...)
```

## Benefits

### 1. Consistency
- Same order flow analysis logic across all brokers
- Standardized orderbook format regardless of data source
- Unified event handling for different asset classes

### 2. Realism
- Synthetic individual orders from aggregated data
- Realistic order sizes and distributions
- Maintains market microstructure characteristics

### 3. Flexibility
- Works with any broker that implements the interface
- Easy to add new brokers
- Configurable normalization parameters

### 4. Performance
- Real-time processing
- Efficient event publishing
- Minimal overhead

## Testing

Use the provided test script to verify DOM subscription functionality:

```bash
python examples/test_dom_subscription.py
```

This will test DOM subscription across different brokers and symbols, showing the normalized orderbook data.

## Configuration

### Normalization Parameters

```python
# In base_broker.py
chunk_size = min(total_size, 50)  # Max 50 per "order"
```

### Event Publishing

```python
# Publish to event manager
await self.event_manager.publish('dom', dom_event)

# Call custom callback
if callback:
    await callback(dom_event)
```

## Future Enhancements

1. **Advanced Normalization**: More sophisticated order size distribution algorithms
2. **Historical Analysis**: Store and analyze normalized orderbook history
3. **Machine Learning**: Use normalized data for ML-based order flow prediction
4. **Cross-Asset Correlation**: Analyze order flow patterns across different asset classes
5. **Real-time Alerts**: Configure alerts based on order flow anomalies

## Conclusion

The DOM Subscription and Normalization System provides a powerful foundation for order flow analysis across all brokers and asset classes. By standardizing the orderbook format and providing real-time normalization, it enables strategies to use the same analysis logic regardless of the underlying data source.

This system is particularly valuable for:
- **Order Flow Trading**: Analyze market microstructure for trading signals
- **Risk Management**: Monitor order book imbalances and large orders
- **Market Analysis**: Understand liquidity patterns and market dynamics
- **Cross-Asset Strategies**: Apply the same logic across futures, equities, and crypto 

## Automatic DOM Subscription in Strategies

### Configuration Options

You can enable automatic DOM (Depth of Market) data subscription in your strategy by adding one of the following to your strategy config:

- `subscribe_dom: true` — Subscribe to DOM data for all symbols in the `symbols` list
- `dom_symbols:` — Subscribe to DOM data for a specific subset of symbols

When enabled, the framework will:
- Call `await broker.subscribe_to_dom_data(symbol, callback=self.on_dom_event)` for each symbol
- Subscribe to `'dom'` events via the event manager
- Provide a default `async def on_dom_event(self, event)` handler in your strategy (override for custom logic)

#### Example Config (YAML)

```yaml
name: OrderFlowStrategy
symbols:
  - BTCUSDT
  - ETHUSDT
subscribe_dom: true
```

or

```yaml
name: OrderFlowStrategy
symbols:
  - BTCUSDT
  - ETHUSDT
dom_symbols:
  - BTCUSDT
```

### Usage in Strategy

```python
class MyOrderFlowStrategy(BaseStrategy):
    async def on_dom_event(self, event: DOMEvent):
        # Your custom DOM event logic here
        print(f"Received DOM event for {event.symbol}")
```

## Full Event Flow for DOM Data

The event trust/chain for DOM data is as follows:

1. **Broker Receives Raw DOM Data**
    - The broker (e.g., Binance, Kraken, Saxo) receives raw order book/depth data from the exchange API.
2. **Normalization**
    - The broker normalizes the raw DOM data into a standardized orderbook format using `normalize_dom_to_orderbook`.
3. **DOMEvent Creation**
    - The broker creates a `DOMEvent` dataclass instance containing:
        - `event_type`: 'dom'
        - `symbol`: The instrument symbol
        - `timestamp`: Event timestamp
        - `dom_data`: Raw DOM data (bids/asks)
        - `normalized_orderbook`: List of synthetic individual orders
        - `data`: Additional metadata (e.g., bid/ask levels)
4. **EventManager Publishing**
    - The broker publishes the `DOMEvent` to the event manager: `await self.event_manager.publish('dom', dom_event)`
5. **Callback Invocation**
    - If a callback was provided to `subscribe_to_dom_data`, it is called with the `DOMEvent`.
6. **Strategy Event Subscription**
    - The strategy subscribes to 'dom' events via the event manager (automatically if `subscribe_dom` or `dom_symbols` is set).
7. **Strategy Handler Execution**
    - The strategy's `on_dom_event` method is called with the normalized `DOMEvent`.

### Event Trust Diagram

```
Exchange API
   ↓
Broker (raw DOM data)
   ↓
normalize_dom_to_orderbook()
   ↓
DOMEvent (standardized)
   ↓
EventManager.publish('dom', dom_event)
   ↓
Strategy.on_dom_event(event)
```

## Example: End-to-End DOM Event Flow

```python
# In your strategy config (YAML):
#
# name: OrderFlowStrategy
# symbols:
#   - BTCUSDT
# subscribe_dom: true

class OrderFlowStrategy(BaseStrategy):
    async def on_dom_event(self, event: DOMEvent):
        print(f"Received DOM event for {event.symbol} at {event.timestamp}")
        # Analyze event.normalized_orderbook here
```

## DOMEvent Structure

The `DOMEvent` dataclass provides a standardized structure for all normalized DOM (Depth of Market) events. This is the object your strategy receives in `on_dom_event`.

```python
@dataclass
class DOMEvent:
    """Depth of Market event with normalized orderbook data."""
    event_type: str  # Always 'dom'
    symbol: str      # The instrument symbol (e.g., 'BTCUSDT', 'AAPL:xnas')
    timestamp: datetime  # UTC timestamp when the DOM snapshot was taken
    dom_data: Dict[str, List[Dict[str, Any]]]  # Raw DOM data as received from the broker
    normalized_orderbook: List[Dict[str, Any]]  # List of synthetic individual orders (see below)
    data: Dict[str, Any] = field(default_factory=dict)  # Additional metadata (e.g., bid/ask levels, normalization info)
```

### Field Descriptions
- **event_type** (`str`): Always `'dom'` for DOM events.
- **symbol** (`str`): The symbol/instrument for which the DOM data applies.
- **timestamp** (`datetime`): UTC timestamp of the DOM snapshot.
- **dom_data** (`Dict[str, List[Dict[str, Any]]]`):
    - The raw order book data as received from the broker/exchange.
    - Example:
      ```python
      {
        'bids': [{'price': 100.0, 'size': 5.0}, ...],
        'asks': [{'price': 101.0, 'size': 3.0}, ...]
      }
      ```
- **normalized_orderbook** (`List[Dict[str, Any]]`):
    - A list of synthetic individual orders, normalized for cross-broker consistency.
    - Each order is a dict with at least:
      - `OrderID` (`str`): Unique synthetic order ID
      - `Side` (`str`): 'BUY' or 'SELL'
      - `Price` (`float` or `Decimal`): Order price
      - `Size` (`float` or `Decimal`): Order size/quantity
      - `Timestamp` (`datetime`): Timestamp of the orderbook snapshot
      - `Type` (`str`): Order type, e.g., 'LMT' (limit)
      - `Source` (`str`): 'DOM_SYNTHETIC' (indicates synthetic normalization)
- **data** (`Dict[str, Any]`):
    - Additional metadata, such as:
      - `bid_levels` (`int`): Number of bid price levels
      - `ask_levels` (`int`): Number of ask price levels
      - `raw_dom` (`dict`): Copy of the original raw DOM data
      - Any other normalization or broker-specific info

This structure ensures that all strategies receive DOM data in a consistent, broker-agnostic format, ready for order flow analysis, signal generation, or visualization. 