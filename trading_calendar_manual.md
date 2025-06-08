# Trading Calendar Manual

## Overview
The Trading Calendar is a powerful tool integrated into the BaseStrategy class that provides comprehensive market timing and scheduling functionality. It helps strategies make informed decisions based on market hours, trading days, and market sessions.

## Basic Usage

### Initialization
The Trading Calendar is automatically initialized in the BaseStrategy class. You can configure it through your strategy's config:

```python
config = {
    'market_code': 'US',  # Default market
    'reference_date': None  # Optional, for backtesting
}
strategy = MyStrategy(name="MyStrategy", config=config)
```

### Market Codes
Supported market codes:
- `US` - United States markets (NYSE, NASDAQ)
- `EU` - European markets
- `ASIA` - Asian markets
- `CRYPTO` - Cryptocurrency markets (24/7)

## Core Functions

### Market Status Checks

#### 1. Check if Market is Open
```python
if strategy.is_market_open():
    # Market is currently open
    # Process trades, update positions, etc.
else:
    # Market is closed
    # Handle closed market logic
```

#### 2. Get Current Market Status
```python
status = strategy.get_market_status()
if status == MarketStatus.PRE_MARKET:
    # Handle pre-market trading
elif status == MarketStatus.REGULAR:
    # Handle regular market hours
elif status == MarketStatus.AFTER_HOURS:
    # Handle after-hours trading
elif status == MarketStatus.CLOSED:
    # Handle closed market
```

### Trading Day Management

#### 1. Check if Date is a Trading Day
```python
check_date = datetime.now().date()
if strategy.is_trading_day(check_date):
    # It's a trading day
    # Proceed with trading activities
else:
    # It's a non-trading day (weekend or holiday)
    # Handle non-trading day logic
```

#### 2. Get Next Trading Day
```python
current_date = datetime.now().date()
next_trading_day = strategy.get_next_trading_day(current_date)
# Use next_trading_day for planning future trades
```

#### 3. Get Trading Days in Range
```python
start_date = datetime(2024, 1, 1).date()
end_date = datetime(2024, 1, 31).date()
trading_days = strategy.get_trading_days_in_range(start_date, end_date)
# Use trading_days for backtesting or planning
```

### Market Hours and Sessions

#### 1. Get Market Hours
```python
# Get today's market hours
market_hours = strategy.get_market_hours()

# Get market hours for specific date
specific_date = datetime(2024, 1, 1).date()
market_hours = strategy.get_market_hours(specific_date)
```

#### 2. Get Market Session Times
```python
check_date = datetime.now().date()
session_times = strategy.get_market_session_times(check_date)
# Access different session times:
# - session_times['pre_market']
# - session_times['regular']
# - session_times['after_hours']
```

### Holiday Management

#### 1. Check if Date is a Holiday
```python
check_date = datetime.now().date()
if strategy.is_holiday(check_date):
    # It's a market holiday
    # Handle holiday logic
```

#### 2. Get Market Information
```python
market_info = strategy.get_market_info()
# Access market information:
# - market_info['name']
# - market_info['timezone']
# - market_info['holidays']
# - market_info['sessions']
```

## Advanced Usage

### Backtesting Support

#### 1. Set Reference Date
```python
# Set a specific date for backtesting
reference_date = datetime(2024, 1, 1)
strategy.set_reference_date(reference_date)
```

#### 2. Get Trading Calendar for Backtesting
```python
start_date = datetime(2024, 1, 1).date()
end_date = datetime(2024, 1, 31).date()
calendar_info = strategy.get_trading_calendar_for_backtest(start_date, end_date)
# Use calendar_info for backtesting setup
```

### Best Practices

1. **Always Check Market Status**
   ```python
   async def on_market_data(self, event):
       if not self.is_market_open():
           return
       # Process market data
   ```

2. **Handle Different Market Sessions**
   ```python
   async def process_trades(self):
       status = self.get_market_status()
       if status == MarketStatus.PRE_MARKET:
           # Use more conservative parameters for pre-market
           self.stop_loss_pct = 0.03
       elif status == MarketStatus.REGULAR:
           # Use standard parameters for regular hours
           self.stop_loss_pct = 0.05
   ```

3. **Plan for Holidays**
   ```python
   async def initialize(self) -> bool:
       # Check next few days for holidays
       today = datetime.now().date()
       for i in range(5):
           check_date = today + timedelta(days=i)
           if self.is_holiday(check_date):
               self.logger.info(f"Market holiday on {check_date}")
   ```

4. **Use Market Hours for Scheduling**
   ```python
   async def schedule_tasks(self):
       market_hours = self.get_market_hours()
       pre_market_start = market_hours['pre_market']['start']
       regular_start = market_hours['regular']['start']
       
       # Schedule tasks based on market hours
       if datetime.now() < pre_market_start:
           # Schedule pre-market tasks
           pass
       elif datetime.now() < regular_start:
           # Schedule regular market tasks
           pass
   ```

## Error Handling

The Trading Calendar methods are designed to be robust and handle edge cases:

```python
try:
    if not self.is_market_open():
        self.logger.warning("Market is closed")
        return
        
    market_hours = self.get_market_hours()
    if not market_hours:
        self.logger.error("Failed to get market hours")
        return
        
except Exception as e:
    self.logger.error(f"Error accessing trading calendar: {e}")
    # Handle error appropriately
```

## Performance Considerations

1. The Trading Calendar is optimized for performance and caches results where appropriate
2. Market status checks are lightweight and can be called frequently
3. For backtesting, use `set_reference_date()` to avoid unnecessary calculations

## Troubleshooting

Common issues and solutions:

1. **Market Status Not Updating**
   - Check if the market code is correct
   - Verify the system timezone matches the market timezone
   - Use `set_reference_date()` for backtesting

2. **Holiday Detection Issues**
   - Verify the market code supports the holiday calendar
   - Check if the date is within the supported range
   - Use `get_market_info()` to verify holiday data

3. **Session Time Discrepancies**
   - Check the market's timezone settings
   - Verify daylight saving time handling
   - Use `get_market_session_times()` to debug session times

## API Reference

### Market Status Methods
- `is_market_open(dt: Optional[datetime] = None) -> bool`
- `get_market_status(dt: Optional[datetime] = None) -> MarketStatus`

### Trading Day Methods
- `is_trading_day(check_date: date) -> bool`
- `get_next_trading_day(start_date: date) -> date`
- `get_trading_days_in_range(start_date: date, end_date: date) -> List[date]`

### Market Hours Methods
- `get_market_hours(target_date: Optional[date] = None) -> Dict`
- `get_market_session_times(check_date: date) -> Dict`

### Holiday Methods
- `is_holiday(check_date: date) -> bool`
- `get_market_info() -> Dict`

### Backtesting Methods
- `set_reference_date(reference_date: Optional[datetime])`
- `get_trading_calendar_for_backtest(start_date: date, end_date: date) -> Dict` 