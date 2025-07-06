# Greeks API Quick Reference

## Core Classes

### Greeks
```python
from src.options.greeks import Greeks

# Create Greeks object
greeks = Greeks(
    delta=0.5234,
    gamma=0.0156,
    theta=-0.0234,
    vega=0.1234,
    rho=0.0567,
    implied_volatility=0.2345
)

# Validation
validation = greeks.validate_greeks_data()
is_reasonable = greeks.is_reasonable()
risk_metrics = greeks.get_risk_metrics()

# Comparison
other_greeks = Greeks(delta=0.5100, ...)
comparison = greeks.compare_with(other_greeks)
```

## SaxoBroker Methods

### get_option_greeks()
```python
# Single option Greeks
greeks_data = await saxo_broker.get_option_greeks("12345")

# Returns:
{
    'delta': float,
    'gamma': float,
    'theta': float,
    'vega': float,
    'rho': float,
    'implied_volatility': float,
    'theoretical_price': float,
    'intrinsic_value': float,
    'time_value': float,
    'bid': float,
    'ask': float,
    'mid': float,
    'last': float,
    'volume': int,
    'timestamp': datetime,
    'theoretical_calculation': bool  # True if fallback used
}
```

### get_option_chain_analysis()
```python
from datetime import datetime, timedelta

expiry = datetime.now() + timedelta(days=30)
analysis = await saxo_broker.get_option_chain_analysis("AAPL", expiry)

# Returns comprehensive chain analysis with:
# - Volume and open interest metrics
# - Put/call ratios
# - Aggregate Greeks by side
# - Volatility smile analysis
# - ATM strikes and moneyness distribution
```

## SaxoOptions Methods

### get_enriched_options_chain()
```python
# Get options chain with live Greeks for all strikes
enriched_chain = await saxo_options.get_enriched_options_chain(
    underlying="AAPL",
    expiry=expiry,
    include_live_greeks=True
)

# Access individual option Greeks
for call in enriched_chain.calls:
    print(f"Strike {call.strike}: Δ={call.greeks.delta:.3f}")
```

### get_option_greeks_bulk()
```python
# Efficiently fetch Greeks for multiple options
option_symbols = ["12345", "12346", "12347"]
bulk_greeks = await saxo_options.get_option_greeks_bulk(option_symbols)

# Returns: Dict[str, Dict[str, Any]]
for symbol, greeks_data in bulk_greeks.items():
    if greeks_data:  # Check if data available
        print(f"Option {symbol}: Δ={greeks_data['delta']:.3f}")
```

### calculate_strategy_metrics()
```python
# Real-time strategy Greeks calculation
strategy = await saxo_options.create_option_strategy(...)
spot_price = 150.25

metrics = await saxo_options.calculate_strategy_metrics(strategy, spot_price)

# Returns:
{
    'strategy_type': str,
    'underlying': str,
    'underlying_price': float,
    'estimated_cost': float,
    'total_delta': float,
    'total_gamma': float,
    'total_theta': float,
    'total_vega': float,
    'total_rho': float,
    'number_of_legs': int,
    'max_profit': str,
    'max_loss': str,
    'break_even_points': List[float],
    'delta_neutral': bool,
    'gamma_risk': bool,
    'theta_decay_per_day': float,
    'vega_exposure': float,
    'leg_details': List[Dict],
    'calculation_timestamp': str
}
```

## Error Handling Patterns

### Basic Error Handling
```python
try:
    greeks_data = await saxo_broker.get_option_greeks(option_symbol)
except Exception as e:
    logger.error(f"Failed to get Greeks: {e}")
    # Fallback to cached or theoretical data
    greeks_data = {}
```

### Data Validation
```python
if greeks_data:
    greeks = Greeks.from_dict(greeks_data)
    if not greeks.is_reasonable():
        logger.warning("Greeks data quality issues detected")
        validation = greeks.validate_greeks_data()
        logger.info(f"Validation: {validation}")
```

### Staleness Check
```python
if 'timestamp' in greeks_data:
    age = (datetime.now(pytz.utc) - greeks_data['timestamp']).total_seconds()
    if age > 300:  # 5 minutes
        logger.warning(f"Greeks data is {age:.0f} seconds old")
```

## Performance Tips

### Use Bulk Operations
```python
# ✅ Good: Bulk fetch for multiple options
if len(option_symbols) > 3:
    bulk_greeks = await get_option_greeks_bulk(option_symbols)

# ❌ Avoid: Individual requests for many options
individual_requests = [await get_option_greeks(sym) for sym in option_symbols]
```

### Cache Awareness
```python
# Check cache before expensive operations
cache_key = f"{underlying}_{expiry.strftime('%Y-%m-%d')}"
if cache_key not in saxo_options._options_chain_cache:
    # Will fetch fresh data
    chain = await saxo_options.get_options_chain(underlying, expiry)
```

### Parallel Processing
```python
# Use asyncio.gather for parallel operations
tasks = [
    saxo_broker.get_option_greeks(symbol1),
    saxo_broker.get_option_greeks(symbol2),
    saxo_broker.get_option_chain_analysis(underlying, expiry)
]
results = await asyncio.gather(*tasks, return_exceptions=True)
```

## Common Use Cases

### Portfolio Greeks Monitoring
```python
async def monitor_portfolio_greeks(positions):
    """Monitor aggregate Greeks across portfolio"""
    total_delta = total_gamma = total_theta = total_vega = 0.0
    
    for position in positions:
        greeks_data = await saxo_broker.get_option_greeks(position.symbol)
        if greeks_data:
            multiplier = 1 if position.side == 'long' else -1
            quantity = position.quantity
            
            total_delta += greeks_data['delta'] * multiplier * quantity
            total_gamma += greeks_data['gamma'] * multiplier * quantity
            total_theta += greeks_data['theta'] * multiplier * quantity
            total_vega += greeks_data['vega'] * multiplier * quantity
    
    return {
        'total_delta': total_delta,
        'total_gamma': total_gamma,
        'total_theta': total_theta,
        'total_vega': total_vega
    }
```

### Volatility Surface Analysis
```python
async def analyze_volatility_surface(underlying, expiries):
    """Analyze implied volatility across strikes and expiries"""
    iv_surface = {}
    
    for expiry in expiries:
        analysis = await saxo_broker.get_option_chain_analysis(underlying, expiry)
        if analysis and 'volatility_analysis' in analysis:
            iv_surface[expiry.strftime('%Y-%m-%d')] = analysis['volatility_analysis']
    
    return iv_surface
```

### Risk Alert System
```python
async def check_greeks_risk_alerts(strategy, thresholds):
    """Check if strategy Greeks exceed risk thresholds"""
    metrics = await saxo_options.calculate_strategy_metrics(strategy, spot_price)
    alerts = []
    
    if abs(metrics['total_delta']) > thresholds['max_delta']:
        alerts.append(f"High delta exposure: {metrics['total_delta']:.2f}")
    
    if metrics['total_gamma'] > thresholds['max_gamma']:
        alerts.append(f"High gamma risk: {metrics['total_gamma']:.3f}")
    
    if abs(metrics['total_theta']) > thresholds['max_theta']:
        alerts.append(f"High theta decay: {metrics['total_theta']:.3f}")
    
    return alerts
```

## Configuration

### Theoretical Greeks Parameters
```python
# Default parameters used for theoretical calculations
DEFAULT_VOLATILITY = 0.25  # 25%
DEFAULT_RISK_FREE_RATE = 0.02  # 2%

# These are used when broker Greeks unavailable
# Can be customized based on market conditions
```

### Cache Settings
```python
# Cache timeouts
OPTIONS_CHAIN_CACHE_SECONDS = 300  # 5 minutes
INDIVIDUAL_GREEKS_CACHE_SECONDS = 60  # 1 minute
ENRICHED_CHAIN_CACHE_SECONDS = 300  # 5 minutes
```

### Logging Configuration
```python
import logging

# Enable detailed Greeks logging
logging.getLogger('SaxoOptions').setLevel(logging.DEBUG)
logging.getLogger('SaxoBroker').setLevel(logging.INFO)

# Log Greeks validation details
logging.getLogger('Greeks').setLevel(logging.DEBUG)
``` 