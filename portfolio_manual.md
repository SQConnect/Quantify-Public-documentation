# Portfolio Management Manual

## Table of Contents
1. [Introduction](#introduction)
2. [PositionManager Overview](#positionmanager-overview)
3. [Initialization](#initialization)
4. [Position Management](#position-management)
5. [Cash Management](#cash-management)
6. [Margin Information](#margin-information)
7. [Portfolio Summary](#portfolio-summary)
8. [Multi-Currency Support](#multi-currency-support)
9. [Options Support](#options-support)
10. [Portfolio Liquidation](#portfolio-liquidation)
11. [Caching and Performance](#caching-and-performance)
12. [Best Practices](#best-practices)
13. [Examples](#examples)

## Introduction

The PositionManager provides a unified interface for strategies to interact with portfolio data across any broker. It handles position tracking, cash management, margin monitoring, and provides comprehensive portfolio summaries. The class works with the unified broker interface, making it broker-agnostic.

**Note**: Risk management functionality is handled separately and will be integrated as a standalone component.

## PositionManager Overview

The PositionManager is the core class for portfolio operations:

```python
from src.portfolio.position_manager import PositionManager
from src.broker_interface.broker_factory import BrokerFactory

# Initialize broker (any supported broker)
broker = BrokerFactory.create_broker('binance', config)

# Initialize PositionManager
position_manager = PositionManager(
    broker=broker,
    cash_reserve_percentage=0.05,  # 5% cash reserve
    max_position_percentage=0.10   # 10% max position size
)
```

### Key Features

- **Broker Agnostic**: Works with any broker from BrokerFactory
- **Unified Interface**: Same API across all brokers
- **Performance Optimized**: Built-in caching with configurable TTL
- **Multi-Currency**: Support for multiple currencies
- **Options Ready**: Built-in options support
- **Emergency Features**: Portfolio liquidation capabilities

## Initialization

### Basic Initialization

```python
from src.portfolio.position_manager import PositionManager

# Minimal initialization
position_manager = PositionManager(broker=broker)

# With custom settings
position_manager = PositionManager(
    broker=broker,
    cash_reserve_percentage=0.10,  # 10% cash reserve
    max_position_percentage=0.05   # 5% max position size
)
```

### Configuration Parameters

- `broker`: Any broker instance from BrokerFactory
- `cash_reserve_percentage`: Minimum cash reserve (default: 5%)
- `max_position_percentage`: Maximum position size as % of portfolio (default: 10%)

## Position Management

### Checking Investment Status

```python
# Check if currently invested in a symbol
is_invested = await position_manager.is_invested('AAPL')
print(f"Invested in AAPL: {is_invested}")

# Check if position record exists (even with 0 quantity)
position_exists = await position_manager.position_exists('AAPL')
print(f"Position record exists: {position_exists}")
```

### Getting Position Information

```python
# Get position size (positive=long, negative=short, 0=no position)
position_size = await position_manager.get_position_size('AAPL')
print(f"AAPL position size: {position_size}")

# Get current market value of position
position_value = await position_manager.get_position_value('AAPL', 'USD')
print(f"AAPL position value: ${position_value}")
```

### Working with Multiple Positions

```python
# Check multiple symbols
symbols = ['AAPL', 'MSFT', 'GOOGL', 'ETHUSDT']
for symbol in symbols:
    invested = await position_manager.is_invested(symbol)
    if invested:
        size = await position_manager.get_position_size(symbol)
        value = await position_manager.get_position_value(symbol)
        print(f"{symbol}: Size={size}, Value=${value:.2f}")
```

## Cash Management

### Available Cash

```python
# Get available cash (after reserves)
available_cash = await position_manager.get_available_cash('USD')
print(f"Available cash: ${available_cash}")

# Get total cash (before reserves)
total_cash = await position_manager.get_total_cash('USD')
print(f"Total cash: ${total_cash}")

# Get buying power (includes margin)
buying_power = await position_manager.get_buying_power('USD')
print(f"Buying power: ${buying_power}")
```

### Multi-Currency Cash

```python
# Check cash in different currencies
usd_cash = await position_manager.get_available_cash('USD')
eur_cash = await position_manager.get_available_cash('EUR')
btc_cash = await position_manager.get_available_cash('BTC')

print(f"USD: ${usd_cash}, EUR: €{eur_cash}, BTC: ₿{btc_cash}")
```

## Margin Information

### Checking Margin Status

```python
# Get detailed margin information
margin_info = await position_manager.check_margin()

print(f"Margin used: ${margin_info['margin_used']}")
print(f"Margin available: ${margin_info['margin_available']}")
print(f"Margin ratio: {margin_info['margin_ratio']:.2%}")

# Force refresh margin data
margin_info = await position_manager.check_margin(force_refresh=True)
```

### Margin Monitoring

```python
# Monitor margin utilization
async def monitor_margin():
    margin_info = await position_manager.check_margin()
    
    if margin_info['margin_ratio'] > 0.8:  # 80% utilization
        print("⚠️ High margin utilization detected!")
        
    if margin_info['margin_ratio'] > 0.9:  # 90% utilization
        print("🚨 Critical margin utilization!")
        # Consider reducing positions
```

## Portfolio Summary

### Getting Complete Portfolio State

```python
# Get comprehensive portfolio summary
summary = await position_manager.get_portfolio_summary()

# Access key information
print(f"Broker: {summary['broker_name']}")
print(f"Account ID: {summary['account_id']}")
print(f"Total Portfolio Value: ${summary['total_portfolio_value']}")
print(f"Total Cash: ${summary['total_cash']}")
print(f"Positions Value: ${summary['positions_value']}")
print(f"Total Positions: {summary['total_positions']}")
print(f"Long Positions: {summary['long_positions']}")
print(f"Short Positions: {summary['short_positions']}")
print(f"Margin Used: ${summary['margin_used']}")
print(f"Margin Available: ${summary['margin_available']}")
```

### Portfolio Health Check

```python
async def portfolio_health_check():
    summary = await position_manager.get_portfolio_summary()
    
    # Check margin utilization
    if summary['margin_ratio'] > 0.7:
        print(f"⚠️ High margin utilization: {summary['margin_ratio']:.1%}")
    
    # Check cash levels
    cash_ratio = summary['total_cash'] / summary['total_portfolio_value']
    if cash_ratio < 0.05:
        print(f"⚠️ Low cash reserves: {cash_ratio:.1%}")
    
    # Check position count
    if summary['total_positions'] > 50:
        print(f"⚠️ High position count: {summary['total_positions']}")
    
    print("✅ Portfolio health check completed")
```

## Multi-Currency Support

### Working with Multiple Currencies

```python
# Get list of currencies with balances
currencies = await position_manager.get_currencies()
print(f"Available currencies: {currencies}")

# Get cash balances for each currency
for currency in currencies:
    balance = await position_manager.get_total_cash(currency)
    print(f"{currency}: {balance}")
```

### Currency Enumeration

```python
from src.portfolio.position_manager import Currency

# Use predefined currency constants
usd_cash = await position_manager.get_available_cash(Currency.USD.value)
btc_cash = await position_manager.get_available_cash(Currency.BTC.value)
eth_cash = await position_manager.get_available_cash(Currency.ETH.value)
```

## Options Support

### Options Expiration Dates

```python
# Get expiration dates for options on a symbol
expiration_dates = await position_manager.get_options_expiration_dates('AAPL')

print("AAPL Options Expiration Dates:")
for date in expiration_dates:
    print(f"  {date.strftime('%Y-%m-%d')}")
```

## Portfolio Liquidation

### Emergency Liquidation

```python
# Emergency liquidation with market orders
results = await position_manager.liquidate_portfolio(emergency=True)

print(f"Total positions: {results['total_positions']}")
print(f"Successful orders: {results['successful_orders']}")
print(f"Failed orders: {results['failed_orders']}")
print(f"Order IDs: {results['order_ids']}")

if results['errors']:
    print("Errors encountered:")
    for error in results['errors']:
        print(f"  - {error}")
```

### Controlled Liquidation

```python
# Controlled liquidation with limit orders
results = await position_manager.liquidate_portfolio(emergency=False)

# Monitor liquidation progress
for order_id in results['order_ids']:
    # Check order status through broker
    order_status = await broker.get_order_status(order_id)
    print(f"Order {order_id}: {order_status}")
```

## Caching and Performance

### Cache Management

```python
# Force refresh cache when needed
await position_manager.force_refresh_cache()

# Cache is automatically managed with 2-minute TTL
# No manual intervention usually needed
```

### Performance Optimization

The PositionManager uses intelligent caching:

- **Cache TTL**: 2 minutes (configurable)
- **Automatic Refresh**: Cache refreshes when expired
- **Graceful Degradation**: Uses stale cache if refresh fails
- **Minimal API Calls**: Batches broker requests

## Best Practices

### 1. Regular Monitoring

```python
async def portfolio_monitor():
    """Regular portfolio monitoring routine."""
    
    # Get current state
    summary = await position_manager.get_portfolio_summary()
    
    # Log key metrics
    logger.info(f"Portfolio Value: ${summary['total_portfolio_value']}")
    logger.info(f"Margin Ratio: {summary['margin_ratio']:.1%}")
    logger.info(f"Position Count: {summary['total_positions']}")
    
    # Check for alerts
    if summary['margin_ratio'] > 0.8:
        logger.warning("High margin utilization")
    
    if summary['total_positions'] > 100:
        logger.warning("High position count")
```

### 2. Error Handling

```python
async def safe_position_check(symbol: str):
    """Safely check position with error handling."""
    try:
        is_invested = await position_manager.is_invested(symbol)
        if is_invested:
            size = await position_manager.get_position_size(symbol)
            value = await position_manager.get_position_value(symbol)
            return {'invested': True, 'size': size, 'value': value}
        return {'invested': False}
        
    except Exception as e:
        logger.error(f"Error checking position for {symbol}: {e}")
        return {'error': str(e)}
```

### 3. Cash Reserve Management

```python
async def check_available_funds(required_amount: float):
    """Check if sufficient funds available for trade."""
    
    available_cash = await position_manager.get_available_cash()
    
    if available_cash >= required_amount:
        return True
    
    # Check if we can use margin
    buying_power = await position_manager.get_buying_power()
    if buying_power >= required_amount:
        # Check margin safety
        margin_info = await position_manager.check_margin()
        if margin_info['margin_ratio'] < 0.7:  # Keep under 70%
            return True
    
    return False
```

## Examples

### Complete Trading Strategy Integration

```python
class ExampleStrategy:
    def __init__(self, broker):
        self.position_manager = PositionManager(
            broker=broker,
            cash_reserve_percentage=0.05,
            max_position_percentage=0.10
        )
    
    async def analyze_and_trade(self, symbol: str):
        """Example trading logic with position management."""
        
        # Check current position
        is_invested = await self.position_manager.is_invested(symbol)
        current_size = await self.position_manager.get_position_size(symbol)
        
        # Get portfolio state
        summary = await self.position_manager.get_portfolio_summary()
        
        # Check if we can trade
        available_cash = await self.position_manager.get_available_cash()
        margin_ratio = summary['margin_ratio']
        
        if margin_ratio > 0.7:
            print("Margin too high, skipping trade")
            return
        
        # Calculate position size based on available cash
        max_trade_amount = available_cash * 0.1  # 10% of available cash
        
        # Your trading logic here
        if not is_invested and should_buy_signal():
            # Calculate shares to buy
            current_price = await get_current_price(symbol)
            shares = int(max_trade_amount / current_price)
            
            # Place order through broker
            order_result = await self.broker.place_order(
                symbol=symbol,
                side='buy',
                quantity=shares,
                order_type='market'
            )
            
            print(f"Bought {shares} shares of {symbol}")
        
        elif is_invested and should_sell_signal():
            # Close position
            await self.broker.place_order(
                symbol=symbol,
                side='sell',
                quantity=abs(current_size),
                order_type='market'
            )
            
            print(f"Sold {symbol} position")

### Portfolio Rebalancing

```python
async def rebalance_portfolio():
    """Example portfolio rebalancing logic."""
    
    # Get current state
    summary = await position_manager.get_portfolio_summary()
    total_value = summary['total_portfolio_value']
    
    # Define target allocations
    target_allocations = {
        'AAPL': 0.20,  # 20%
        'MSFT': 0.20,  # 20%
        'GOOGL': 0.15, # 15%
        'ETHUSDT': 0.10, # 10%
        # Cash: 35%
    }
    
    for symbol, target_weight in target_allocations.items():
        current_value = await position_manager.get_position_value(symbol)
        current_weight = current_value / total_value
        
        target_value = total_value * target_weight
        difference = target_value - current_value
        
        if abs(difference) > total_value * 0.02:  # 2% threshold
            print(f"{symbol}: Current {current_weight:.1%}, "
                  f"Target {target_weight:.1%}, "
                  f"Difference: ${difference:.2f}")
            
            # Rebalancing logic here
            # ...
```

This manual focuses exclusively on the PositionManager functionality. Risk management features will be documented separately once the risk management refactoring is complete. 