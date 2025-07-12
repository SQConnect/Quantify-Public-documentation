# Simple Options Wrapper: Broker Encapsulation

## Overview

The Simple Options Wrapper has been refactored to encapsulate the broker like `OptionsHelper` does, ensuring users never need to deal with broker details directly. This provides a clean, simple interface that hides framework internals.

## The Problem with Broker Exposure

### Before: Users Deal with Broker

```python
# OLD WAY - Users need to pass broker around
result = await buy_call(broker, "AAPL", 150.0, "2024-01-19", 1, 5.50)
result = await get_option_price(broker, "AAPL", 150.0, "2024-01-19", "call")

# Users see broker details everywhere
class MyStrategy:
    def __init__(self, broker):
        self.broker = broker  # Users deal with broker
    
    async def trade(self):
        # Pass broker to every function
        result = await buy_call(self.broker, "AAPL", 150.0, "2024-01-19", 1, 5.50)
```

**Problems:**
- Users need to understand broker details
- Broker parameter passed everywhere
- Inconsistent with OptionsHelper pattern
- Framework internals exposed

## The Solution: Broker Encapsulation

### After: Broker Encapsulated Like OptionsHelper

```python
# NEW WAY - Broker encapsulated, users never see it
# Initialize once
options_trader = SimpleOptionsTrader(broker)
set_simple_options_trader(options_trader)

# Use anywhere without broker parameter
result = await buy_call("AAPL", 150.0, "2024-01-19", 1, 5.50)
result = await get_option_price("AAPL", 150.0, "2024-01-19", "call")

# Clean strategy integration
class MyStrategy:
    def __init__(self, broker):
        # Initialize once - broker is encapsulated
        self.options_trader = SimpleOptionsTrader(broker)
        set_simple_options_trader(self.options_trader)
    
    async def trade(self):
        # No broker parameter needed
        result = await buy_call("AAPL", 150.0, "2024-01-19", 1, 5.50)
```

**Benefits:**
- Users never see broker details
- Clean, simple interface
- Consistent with OptionsHelper pattern
- Framework internals hidden

## Implementation Details

### 1. SimpleOptionsTrader Encapsulation

```python
class SimpleOptionsTrader:
    """Simple options trading interface that encapsulates the broker."""
    
    def __init__(self, broker: 'BaseBroker'):
        """Initialize with broker - users don't see broker details after this."""
        self.broker = broker
        self.positions: Dict[str, SimpleOptionPosition] = {}
        self.logger = get_logger(__name__, "SimpleOptionsTrader")
    
    async def place_order(self, order: SimpleOptionOrder) -> Dict[str, Any]:
        """Place order - broker details handled internally."""
        # All broker interaction happens here
        return await self._place_single_leg_order(order) or await self._place_multi_leg_order(order)
```

### 2. SimpleOptionData Encapsulation

```python
class SimpleOptionData:
    """Simple options data that encapsulates the broker."""
    
    def __init__(self, underlying: str, broker: 'BaseBroker'):
        """Initialize with broker - users don't see broker details after this."""
        self.underlying = underlying
        self.broker = broker
        self._option_chains: Dict[str, Dict] = {}
        self._current_price: Optional[float] = None
        self.logger = get_logger(__name__, "SimpleOptionData")
    
    async def get_current_price(self) -> float:
        """Get current price - broker details handled internally."""
        if self._current_price is None:
            quote = await self.broker.get_quote(self.underlying)
            self._current_price = quote.get('price', 0.0)
        return self._current_price
```

### 3. Global Instance Management

```python
# Global instances for convenience functions
_simple_options_trader: Optional[SimpleOptionsTrader] = None
_simple_options_data_instances: Dict[str, SimpleOptionData] = {}

def set_simple_options_trader(trader: SimpleOptionsTrader):
    """Set the global simple options trader instance."""
    global _simple_options_trader
    _simple_options_trader = trader

def get_simple_options_trader() -> SimpleOptionsTrader:
    """Get the global simple options trader instance."""
    if _simple_options_trader is None:
        raise RuntimeError("SimpleOptionsTrader not initialized. Call set_simple_options_trader() first.")
    return _simple_options_trader
```

### 4. Convenience Functions Without Broker

```python
# Convenience functions that don't require broker parameter
async def buy_call(underlying: str, strike: float, expiry: str, quantity: int = 1, price: Optional[float] = None):
    """Buy a call option - no broker parameter needed."""
    trader = get_simple_options_trader()  # Gets global instance
    order = SimpleOptionOrder(
        underlying=underlying,
        strategy=SimpleStrategyType.BUY_CALL,
        strike=strike,
        expiry=expiry,
        quantity=quantity,
        price=price
    )
    return await trader.place_order(order)

async def get_option_price(underlying: str, strike: float, expiry: str, option_type: str) -> Dict[str, Any]:
    """Get option price and Greeks - no broker parameter needed."""
    options_data = get_simple_options_data(underlying)  # Gets global instance
    return await options_data.get_option_price(strike, expiry, option_type)
```

## Usage Patterns

### 1. Strategy Integration

```python
class MyOptionsStrategy(BaseStrategy):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # Initialize once - broker is encapsulated
        self.options_trader = SimpleOptionsTrader(self.broker)
        self.options_data = SimpleOptionData(self.symbol, self.broker)
        
        # Set global instances for convenience functions
        set_simple_options_trader(self.options_trader)
        set_simple_options_data(self.symbol, self.options_data)
    
    async def on_signal(self, signal):
        if signal.action == "buy_call":
            # Simple order definition - no broker needed
            order = SimpleOptionOrder(
                underlying=signal.symbol,
                strategy=SimpleStrategyType.BUY_CALL,
                strike=signal.metadata['strike'],
                expiry=signal.metadata['expiry'],
                quantity=signal.quantity,
                price=signal.price
            )
            await self.options_trader.place_order(order)
        
        elif signal.action == "bull_spread":
            # Even simpler with convenience function - no broker needed
            await bull_call_spread(
                underlying=signal.symbol,
                strikes=signal.metadata['strikes'],
                expiry=signal.metadata['expiry'],
                quantity=signal.quantity,
                price=signal.price
            )
    
    async def analyze_market(self):
        # Get market data - no broker needed
        current_price = await self.options_data.get_current_price()
        iv_percentile = await self.options_data.get_iv_percentile(expiry, "call")
        
        # Make trading decisions based on simple data
        if iv_percentile > 80:
            # High IV - sell options
            pass
```

### 2. Quick Trading

```python
# Initialize once
options_trader = SimpleOptionsTrader(broker)
set_simple_options_trader(options_trader)

# Trade anywhere without broker parameter
result = await buy_call("AAPL", 150.0, "2024-01-19", 1, 5.50)
result = await bull_call_spread("AAPL", [150.0, 155.0], "2024-01-19", 1, 2.50)
result = await iron_condor("AAPL", [140.0, 145.0, 155.0, 160.0], "2024-01-19", 1, 1.75)
```

### 3. Data Analysis

```python
# Initialize once
options_data = SimpleOptionData("AAPL", broker)
set_simple_options_data("AAPL", options_data)

# Analyze anywhere without broker parameter
current_price = await get_current_price("AAPL")
atm_options = await get_atm_options("AAPL", "2024-01-19")
iv_percentile = await get_iv_percentile("AAPL", "2024-01-19", "call")
smile_data = await get_volatility_smile("AAPL", "2024-01-19")
summary = await get_chain_summary("AAPL", "2024-01-19")
```

## Comparison: Before vs After

| Aspect | Before (Broker Exposed) | After (Broker Encapsulated) |
|--------|-------------------------|------------------------------|
| **User Interface** | `buy_call(broker, ...)` | `buy_call(...)` |
| **Strategy Code** | Pass broker everywhere | Initialize once, use anywhere |
| **Framework Exposure** | Users see broker details | Broker details hidden |
| **Consistency** | Different from OptionsHelper | Same pattern as OptionsHelper |
| **Complexity** | High (broker management) | Low (simple interface) |
| **Error Rate** | High (broker errors) | Low (encapsulated errors) |

## Benefits of Broker Encapsulation

### 1. **Clean Interface**
- Users focus on trading logic, not broker details
- Simple function signatures
- No broker parameter pollution

### 2. **Consistency**
- Same pattern as OptionsHelper
- Unified approach across the framework
- Familiar to users

### 3. **Error Prevention**
- Broker errors handled internally
- Validation at initialization
- Clear error messages

### 4. **Maintainability**
- Broker changes don't affect user code
- Easy to add new brokers
- Centralized broker management

### 5. **Usability**
- Lower barrier to entry
- Faster development
- Better for prototyping

## Migration Guide

### From Old Interface

**Old Code:**
```python
# Users had to pass broker everywhere
result = await buy_call(broker, "AAPL", 150.0, "2024-01-19", 1, 5.50)
```

**New Code:**
```python
# Initialize once
options_trader = SimpleOptionsTrader(broker)
set_simple_options_trader(options_trader)

# Use anywhere without broker
result = await buy_call("AAPL", 150.0, "2024-01-19", 1, 5.50)
```

### Strategy Migration

**Old Strategy:**
```python
class MyStrategy:
    def __init__(self, broker):
        self.broker = broker
    
    async def trade(self):
        result = await buy_call(self.broker, "AAPL", 150.0, "2024-01-19", 1, 5.50)
```

**New Strategy:**
```python
class MyStrategy:
    def __init__(self, broker):
        self.options_trader = SimpleOptionsTrader(broker)
        set_simple_options_trader(self.options_trader)
    
    async def trade(self):
        result = await buy_call("AAPL", 150.0, "2024-01-19", 1, 5.50)
```

## Conclusion

The broker encapsulation in the Simple Options Wrapper provides a clean, consistent interface that hides framework internals while maintaining all functionality. Users can focus on writing good trading strategies without dealing with broker complexity, exactly as requested.

This approach:
- Hides framework internals
- Provides simple strategy integrations
- Maintains tidy code
- Follows OptionsHelper pattern
- Reduces complexity for users 