# Position Manager Documentation

The `PositionManager` is a simple utility that queries portfolio information from any broker. You pass it a broker instance, and it gives you position data, cash amounts, and portfolio info.

**For strategies:** All strategies that inherit from `BaseStrategy` automatically get `self.portfolio` - you don't need to create anything.

## Strategy Usage

```python
class MyStrategy(BaseStrategy):
    async def run(self):
        # Check if you own something
        invested = await self.portfolio.is_invested("AAPL")
        
        # Get position size
        size = await self.portfolio.get_position_size("AAPL")  # returns 0.0 if no position
        
        # Get available cash 
        cash = await self.portfolio.get_available_cash()  # respects 5% reserve by default
        
        # Get portfolio summary
        summary = await self.portfolio.get_portfolio_summary()
```

## Constructor (for non-strategy usage)

```python
PositionManager(
    broker,                               # Any connected broker instance
    cash_reserve_percentage=0.05,         # Keep 5% cash in reserve
    max_position_percentage=0.10          # Max 10% per position (not enforced, just stored)
)
```

## Methods

All methods return sensible defaults (0.0, False, empty dict) if something goes wrong.

**Position Info:**
- `await is_invested(symbol)` → bool
- `await position_exists(symbol)` → bool  
- `await get_position_size(symbol)` → float
- `await get_position_value(symbol)` → float

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

## That's It

It's just a thin wrapper around broker calls with some caching (30 seconds) and reserve calculation. Nothing fancy. 