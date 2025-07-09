# Simple Options Trading

The Simple Options Trading wrapper makes options trading as easy as regular equity orders. No need to understand complex option chains, legs, or strategy definitions.

## Overview

Options trading in Quantify can be complex with multiple layers:
- Option chains and leg definitions
- Multi-leg order management
- Strategy-specific implementations
- Complex position tracking

The `SimpleOptionsTrader` abstracts all this complexity away, providing a clean interface that works just like regular equity trading.

## Quick Start

### Basic Usage

```python
from src.options.simple_options import SimpleOptionsTrader, SimpleOptionOrder, SimpleStrategyType

# Initialize (same as equity trading)
options_trader = SimpleOptionsTrader(broker)

# Buy a call option (as simple as buying stock)
order = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.BUY_CALL,
    strike=150.0,
    expiry="2024-01-19",
    quantity=1,
    price=5.50
)

result = await options_trader.place_order(order)
```

### Using Convenience Functions

Even simpler with convenience functions:

```python
from src.options.simple_options import buy_call, bull_call_spread

# Buy a call option
result = await buy_call(
    broker=broker,
    underlying="AAPL",
    strike=150.0,
    expiry="2024-01-19",
    quantity=1,
    price=5.50
)

# Place a bull call spread
result = await bull_call_spread(
    broker=broker,
    underlying="AAPL",
    strikes=[150.0, 155.0],
    expiry="2024-01-19",
    quantity=1,
    price=2.50
)
```

## Supported Strategies

### Single Leg Strategies

| Strategy | Description | Parameters |
|----------|-------------|------------|
| `BUY_CALL` | Buy a call option | `strike` |
| `SELL_CALL` | Sell a call option | `strike` |
| `BUY_PUT` | Buy a put option | `strike` |
| `SELL_PUT` | Sell a put option | `strike` |

### Vertical Spreads

| Strategy | Description | Parameters |
|----------|-------------|------------|
| `BULL_CALL_SPREAD` | Buy lower strike call, sell higher strike call | `strikes=[lower, higher]` |
| `BEAR_CALL_SPREAD` | Sell lower strike call, buy higher strike call | `strikes=[lower, higher]` |
| `BULL_PUT_SPREAD` | Sell higher strike put, buy lower strike put | `strikes=[lower, higher]` |
| `BEAR_PUT_SPREAD` | Buy higher strike put, sell lower strike put | `strikes=[lower, higher]` |

### Volatility Strategies

| Strategy | Description | Parameters |
|----------|-------------|------------|
| `LONG_STRADDLE` | Buy call and put at same strike | `strike` |
| `SHORT_STRADDLE` | Sell call and put at same strike | `strike` |
| `LONG_STRANGLE` | Buy call and put at different strikes | `strikes=[put_strike, call_strike]` |
| `SHORT_STRANGLE` | Sell call and put at different strikes | `strikes=[put_strike, call_strike]` |

### Income Strategies

| Strategy | Description | Parameters |
|----------|-------------|------------|
| `IRON_CONDOR` | Sell put spread + sell call spread | `strikes=[put_low, put_high, call_low, call_high]` |
| `BUTTERFLY` | Buy 2 ATM, sell 1 OTM call + 1 ITM call | `strikes=[low, mid, high]` |

## Order Parameters

### Required Parameters

- `underlying`: Stock symbol (e.g., "AAPL")
- `strategy`: Strategy type from `SimpleStrategyType`
- `quantity`: Number of contracts

### Optional Parameters

- `price`: Limit price (omit for market orders)
- `order_type`: "limit" or "market" (default: "limit")
- `expiry`: Expiration date as string ("YYYY-MM-DD") or datetime
- `days_to_expiry`: Days until expiration (auto-calculates expiry date)
- `metadata`: Additional data for the order

### Strategy-Specific Parameters

- **Single leg**: `strike` (float)
- **Spreads**: `strikes` (list of 2 floats)
- **Iron Condor**: `strikes` (list of 4 floats)
- **Butterfly**: `strikes` (list of 3 floats)

## Examples

### Single Leg Options

```python
# Buy a call option
call_order = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.BUY_CALL,
    strike=150.0,
    expiry="2024-01-19",
    quantity=1,
    price=5.50
)

# Sell a put option (income)
put_order = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.SELL_PUT,
    strike=140.0,
    expiry="2024-01-19",
    quantity=1,
    price=3.25  # Credit received
)
```

### Vertical Spreads

```python
# Bull call spread (defined risk)
bull_spread = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.BULL_CALL_SPREAD,
    strikes=[150.0, 155.0],  # Buy 150 call, sell 155 call
    expiry="2024-01-19",
    quantity=1,
    price=2.50  # Net debit
)

# Bear put spread
bear_spread = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.BEAR_PUT_SPREAD,
    strikes=[155.0, 150.0],  # Buy 155 put, sell 150 put
    expiry="2024-01-19",
    quantity=1,
    price=3.00  # Net debit
)
```

### Volatility Strategies

```python
# Long straddle (volatility play)
straddle = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.LONG_STRADDLE,
    strike=150.0,  # Buy call and put at same strike
    expiry="2024-01-19",
    quantity=1,
    price=8.00  # Net debit
)

# Long strangle (cheaper volatility play)
strangle = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.LONG_STRANGLE,
    strikes=[145.0, 155.0],  # Buy 145 put, 155 call
    expiry="2024-01-19",
    quantity=1,
    price=6.50  # Net debit
)
```

### Income Strategies

```python
# Iron condor (income strategy)
iron_condor = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.IRON_CONDOR,
    strikes=[140.0, 145.0, 155.0, 160.0],  # 4 strikes
    expiry="2024-01-19",
    quantity=1,
    price=1.75  # Net credit
)
```

## Advanced Usage

### Using Days to Expiry

```python
# Automatically calculate expiry date
order = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.BUY_CALL,
    strike=150.0,
    days_to_expiry=30,  # 30 days from now
    quantity=1,
    price=5.50
)
```

### Market Orders

```python
# Market order (no price needed)
market_order = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.BUY_CALL,
    strike=150.0,
    expiry="2024-01-19",
    quantity=1,
    order_type="market"
)
```

### Multiple Strategies

```python
# Place multiple strategies in a loop
strategies = [
    (SimpleStrategyType.BUY_CALL, [150.0]),
    (SimpleStrategyType.BULL_CALL_SPREAD, [150.0, 155.0]),
    (SimpleStrategyType.LONG_STRADDLE, [150.0]),
]

for strategy_type, strikes in strategies:
    order = SimpleOptionOrder(
        underlying="AAPL",
        strategy=strategy_type,
        strike=strikes[0] if len(strikes) == 1 else None,
        strikes=strikes if len(strikes) > 1 else None,
        expiry="2024-01-19",
        quantity=1
    )
    result = await options_trader.place_order(order)
```

## Position Management

### Get Current Positions

```python
# Get all options positions
positions = await options_trader.get_positions()

for position in positions:
    print(f"Position: {position.underlying} {position.strategy.value}")
    print(f"  P&L: ${position.pnl}")
    print(f"  Delta: {position.delta}")
    print(f"  Days to expiry: {position.days_to_expiry}")
```

### Close Positions

```python
# Close a position
result = await options_trader.close_position(
    position_id="position_123",
    price=2.50  # Optional limit price
)
```

## Comparison: Old vs New

### Old Way (Complex)

```python
# 1. Get option chain
option_chain = await broker.get_options_chain("AAPL", expiry)

# 2. Find specific options
call_150 = None
call_155 = None
for option in option_chain['calls']:
    if abs(option.strike - 150.0) < 0.01:
        call_150 = option
    elif abs(option.strike - 155.0) < 0.01:
        call_155 = option

# 3. Define legs manually
legs = [
    {"leg": call_150, "side": "Buy", "quantity": 1},
    {"leg": call_155, "side": "Sell", "quantity": 1}
]

# 4. Place multi-leg order
result = await broker.place_multi_leg_order(legs, "Limit", 2.50)
```

### New Way (Simple)

```python
# 1. Define strategy
order = SimpleOptionOrder(
    underlying="AAPL",
    strategy=SimpleStrategyType.BULL_CALL_SPREAD,
    strikes=[150.0, 155.0],
    expiry="2024-01-19",
    quantity=1,
    price=2.50
)

# 2. Place order
result = await options_trader.place_order(order)
```

## Benefits

1. **Simplicity**: No need to understand option chains or leg definitions
2. **Consistency**: Same interface for all strategy types
3. **Error Prevention**: Built-in validation and error handling
4. **Flexibility**: Support for both single and multi-leg strategies
5. **Convenience**: Helper functions for common strategies
6. **Integration**: Works with existing broker infrastructure

## Error Handling

The wrapper includes comprehensive error handling:

```python
try:
    result = await options_trader.place_order(order)
except ValueError as e:
    print(f"Validation error: {e}")
except Exception as e:
    print(f"Trading error: {e}")
```

Common validation errors:
- Missing required parameters for strategy type
- Invalid strike prices
- Option not found in chain
- Invalid expiry date

## Integration with Strategies

The simple options wrapper can be easily integrated into trading strategies:

```python
class MyOptionsStrategy(BaseStrategy):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.options_trader = SimpleOptionsTrader(self.broker)
    
    async def on_signal(self, signal):
        if signal.action == "buy_call":
            order = SimpleOptionOrder(
                underlying=signal.symbol,
                strategy=SimpleStrategyType.BUY_CALL,
                strike=signal.metadata['strike'],
                expiry=signal.metadata['expiry'],
                quantity=signal.quantity,
                price=signal.price
            )
            await self.options_trader.place_order(order)
```

This makes options trading as straightforward as regular equity trading while maintaining all the power and flexibility of complex options strategies. 