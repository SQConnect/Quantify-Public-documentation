# Position Manager Documentation

The `PositionManager` is a singleton utility that provides portfolio and position information from any broker. You do not instantiate it directly—instead, you use the `get_positions(broker)` class method, passing your broker on first use. All subsequent calls (from any strategy or component) will use the same instance.

## Strategy Usage

```python
from src.portfolio.position_manager import PositionManager

class MyStrategy(BaseStrategy):
    async def run(self):
        # Get all positions as Position objects (pass broker on first use)
        positions = await PositionManager.get_positions(self.broker)
        for pos in positions:
            print(pos.symbol, pos.quantity, pos.get_value())
```

## Singleton Constructor (for first use)

```python
# On first use, pass your broker:
positions = await PositionManager.get_positions(broker)
# On subsequent use, just call get_positions():
positions = await PositionManager.get_positions()
```

## Methods

All methods return sensible defaults (0.0, False, empty dict) if something goes wrong.

**Position Info:**
- `await is_invested(symbol)` → bool
- `await position_exists(symbol)` → bool  
- `await get_position_size(symbol)` → float
- `await get_position_value(symbol)` → float
- `await get_positions()` → list of `Position` objects (see below)

**Cash Info:**
- `await get_available_cash()` → float (after reserve)
- `await get_total_cash()` → float (before reserve)
- `await get_buying_power()` → float

**Other:**
- `await check_margin()` → dict with margin info
- `await get_currencies()` → list of currencies with balances
- `await get_options_expiration_dates(symbol)` → list of dates (if broker supports options)
- `await liquidate_portfolio(emergency=False)` → dict with liquidation results
- `await get_portfolio_summary()` → dict with complete portfolio state

## Position Object

All positions returned are instances of the `Position` dataclass, with fields like:
- `symbol`, `quantity`, `avg_price`, `market_price`, `unrealized_pnl`, `realized_pnl`, `entry_time`, `exit_time`, `side`, `fees`, `strategy_name`

And methods like:
- `get_value()`, `get_unrealized_pnl()`, `is_open()`, `to_dict()`

## That's It

PositionManager is now a singleton, ensuring all strategies and components share a consistent, up-to-date view of the portfolio. Use `get_positions()` everywhere for best results. 