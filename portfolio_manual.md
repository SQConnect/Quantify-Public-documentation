# Portfolio Management Manual

## Table of Contents
1. [Introduction](#introduction)
2. [Portfolio Structure](#portfolio-structure)
3. [Account Management](#account-management)
4. [Position Management](#position-management)
5. [Cash Management](#cash-management)
6. [Risk Management](#risk-management)
7. [Multi-Currency Support](#multi-currency-support)
8. [Portfolio Monitoring](#portfolio-monitoring)
9. [Best Practices](#best-practices)
10. [Examples](#examples)

## Introduction

The portfolio management system provides a unified interface for managing positions, cash, and risk across all asset classes. It supports multiple brokers and provides comprehensive tools for portfolio monitoring and risk management.

## Portfolio Structure

The portfolio system is organized into several key components:

```python
from src.portfolio.portfolio import Portfolio

# Initialize portfolio manager
portfolio = Portfolio(
    broker_accounts={
        'kraken': kraken_broker,
        'saxo': saxo_broker
    },
    risk_limits={
        'max_position_size': 0.1,  # 10% of portfolio
        'max_cash_utilization': 0.8,  # 80% of available cash
        'min_cash_reserve': 0.2,  # 20% cash reserve
        'max_leverage': 2.0,  # 2x leverage
        'max_drawdown': 0.15  # 15% max drawdown
    }
)
```

### Key Components

1. **Broker Accounts**
   - Multiple broker support
   - Unified interface for all brokers
   - Automatic synchronization

2. **Risk Limits**
   - Position size limits
   - Cash utilization limits
   - Leverage limits
   - Drawdown limits

## Account Management

### Account Summary

Get a complete overview of your accounts across all brokers:

```python
# Get account summary
summary = await portfolio.get_account_summary()

# Access key metrics
total_equity = summary['total_equity']
total_margin_used = summary['total_margin_used']
available_margin = summary['available_margin']
margin_utilization = summary['margin_utilization']

# Access broker-specific information
for broker, account in summary['accounts'].items():
    broker_equity = account['equity']
    broker_margin = account['margin_used']
```

### Account Monitoring

Monitor account changes and receive alerts:

```python
# Subscribe to account updates
await portfolio.subscribe_to_account_updates(callback=handle_account_update)

# Example callback
async def handle_account_update(update):
    if update['type'] == 'equity_change':
        logger.info(f"Equity changed by {update['change']}")
    elif update['type'] == 'margin_call':
        logger.warning("Margin call received!")
```

## Position Management

### Position Tracking

Track positions across all asset classes:

```python
# Get all positions
positions = await portfolio.get_positions()

# Get positions by asset class
equity_positions = await portfolio.get_positions(asset_class='equity')
options_positions = await portfolio.get_positions(asset_class='options')
crypto_positions = await portfolio.get_positions(asset_class='crypto')

# Get positions by broker
kraken_positions = await portfolio.get_positions(broker='kraken')
saxo_positions = await portfolio.get_positions(broker='saxo')
```

### Position Risk Analysis

Analyze risk metrics for positions:

```python
# Get risk metrics for all positions
risk_metrics = await portfolio.get_position_risk()

# Get risk metrics for specific asset class
options_risk = await portfolio.get_position_risk(asset_class='options')

# Access risk metrics
for position_id, metrics in risk_metrics.items():
    position_value = metrics['position_value']
    margin_used = metrics['margin_used']
    leverage = metrics['leverage']
    var_95 = metrics['var_95']  # 95% Value at Risk
```

## Cash Management

### Cash Balances

Manage cash across all accounts:

```python
# Get cash balances
cash = await portfolio.get_cash_balances()

# Get cash by currency
usd_cash = await portfolio.get_cash_balances(currency='USD')
eur_cash = await portfolio.get_cash_balances(currency='EUR')

# Get cash by broker
kraken_cash = await portfolio.get_cash_balances(broker='kraken')
saxo_cash = await portfolio.get_cash_balances(broker='saxo')
```

### Cash Utilization

Monitor and control cash utilization:

```python
# Get cash utilization metrics
utilization = await portfolio.get_cash_utilization()

# Check if cash utilization is within limits
if utilization['total_utilization'] > portfolio.risk_limits['max_cash_utilization']:
    logger.warning("Cash utilization exceeds limit")
```

## Risk Management

### Portfolio Risk Metrics

Calculate comprehensive risk metrics:

```python
# Get portfolio risk metrics
risk_metrics = await portfolio.get_portfolio_risk()

# Access key metrics
total_var = risk_metrics['total_var']
portfolio_beta = risk_metrics['portfolio_beta']
correlation_matrix = risk_metrics['correlation_matrix']
```

### Risk Limits

Enforce risk limits across the portfolio:

```python
# Check position size limits
if await portfolio.check_position_size_limit(symbol='AAPL', size=1000):
    # Position size is within limits
    pass

# Check leverage limits
if await portfolio.check_leverage_limit(leverage=1.5):
    # Leverage is within limits
    pass
```

## Multi-Currency Support

### Currency Conversion

Handle multiple currencies:

```python
# Get exchange rates
rates = await portfolio.get_exchange_rates()

# Convert amount between currencies
usd_amount = await portfolio.convert_currency(100, 'EUR', 'USD')

# Get portfolio value in specific currency
portfolio_value = await portfolio.get_portfolio_value(currency='USD')
```

### Currency Risk

Manage currency exposure:

```python
# Get currency exposure
exposure = await portfolio.get_currency_exposure()

# Hedge currency risk
if exposure['EUR'] > 0.3:  # More than 30% EUR exposure
    await portfolio.hedge_currency('EUR')
```

## Portfolio Monitoring

### Real-time Monitoring

Monitor portfolio changes in real-time:

```python
# Subscribe to portfolio updates
await portfolio.subscribe_to_portfolio_updates(callback=handle_portfolio_update)

# Example callback
async def handle_portfolio_update(update):
    if update['type'] == 'position_change':
        logger.info(f"Position changed: {update['symbol']}")
    elif update['type'] == 'risk_alert':
        logger.warning(f"Risk alert: {update['message']}")
```

### Performance Tracking

Track portfolio performance:

```python
# Get performance metrics
performance = await portfolio.get_performance_metrics()

# Access key metrics
total_return = performance['total_return']
sharpe_ratio = performance['sharpe_ratio']
max_drawdown = performance['max_drawdown']
```

## Best Practices

1. **Risk Management**
   - Set appropriate risk limits for your strategy
   - Monitor risk metrics regularly
   - Implement automated risk controls
   - Diversify across asset classes and currencies

2. **Cash Management**
   - Maintain adequate cash reserves
   - Monitor cash utilization
   - Implement cash sweep strategies
   - Consider currency exposure

3. **Position Management**
   - Size positions according to risk limits
   - Monitor position concentration
   - Implement position limits
   - Regular position review

4. **Portfolio Monitoring**
   - Set up real-time monitoring
   - Implement automated alerts
   - Regular performance review
   - Document all changes

## Examples

### Complete Portfolio Management Workflow

```python
async def manage_portfolio(portfolio):
    # Get portfolio summary
    summary = await portfolio.get_account_summary()
    
    # Check margin utilization
    if summary['margin_utilization'] > 0.7:
        logger.warning("High margin utilization detected")
        # Implement risk reduction measures
    
    # Analyze position risks
    risk_metrics = await portfolio.get_position_risk()
    for position_id, metrics in risk_metrics.items():
        # Check leverage
        if metrics['leverage'] > portfolio.risk_limits['max_leverage']:
            logger.warning(f"High leverage detected for position {position_id}")
            # Consider reducing position size
        
        # Check VaR
        if metrics['var_95'] > summary['total_equity'] * 0.1:
            logger.warning(f"High VaR detected for position {position_id}")
            # Consider hedging or reducing position
    
    # Monitor cash utilization
    utilization = await portfolio.get_cash_utilization()
    if utilization['total_utilization'] > portfolio.risk_limits['max_cash_utilization']:
        logger.warning("High cash utilization")
        # Consider reducing positions or adding funds
    
    # Check currency exposure
    exposure = await portfolio.get_currency_exposure()
    for currency, amount in exposure.items():
        if amount > 0.3:  # More than 30% exposure
            logger.warning(f"High {currency} exposure")
            # Consider hedging
    
    # Get performance metrics
    performance = await portfolio.get_performance_metrics()
    if performance['max_drawdown'] > portfolio.risk_limits['max_drawdown']:
        logger.warning("Maximum drawdown exceeded")
        # Implement risk reduction measures
```

### Automated Portfolio Rebalancing

```python
async def rebalance_portfolio(portfolio, target_weights):
    # Get current positions
    positions = await portfolio.get_positions()
    
    # Calculate required changes
    for symbol, target_weight in target_weights.items():
        current_value = positions.get(symbol, {}).get('market_value', 0)
        target_value = portfolio.get_portfolio_value() * target_weight
        
        if abs(current_value - target_value) > portfolio.get_portfolio_value() * 0.01:  # 1% threshold
            # Calculate required position change
            change = target_value - current_value
            
            # Execute rebalancing trade
            if change > 0:
                await portfolio.enter_position(symbol, change)
            else:
                await portfolio.exit_position(symbol, abs(change))
```

---

This manual provides a comprehensive guide to portfolio management across all asset classes. For specific asset class details, refer to the respective manuals (e.g., options_manual.md for options-specific features). 