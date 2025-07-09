# Options Greeks Manual

## Overview

This manual documents the comprehensive options Greeks support in the Quantify Options Framework. The system provides real-time Greeks data, theoretical calculations, data validation, and advanced analytics for options trading through a unified broker interface.

### Broker Support

All Greeks functionality is accessed through the abstract `BaseBroker` interface:

- **✅ Saxo Bank**: Full Greeks support with real-time data
- **⚠️ Other Brokers**: May have limited or no options support

All examples use the standard broker interface (`broker.method_name()`) for maximum compatibility.

## Table of Contents

1. [Greeks Data Sources](#greeks-data-sources)
2. [Core Components](#core-components)
3. [API Reference](#api-reference)
4. [Usage Examples](#usage-examples)
5. [Data Validation](#data-validation)
6. [Performance Optimization](#performance-optimization)
7. [Error Handling](#error-handling)
8. [Best Practices](#best-practices)

## Greeks Data Sources

The system supports multiple data sources in priority order:

### 1. Live Saxo API Data (Primary)
- **Source**: Real-time Greeks from Saxo Bank's API
- **Coverage**: Delta, Gamma, Theta, Vega, Rho, Implied Volatility
- **Update Frequency**: Real-time via WebSocket subscriptions
- **Reliability**: Highest accuracy for actively traded options

### 2. Theoretical Calculation (Fallback)
- **Source**: Black-Scholes model implementation
- **Coverage**: All standard Greeks
- **Use Case**: When broker data unavailable or incomplete
- **Parameters**: Configurable volatility and risk-free rate

### 3. Cached Data (Performance)
- **Source**: Previously fetched data with timestamps
- **Validity**: 5-minute cache for options chains, 1-minute for individual Greeks
- **Purpose**: Performance optimization and rate limiting

## Core Components

### Greeks Class

The `Greeks` dataclass provides comprehensive Greeks representation:

```python
@dataclass
class Greeks:
    delta: float = 0.0           # Price sensitivity to underlying
    gamma: float = 0.0           # Delta sensitivity to underlying
    theta: float = 0.0           # Time decay
    vega: float = 0.0            # Volatility sensitivity
    rho: float = 0.0             # Interest rate sensitivity
    implied_volatility: float = 0.0  # Market-implied volatility
```

#### Key Methods:
- `validate_greeks_data()` - Comprehensive validation with quality scoring
- `get_risk_metrics()` - Risk level assessment for each Greek
- `compare_with(other)` - Compare Greeks between different time points
- `is_reasonable()` - Quick reasonableness check

### SaxoBroker Greeks Methods

#### `get_option_greeks(option_symbol: str)`
Fetches comprehensive Greeks data for a single option.

**Returns:**
```python
{
    'delta': 0.5234,
    'gamma': 0.0156,
    'theta': -0.0234,
    'vega': 0.1234,
    'rho': 0.0567,
    'implied_volatility': 0.2345,
    'theoretical_price': 12.45,
    'intrinsic_value': 8.50,
    'time_value': 3.95,
    'bid': 12.40,
    'ask': 12.50,
    'mid': 12.45,
    'last': 12.43,
    'volume': 1250,
    'timestamp': datetime.now(pytz.utc)
}
```

#### `get_option_chain_analysis(underlying: str, expiry: datetime)`
Performs comprehensive analysis on the entire options chain.

**Returns:**
```python
{
    'underlying_symbol': 'AAPL',
    'underlying_price': 150.25,
    'expiry_date': '2024-01-19',
    'days_to_expiry': 45,
    'total_options': 40,
    'calls_count': 20,
    'puts_count': 20,
    'total_call_volume': 15000,
    'total_put_volume': 12000,
    'put_call_ratio_volume': 0.8,
    'calls_analysis': {
        'total_delta': 8.5,
        'total_gamma': 1.2,
        'total_theta': -2.5,
        'total_vega': 15.8,
        'max_volume_strike': 150.0,
        'itm_count': 8,
        'otm_count': 12
    },
    'puts_analysis': {
        'total_delta': -6.8,
        'total_gamma': 1.1,
        'total_theta': -2.2,
        'total_vega': 14.2,
        'max_volume_strike': 145.0,
        'itm_count': 5,
        'otm_count': 15
    },
    'volatility_analysis': {
        'atm_iv': 0.25,
        'iv_skew': 0.05,
        'call_iv_range': {'min': 0.22, 'max': 0.28},
        'put_iv_range': {'min': 0.24, 'max': 0.30}
    }
}
```

### SaxoOptions Enhanced Methods

#### `get_enriched_options_chain(underlying: str, expiry: datetime, include_live_greeks: bool = True)`
Fetches options chain with live Greeks for each option.

#### `get_option_greeks_bulk(option_symbols: List[str])`
Efficiently fetches Greeks for multiple options in parallel.

#### `calculate_strategy_metrics(strategy: BaseOptionsStrategy, spot_price: float)`
Calculates real-time strategy metrics using actual Greeks data.

## Usage Examples

### Basic Greeks Retrieval

```python
# Get Greeks for a single option (works with any broker that supports options)
broker = await setup_broker()  # SaxoBroker, etc.
greeks_data = await broker.get_option_greeks("12345")

print(f"Delta: {greeks_data['delta']:.4f}")
print(f"Gamma: {greeks_data['gamma']:.4f}")
print(f"Theta: {greeks_data['theta']:.4f}")
print(f"Vega: {greeks_data['vega']:.4f}")
print(f"IV: {greeks_data['implied_volatility']:.2%}")
```

### Options Chain Analysis

```python
from datetime import datetime, timedelta

# Analyze entire options chain
expiry = datetime.now() + timedelta(days=30)
analysis = await broker.get_option_chain_analysis("AAPL", expiry)

print(f"Put/Call Volume Ratio: {analysis['put_call_ratio_volume']:.2f}")
print(f"ATM Implied Volatility: {analysis['volatility_analysis']['atm_iv']:.2%}")
print(f"IV Skew: {analysis['volatility_analysis']['iv_skew']:.2%}")
```

### Enriched Options Chain

```python
# Get options chain with live Greeks
enriched_chain = await broker.get_enriched_options_chain("AAPL", expiry)

for call in enriched_chain.calls:
    greeks = call.greeks
    print(f"Strike {call.strike}: Δ={greeks.delta:.3f}, Γ={greeks.gamma:.3f}, "
          f"Θ={greeks.theta:.3f}, ν={greeks.vega:.3f}, IV={greeks.implied_volatility:.1%}")
```

### Strategy Greeks Analysis

```python
# Create and analyze a strategy
strategy = await broker.create_option_strategy(
    "IRON_CONDOR", 
    "AAPL", 
    expiry,
    buy_call_strike=160,
    sell_call_strike=155,
    sell_put_strike=145,
    buy_put_strike=140,
    quantity=10
)

# Calculate real-time metrics
spot_price = await broker.get_current_price("AAPL")
metrics = await broker.calculate_strategy_metrics(strategy, spot_price)

print(f"Strategy Delta: {metrics['total_delta']:.4f}")
print(f"Strategy Gamma: {metrics['total_gamma']:.4f}")
print(f"Strategy Theta: {metrics['total_theta']:.4f}")
print(f"Strategy Vega: {metrics['total_vega']:.4f}")
```

### Bulk Greeks Fetching

```python
# Get Greeks for multiple options efficiently
option_symbols = ["12345", "12346", "12347", "12348"]
bulk_greeks = await broker.get_option_greeks_bulk(option_symbols)

for symbol, greeks_data in bulk_greeks.items():
    if greeks_data:
        print(f"Option {symbol}: Δ={greeks_data['delta']:.3f}, "
              f"IV={greeks_data['implied_volatility']:.1%}")
```

## Data Validation

### Greeks Validation

The system includes comprehensive validation for Greeks data quality:

```python
# Validate Greeks data
greeks = Greeks(delta=0.5, gamma=0.02, theta=-0.01, vega=0.1, rho=0.05, implied_volatility=0.25)
validation = greeks.validate_greeks_data()

print(f"Valid: {validation['is_valid']}")
print(f"Quality Score: {validation['quality_score']}/100")
print(f"Warnings: {validation['warnings']}")
print(f"Errors: {validation['errors']}")

# Quick reasonableness check
if greeks.is_reasonable():
    print("✅ Greeks data looks reasonable")
else:
    print("⚠️ Greeks data may have issues")
```

### Validation Rules

The system checks for:

1. **Delta Range**: Must be between -1 and 1
2. **Gamma Sign**: Should be positive for long options
3. **Theta Sign**: Usually negative for long options
4. **Vega Sign**: Should be positive
5. **Implied Volatility**: Must be positive, reasonable range 2%-500%
6. **Cross-validation**: Gamma/Delta ratio for near-expiry detection

### Quality Scoring

- **100 points**: Perfect data with no issues
- **-15 points**: Per warning (unusual but valid values)
- **0 points**: Any validation errors

## Performance Optimization

### Caching Strategy

```python
# Options chain caching (5 minutes)
cache_key = f"{underlying}_{expiry.strftime('%Y-%m-%d')}"
if cache_key in self._options_chain_cache:
    cached_data = self._options_chain_cache[cache_key]
    if (datetime.now() - cached_data['timestamp']).total_seconds() < 300:
        return cached_data['data']

# Enriched chain caching
enriched_cache_key = f"{cache_key}_enriched"
```

### Parallel Processing

```python
# Bulk Greeks fetching uses asyncio.gather for parallel requests
tasks = [self.broker.get_option_greeks(symbol) for symbol in option_symbols]
greeks_results = await asyncio.gather(*tasks, return_exceptions=True)
```

### Rate Limiting

- **Options Chain**: 5-minute cache
- **Individual Greeks**: 1-minute cache
- **Bulk Requests**: Parallel processing with exception handling

## Error Handling

### Graceful Degradation

1. **Primary Source Fails**: Automatically falls back to theoretical calculation
2. **Partial Data**: Uses available Greeks, calculates missing ones
3. **Network Issues**: Returns cached data with staleness warnings
4. **Invalid Data**: Logs warnings, provides fallback values

### Exception Handling

```python
try:
    greeks_data = await broker.get_option_greeks(option_symbol)
except Exception as e:
    logger.error(f"Failed to get Greeks for {option_symbol}: {e}")
    # Fallback to theoretical calculation
    greeks_data = await broker._calculate_theoretical_greeks(option_symbol)
```

### Logging

The system provides comprehensive logging:

- **DEBUG**: Individual Greeks extraction details
- **INFO**: Successful operations and cache hits
- **WARNING**: Data quality issues and fallbacks
- **ERROR**: Failed operations with full stack traces

## Best Practices

### 1. Data Freshness

```python
# Always check timestamp for critical decisions
if greeks_data.get('timestamp'):
    age = (datetime.now(pytz.utc) - greeks_data['timestamp']).total_seconds()
    if age > 60:  # 1 minute
        logger.warning(f"Greeks data is {age:.0f} seconds old")
```

### 2. Validation

```python
# Always validate Greeks before using in calculations
greeks = Greeks.from_dict(greeks_data)
if not greeks.is_reasonable():
    logger.warning("Greeks data failed reasonableness check")
    validation = greeks.validate_greeks_data()
    logger.info(f"Validation details: {validation}")
```

### 3. Error Handling

```python
# Always have fallback strategies
try:
    live_greeks = await get_live_greeks(symbol)
except Exception:
    theoretical_greeks = await calculate_theoretical_greeks(symbol)
    logger.info("Using theoretical Greeks as fallback")
```

### 4. Performance

```python
# Use bulk operations for multiple options
if len(option_symbols) > 5:
    bulk_greeks = await get_option_greeks_bulk(option_symbols)
else:
    individual_greeks = [await get_option_greeks(sym) for sym in option_symbols]
```

### 5. Strategy Analysis

```python
# Use enriched chains for comprehensive strategy analysis
enriched_chain = await get_enriched_options_chain(underlying, expiry)
strategy_metrics = await calculate_strategy_metrics(strategy, spot_price)

# Validate strategy Greeks
total_delta = strategy_metrics['total_delta']
if abs(total_delta) > 1.0:
    logger.warning(f"Strategy has high delta exposure: {total_delta:.2f}")
```

### 6. Risk Management

```python
# Monitor Greeks risk levels
risk_metrics = greeks.get_risk_metrics()
for greek_name, risk_info in risk_metrics.items():
    if risk_info['level'] == 'high':
        logger.warning(f"High {greek_name}: {risk_info['value']:.3f}")
```

## Troubleshooting

### Common Issues

1. **No Greeks Data Available**
   - Check if option is actively traded
   - Verify option symbol format
   - Use theoretical calculation as fallback

2. **Stale Data**
   - Check network connectivity
   - Verify API credentials
   - Clear cache if necessary

3. **Invalid Greeks Values**
   - Run validation to identify specific issues
   - Check if option is near expiry
   - Verify underlying price data

4. **Performance Issues**
   - Use bulk operations for multiple options
   - Implement proper caching
   - Monitor API rate limits

### Debug Commands

```python
# Enable debug logging
import logging
logging.getLogger('SaxoOptions').setLevel(logging.DEBUG)

# Check cache status
print(f"Cache size: {len(saxo_options._options_chain_cache)}")

# Validate individual Greeks
validation = greeks.validate_greeks_data()
print(f"Quality score: {validation['quality_score']}")
```

## Conclusion

The enhanced options Greeks support provides professional-grade functionality for options trading, including:

- ✅ Real-time Greeks from Saxo Bank API
- ✅ Theoretical fallback calculations
- ✅ Comprehensive data validation
- ✅ Performance optimization
- ✅ Advanced analytics and risk metrics
- ✅ Robust error handling

This implementation ensures reliable Greeks data for all options trading operations, from individual option analysis to complex multi-leg strategy evaluation. 