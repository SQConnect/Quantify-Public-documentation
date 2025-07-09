# The Quantify Options Trading Framework

This document provides a comprehensive guide to using the options trading module within the Quantify framework. The framework has been completely redesigned for production use with multiple brokers and strategies.

## 1. Overview

The options framework provides a robust, production-ready system for defining, executing, and managing both single and multi-leg options strategies. It abstracts away broker-specific complexities while maintaining full control over strategy implementation.

### 🚀 **Key Features**
- ✅ **25+ Strategy Factory**: Pre-built implementations of all major options strategies
- ✅ **Multi-Broker Support**: Works seamlessly with Saxo, Interactive Brokers, and more
- ✅ **Multi-Asset Options**: Support for stock options, index options, and futures options
- ✅ **Real-time Greeks**: Live Greeks calculations and monitoring with data validation
- ✅ **Position Management**: Advanced position tracking and risk management
- ✅ **Strategy Builder**: Flexible API for custom strategy creation
- ✅ **Greeks Analysis**: Comprehensive Greeks data with quality scoring and risk assessment



### **Framework Usage Patterns**

1. **Strategy Factory Approach** (Recommended): Use pre-built strategy implementations from the comprehensive factory
2. **Programmatic Strategy Execution**: Create specific strategies on command for discretionary trading
3. **Event-Driven Strategies**: Build fully automated strategies that react to real-time market data

## 2. Core Components

### `OptionLeg`
The fundamental building block representing a single option contract.

- **File:** `src/options/option_leg.py`
- **Key Attributes:**
  - `symbol`: Broker-specific option identifier (e.g., UIC for Saxo)
  - `strike`: Strike price as `Decimal`
  - `option_type`: `OptionType.CALL` or `OptionType.PUT`
  - `expiry`: Expiration `datetime`
  - `greeks`: Live `Greeks` object with real-time values
  - `bid`, `ask`, `mid`: Current market prices
  - `asset_type`: Type of option (StockOption, StockIndexOption, FuturesOption)
  - `underlying_type`: Type of underlying asset

### `OptionChain`
Container for all available options for a specific underlying and expiry date.

- **File:** `src/options/chain.py`
- **Key Methods:**
  - `add_leg(leg)`: Adds an `OptionLeg` to the chain
  - `get_leg_by_strike(strike, option_type)`: Finds specific option
  - `sort_legs()`: Organizes calls and puts by strike
  - `filter_by_delta(min_delta, max_delta)`: Filter options by Greeks
  - `filter_by_asset_type(asset_type)`: Filter by option type

### `OptionsHelper` (Broker Interface)
The primary interface for all broker options functionality.

- **File:** `src/options/helper.py`
- **Key Methods:**
  - `get_options_chain(underlying, expiry)`: Fetches live option chains
  - `place_option_order(leg, side, quantity, order_type, price)`: Single leg orders
  - `place_multi_leg_order(legs, order_type, price)`: Complex strategy orders
  - `subscribe_to_greeks(leg)`: Real-time Greeks monitoring
  - `get_strategy_price(legs)`: Multi-leg pricing

### `Greeks`
Real-time Greek values for options with comprehensive validation and analysis.

- **File:** `src/options/greeks.py`
- **Attributes:** `delta`, `gamma`, `theta`, `vega`, `rho`, `implied_volatility`
- **Key Features:**
  - Real-time Greeks from broker APIs
  - Theoretical Greeks calculation fallback
  - Data quality validation and scoring
  - Risk metrics and anomaly detection

**📖 For detailed Greeks documentation, see:**
- **[Options Greeks Manual](options_greeks_manual.md)** - Complete guide to Greeks functionality
- **[Greeks API Reference](greeks_api_reference.md)** - Quick reference for all Greeks methods

### `BaseOptionsStrategy`
Abstract base class for all strategy implementations.

- **File:** `src/options/strategies/base_strategy.py`
- **Key Features:**
  - Strategy type classification and metadata
  - Leg definition and validation
  - Broker integration methods
  - Event handling capabilities

## 3. Order Types and Events

### Single Orders
```python
# Place a single option order
result = await broker.place_order(
    symbol="12345",  # Option UIC/symbol
    order_type="Limit",  # Market, Limit, Stop, StopLimit, TrailingStop
    side="Buy",
    quantity=1,
    price=2.50,  # Required for Limit orders
    time_in_force="GTC",  # GTC, DAY, FOK, IOC
    to_open_close="ToOpen",  # ToOpen or ToClose
    validation_type="Full"  # None, Light, Full
)
```

### Multi-Leg Orders
```python
# Place a multi-leg option order
legs = [
    {
        "instrument_id": "12345",
        "side": "Buy",
        "quantity": 1,
        "to_open_close": "ToOpen",
        "price": 2.50  # Optional leg-specific price
    },
    {
        "instrument_id": "67890",
        "side": "Sell",
        "quantity": 1,
        "to_open_close": "ToOpen"
    }
]

result = await broker.place_multi_leg_order(
    legs=legs,
    order_type="Limit",
    price=1.25,  # Net debit/credit
    strategy_type="BULL_CALL_SPREAD",
    validation_type="Full",
    order_duration={"DurationType": "DayOrder"}
)
```

### Order Events
The framework provides comprehensive order event handling:

```python
async def handle_order_event(event_data):
    """
    Handle various order events:
    - OrderPlaced: Initial order placement
    - OrderActivated: Order becomes active
    - OrderUpdated: Order details updated
    - OrderCancelled: Order cancelled
    - OrderExpired: Order expired
    - OrderFilled: Partial or full fills
    - OrderRejected: Order rejections
    """
    event_type = event_data['event_type']
    order_id = event_data['order_id']
    status = event_data['status']
    
    if event_type == 'OrderFilled':
        execution_price = event_data['execution_price']
        is_partial = event_data['is_partial']
        # Handle fill
    elif event_type == 'OrderRejected':
        rejection_reason = event_data['rejection_reason']
        # Handle rejection
```

## 4. Strategy Factory (Recommended Approach)

The framework includes a comprehensive factory with 25+ pre-built strategy implementations. **This is the recommended approach for most use cases.**

### Available Strategies

#### **Vertical Spreads**
- `BullCallSpread`, `BearCallSpread`
- `BullPutSpread`, `BearPutSpread`
- `CreditSpread`, `DebitSpread`

#### **Volatility Strategies**
- `LongStraddle`, `ShortStraddle`
- `LongStrangle`, `ShortStrangle`
- `IronCondor`, `IronButterfly`
- `VegaPositive`, `VegaNegative`

#### **Calendar & Time Spreads**
- `CalendarSpread`, `CalendarCall`, `CalendarPut`
- `DiagonalSpread`, `DiagonalCall`, `DiagonalPut`
- `DoubleCalendar`, `DiagonalButterfly`

#### **Complex Strategies**
- `ButterflySpread`, `ButterflyCall`, `ButterflyPut`
- `CondorSpread`, `CondorCall`, `CondorPut`
- `BackRatio`, `FrontRatio`
- `JadeeLizard`, `BigLizard`
- `Synthetic`, `Conversion`, `Reversal`

### Factory Usage

```python
from src.broker_interface.broker_factory import BrokerFactory
from src.core.config.config_manager import ConfigManager
from datetime import datetime

# Setup broker
config_manager = ConfigManager()
saxo_config = config_manager.get_broker_config('saxo')
broker = BrokerFactory.create_broker('saxo', saxo_config)
await broker.connect()

# Create Bull Call Spread using broker's options functionality
strategy = await broker.create_option_strategy(
    strategy_type="BullCallSpread",
    underlying="NVDA:xnas",
    expiry=datetime(2024, 12, 20),
    long_strike=800.0,
    short_strike=850.0,
    quantity=1
)

# Strategy is now a proper BullCallSpreadStrategy object
print(f"Strategy type: {strategy.strategy_type}")
print(f"Max profit: {strategy.max_profit}")
print(f"Max loss: {strategy.max_loss}")
print(f"Break-even: {strategy.break_even_points}")

# Execute the strategy
if hasattr(strategy, 'legs') and strategy.legs:
    order_results = await broker.place_strategy_orders(strategy)
    print(f"Orders placed: {order_results}")
```

### Advanced Strategy Configuration

```python
# Iron Condor with specific parameters
iron_condor = await broker.create_option_strategy(
    strategy_type="IronCondor",
    underlying="SPY:arcx",
    expiry=datetime(2024, 12, 20),
    # Iron Condor specific parameters
    put_wing_width=10.0,
    call_wing_width=10.0,
    body_width=20.0,
    quantity=5,
    target_delta=0.15
)

# Calendar Spread
calendar = await broker.create_option_strategy(
    strategy_type="CalendarSpread",
    underlying="AAPL:xnas",
    short_expiry=datetime(2024, 11, 15),
    long_expiry=datetime(2024, 12, 20),
    strike=150.0,
    quantity=2
)
```

## 5. Working with Different Option Types

The framework supports multiple types of options:

### Stock Options
```python
# Regular equity options
stock_chain = await broker.get_options_chain(
    underlying="AAPL:xnas",
    expiry=datetime(2024, 12, 20)
)
```

### Index Options
```python
# Index options (e.g., SPX)
index_chain = await broker.get_options_chain(
    underlying="SPX:indexs",
    expiry=datetime(2024, 12, 20)
)
```

### Futures Options
```python
# Options on futures
futures_chain = await broker.get_options_chain(
    underlying="ESZ4:cme",  # December 2024 E-mini S&P 500
    expiry=datetime(2024, 12, 20)
)
```

### Asset Type Detection
The framework automatically detects the appropriate option type based on the underlying:
- Stock underlying → StockOption
- Index underlying → StockIndexOption
- Future underlying → FuturesOption

## 6. Order Management

### Order Status Tracking
```python
# Get current status of an order
status = await broker.get_order_status(order_id)
print(f"Status: {status['status']}")
print(f"Filled Amount: {status['filled_amount']}")
print(f"Remaining: {status['remaining_amount']}")
```

### Order Event Handling
```python
from src.core.events import EventType, OrderEvent

async def handle_order_events(event: OrderEvent):
    """Handle various order events"""
    if event.event_type == 'OrderFilled':
        # Handle fill
        filled_amount = event.data['filled_amount']
        execution_price = event.data['execution_price']
        is_partial = event.data['is_partial']
        
    elif event.event_type == 'OrderRejected':
        # Handle rejection
        reason = event.data['rejection_reason']
        error_code = event.data['error_code']
        
    elif event.event_type == 'OrderCancelled':
        # Handle cancellation
        reason = event.data['cancellation_reason']

# Subscribe to order events
event_manager.subscribe(EventType.ORDER, handle_order_events)
```

## 7. Error Handling and Validation

The framework includes comprehensive error handling and validation:

### Order Validation
```python
# Validate order parameters
try:
    result = await broker.place_order(
        symbol="12345",
    order_type="Limit",
        side="Buy",
        quantity=1,
        price=2.50,
        validation_type="Full"  # Performs full validation
)
except OrderError as e:
    print(f"Order validation failed: {e}")
```

### Strategy Validation
```python
# Validate strategy parameters
try:
    strategy = await strategy_factory.create_option_strategy(
        strategy_type="IronCondor",
    underlying="SPY:arcx",
        expiry=datetime(2024, 12, 20),
        put_wing_width=10.0,
        call_wing_width=10.0,
        body_width=20.0
    )
except ValueError as e:
    print(f"Strategy validation failed: {e}")
```

## 8. Greeks Analysis and Risk Management

The framework provides comprehensive Greeks analysis with real-time monitoring and data validation.

**📋 Broker Support Note**: Greeks functionality is fully supported by Saxo Bank broker. Other brokers may have limited or no options support. All examples below use the abstract broker interface and will work with any broker that supports options.

### Getting Greeks Data

```python
# Get live Greeks for a specific option
greeks_data = await broker.get_option_greeks("12345")  # UIC/symbol
print(f"Delta: {greeks_data['delta']:.4f}")
print(f"Gamma: {greeks_data['gamma']:.4f}")
print(f"Theta: {greeks_data['theta']:.4f}")
print(f"Vega: {greeks_data['vega']:.4f}")
print(f"Implied Volatility: {greeks_data['implied_volatility']:.2%}")
```

### Options Chain Analysis

```python
# Get comprehensive options chain analysis
analysis = await broker.get_option_chain_analysis("AAPL:xnas", expiry_date)

# Volatility smile analysis
smile = analysis['volatility_smile']
print(f"IV Skew: {smile['iv_skew']:.4f}")
print(f"Smile Convexity: {smile['convexity']:.4f}")

# Put/Call ratios
ratios = analysis['put_call_ratios']
print(f"Volume Ratio: {ratios['volume_ratio']:.2f}")
print(f"Open Interest Ratio: {ratios['open_interest_ratio']:.2f}")
```

### Strategy Greeks Monitoring

```python
# Calculate real-time strategy Greeks
metrics = await broker.calculate_strategy_metrics(strategy, spot_price)

print(f"Portfolio Delta: {metrics['total_delta']:.4f}")
print(f"Portfolio Gamma: {metrics['total_gamma']:.4f}")
print(f"Portfolio Theta: {metrics['total_theta']:.4f}")
print(f"Portfolio Vega: {metrics['total_vega']:.4f}")

# Risk assessment
print(f"Delta Neutral: {metrics['delta_neutral']}")
print(f"Gamma Risk: {metrics['gamma_risk']}")
print(f"Vega Exposure: {metrics['vega_exposure']:.2f}")
```

**📖 For complete Greeks documentation:**
- **[Options Greeks Manual](options_greeks_manual.md)** - Comprehensive guide with examples
- **[Greeks API Reference](greeks_api_reference.md)** - Quick reference for all methods

## 9. Best Practices

1. **Always Use Proper Asset Types**
   - Let the framework detect the appropriate option type
   - Don't mix different option types in the same strategy

2. **Handle Order Events**
   - Subscribe to order events for real-time updates
   - Implement proper error handling
   - Track order status changes

3. **Use Strategy Factory**
   - Prefer factory methods over manual strategy creation
   - Leverage built-in validation and error handling
   - Use proper strategy types for better organization

4. **Manage Order Lifecycle**
   - Track order status through events
   - Handle partial fills appropriately
   - Clean up completed orders

5. **Proper Error Handling**
   - Implement try-catch blocks
   - Log errors appropriately
   - Handle rejections and cancellations

6. **Strategy Management**
   - Monitor positions with real-time Greeks
   - Track Greeks changes and risk metrics
   - Implement proper risk management with Greeks validation

7. **Greeks Data Quality**
   - Validate Greeks data quality scores
   - Use theoretical calculation fallbacks when needed
   - Monitor data anomalies and outliers

8. **Testing**
   - Test strategies in simulation first
   - Validate order parameters
   - Check strategy configurations

The Quantify Options Framework provides a comprehensive, production-ready solution for options trading across multiple brokers. With its extensive strategy factory, real-time data integration, and robust error handling, it supports everything from simple directional trades to complex multi-leg strategies.

## 10. Integration Examples

### Full Strategy Lifecycle with Greeks Monitoring

```python
async def complete_options_workflow():
    """Complete workflow from strategy creation to management with Greeks monitoring"""
    
    # 1. Setup
    broker = await setup_broker()
    
    # 2. Create strategy using broker
    strategy = await broker.create_option_strategy(
        strategy_type="IronCondor",
        underlying="SPY:arcx",
        expiry=datetime(2024, 12, 20),
        quantity=10
    )
    
    # 3. Analyze Greeks before execution
    spot_price = await broker.get_current_price("SPY:arcx")
    metrics = await broker.calculate_strategy_metrics(strategy, spot_price)
    
    print(f"Strategy Greeks - Delta: {metrics['total_delta']:.4f}, "
          f"Gamma: {metrics['total_gamma']:.4f}, "
          f"Theta: {metrics['total_theta']:.4f}")
    
    # 4. Execute strategy
    orders = await broker.place_strategy_orders(strategy)
    
    # 5. Monitor and manage with Greeks
    tracker = StrategyPerformanceTracker()
    await tracker.track_strategy("iron_condor_1", strategy)
    
    # 6. Real-time monitoring with Greeks risk management
    while strategy.is_active:
        await tracker.update_performance("iron_condor_1")
        
        # Update Greeks and risk metrics
        current_spot = await broker.get_current_price("SPY:arcx")
        current_metrics = await broker.calculate_strategy_metrics(strategy, current_spot)
        
        # Greeks-based risk management
        if abs(current_metrics['total_delta']) > 0.5:  # Delta risk
            print("WARNING: High delta exposure detected")
        
        if current_metrics['gamma_risk']:  # Gamma risk
            print("WARNING: High gamma risk detected")
        
        if current_metrics['theta_decay_per_day'] < -100:  # Theta decay
            print("WARNING: High theta decay")
        
        # Check exit conditions
        performance = tracker.get_performance_report("iron_condor_1")
        if performance['current_pnl'] > 500:  # Profit target
            await strategy.close_position()
            break
            
        await asyncio.sleep(300)  # Check every 5 minutes
```

### Advanced Greeks Analysis Workflow

```python
async def advanced_greeks_analysis():
    """Advanced Greeks analysis and option chain evaluation"""
    
    # 1. Get enriched options chain with live Greeks
    enriched_chain = await broker.get_enriched_options_chain(
        underlying="AAPL:xnas",
        expiry=datetime(2024, 12, 20),
        include_live_greeks=True
    )
    
    # 2. Comprehensive chain analysis
    analysis = await broker.get_option_chain_analysis("AAPL:xnas", expiry_date)
    
    # 3. Greeks quality validation
    from src.options.greeks import Greeks
    
    for call in enriched_chain.calls:
        validation_result = Greeks.validate_greeks_data(call.greeks)
        quality_score = validation_result['quality_score']
        
        if quality_score < 70:
            print(f"Low quality Greeks for {call.symbol}: {quality_score}")
        
        # Risk assessment
        risk_metrics = call.greeks.get_risk_metrics()
        if risk_metrics['overall_risk'] == 'HIGH':
            print(f"High risk option detected: {call.symbol}")
    
    # 4. Volatility smile analysis
    smile = analysis['volatility_smile']
    if abs(smile['iv_skew']) > 0.1:
        print(f"Significant IV skew detected: {smile['iv_skew']:.4f}")
    
    # 5. Strategy optimization based on Greeks
    optimal_strikes = []
    for call in enriched_chain.calls:
        if (0.3 <= call.greeks.delta <= 0.7 and  # Good delta range
            call.greeks.gamma > 0.01 and           # Sufficient gamma
            call.greeks.implied_volatility > 0.15): # Reasonable IV
            optimal_strikes.append(call.strike)
    
    print(f"Optimal strikes for strategy: {optimal_strikes}")
```

This framework enables sophisticated options trading with institutional-grade Greeks analysis, comprehensive risk management, and real-time monitoring capabilities.

## 11. Additional Documentation

### Greeks Documentation
For comprehensive Greeks functionality documentation:

- **[Options Greeks Manual](options_greeks_manual.md)** - Complete guide covering:
  - Overview of Greeks data sources and calculation methods
  - Core components: Greeks class, validation, and risk metrics
  - Usage examples for all major functions
  - Data quality validation and scoring system
  - Performance optimization strategies
  - Error handling and troubleshooting
  - Best practices and common use cases

- **[Greeks API Reference](greeks_api_reference.md)** - Quick reference guide with:
  - Core classes and their methods
  - Method signatures and return types
  - Code examples for common operations
  - Error handling patterns
  - Performance tips and configuration options

### Framework Architecture
The options framework is built on a modular architecture:

```
src/options/
├── greeks.py              # Greeks calculation and validation
├── option_leg.py          # Individual option contracts
├── chain.py               # Options chain management
├── helper.py              # Broker integration
├── pricing.py             # Pricing models (Black-Scholes, etc.)
└── strategies/            # Strategy implementations
    ├── base_strategy.py   # Abstract strategy base
    ├── vertical_spread.py # Vertical spread strategies
    ├── straddle.py        # Straddle strategies
    └── ...                # Other strategy types
```

### Key Integration Points
- **Broker Interface**: `SaxoOptions` class provides broker-specific implementation
- **Strategy Factory**: Creates and manages strategy objects
- **Greeks Engine**: Real-time Greeks calculation and monitoring
- **Risk Management**: Greeks-based risk assessment and alerts
- **Data Validation**: Quality scoring and anomaly detection

## 12. Broker Integration

### 12.1 Saxo Bank Integration

The framework provides comprehensive integration with Saxo Bank's OpenAPI, supporting:

#### Authentication & Security
- OAuth2 with refresh token management
- Token expiry monitoring and auto-refresh
- Coming soon: Certificate-based authentication
- Coming soon: PKCE flow support

#### Order Management
- Single and multi-leg orders
- Market, limit, stop, and trailing orders
- Order conditions and relations
- Coming soon: Algo orders support
- Coming soon: Enhanced pre-trade validation

#### Market Data
- Real-time streaming quotes
- Options chain subscriptions
- Greeks monitoring
- Coming soon: Delayed data support
- Coming soon: Order book depth
- Coming soon: Trade message handling

#### Position & Risk Management
- Position tracking and netting
- Margin monitoring
- Account value calculations
- Coming soon: Enhanced position netting
- Coming soon: Pre-trade margin checks
- Coming soon: Risk limit enforcement

#### Corporate Actions
- Coming soon: Split handling
- Coming soon: Merger processing
- Coming soon: Account activity monitoring
- Coming soon: Corporate action adjustments

## 13. Strategy Implementation

### 13.1 Single-Leg Strategies
...
