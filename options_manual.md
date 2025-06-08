# Options Trading Manual

## Table of Contents
1. [Introduction](#introduction)
2. [Key Concepts](#key-concepts)
3. [Data Structures](#data-structures)
4. [Option Chain Management](#option-chain-management)
5. [Strategy Analysis](#strategy-analysis)
6. [Rolling Strategies](#rolling-strategies)
7. [Implied Volatility Analysis](#implied-volatility-analysis)
8. [Best Practices](#best-practices)
9. [Examples](#examples)
10. [Troubleshooting & FAQ](#troubleshooting--faq)
11. [Integrating Options into the Events System](#integrating-options-into-the-events-system)

---

## Introduction

This manual explains how to work with options in the Quantify framework. The options system is designed to be broker-agnostic, allowing you to build, analyze, and execute option strategies across multiple brokers using a unified interface.

---

## Key Concepts

- **OptionChain**: Represents the full set of available options for an underlying asset and expiry.
- **OptionLeg**: Represents a single option contract (call or put, buy or sell) as part of a strategy.
- **OptionStrategy**: Represents a multi-leg options position (e.g., vertical spread, straddle, iron condor).
- **Greeks**: Risk metrics (delta, gamma, theta, vega, rho) for each leg and the overall strategy.
- **OptionsHelper**: A generic utility for building and analyzing strategies, independent of broker.
- **Implied Volatility**: The market's expectation of future volatility
- **Moneyness**: ITM (In-The-Money), ATM (At-The-Money), OTM (Out-of-The-Money)

---

## Data Structures

The framework uses several key data structures to represent and manage options data:

### OptionChain

The `OptionChain` class represents a complete options chain for a given underlying asset. It stores and manages both calls and puts for a specific expiration date.

```python
chain = OptionChain(
    symbol="AAPL",
    expiration=datetime(2024, 12, 20),
    spot_price=180.50,
    risk_free_rate=0.02
)
```

#### Key Components:

1. **Calls DataFrame**
   ```python
   chain.calls = pd.DataFrame(columns=[
       'symbol',          # Option symbol (e.g., 'AAPL231220C180000')
       'strike',          # Strike price
       'price',           # Last traded price
       'bid',            # Current bid price
       'ask',            # Current ask price
       'volume',         # Trading volume
       'open_interest',  # Open interest
       'implied_volatility',  # Calculated IV
       'delta',          # Option delta
       'gamma',          # Option gamma
       'theta',          # Option theta
       'vega',           # Option vega
       'rho'             # Option rho
   ])
   ```

2. **Puts DataFrame**
   ```python
   chain.puts = pd.DataFrame(columns=[
       # Same columns as calls
   ])
   ```

#### OptionChain Structure Breakdown

The `OptionChain` class is designed to provide a comprehensive view of options data for a specific underlying asset and expiration date. Here's a detailed breakdown of its structure and functionality:

1. **Core Attributes**
   ```python
   class OptionChain:
       def __init__(self, symbol: str, expiration: datetime, spot_price: float,
                    risk_free_rate: float = 0.02):
           self.symbol = symbol          # Underlying asset symbol
           self.expiration = expiration  # Option expiration date
           self.spot_price = spot_price  # Current price of underlying
           self.risk_free_rate = risk_free_rate  # Risk-free interest rate
   ```

2. **Data Organization**
   - **Calls and Puts**: Separate DataFrames for calls and puts
   - **Strike Prices**: Organized in ascending order
   - **Greeks**: Calculated for each option
   - **Market Data**: Includes bid/ask, volume, and open interest

3. **Data Access Methods**
   ```python
   # Moneyness-based access
   atm_options = chain.get_atm_options('call')  # At-the-money options
   itm_options = chain.get_itm_options('put')   # In-the-money options
   otm_options = chain.get_otm_options('call')  # Out-of-the-money options
   
   # Greeks-based access
   high_iv_options = chain.get_high_iv_options('call', min_iv_percentile=90)
   high_theta_options = chain.get_high_theta_options('put', min_theta_percentile=90)
   high_gamma_options = chain.get_high_gamma_options('call', min_gamma_percentile=90)
   high_vega_options = chain.get_high_vega_options('put', min_vega_percentile=90)
   ```

4. **Volatility Analysis Methods**
   ```python
   # Volatility patterns
   skew = chain.get_volatility_skew('call', delta_range=(0.2, 0.8))
   smile = chain.get_volatility_smile('put', moneyness_range=(-0.1, 0.1))
   term_structure = chain.get_term_structure('call', delta=0.5)
   surface = chain.get_implied_volatility_surface('put')
   ```

5. **Option Leg Creation**
   ```python
   leg = chain.get_option_leg(
       option_type='call',
       strike=180.00,
       quantity=1,
       action='buy'
   )
   ```

6. **Data Management**
   ```python
   # Adding new options
   chain.add_option(
       option_type='call',
       strike=180.00,
       price=5.50,
       bid=5.45,
       ask=5.55,
       volume=100,
       open_interest=500
   )
   
   # Serialization
   chain_dict = chain.to_dict()
   new_chain = OptionChain.from_dict(chain_dict)
   ```

7. **Data Validation and Error Handling**
   - Validates option data before adding to chain
   - Ensures consistent data types
   - Handles missing or invalid data
   - Maintains data integrity

8. **Performance Considerations**
   - Efficient data storage using pandas DataFrames
   - Optimized filtering and selection methods
   - Cached calculations for frequently accessed data
   - Memory-efficient data structures

9. **Integration Points**
   - Broker interface integration
   - Market data feed integration
   - Strategy builder integration
   - Risk management system integration

10. **Usage Examples**
    ```python
    # Create and populate chain
    chain = OptionChain("AAPL", expiration, spot_price)
    
    # Add options from broker data
    for option in broker_options:
        chain.add_option(**option)
    
    # Analyze volatility patterns
    skew = chain.get_volatility_skew('call')
    smile = chain.get_volatility_smile('put')
    
    # Find trading opportunities
    high_iv_calls = chain.get_high_iv_options('call')
    high_theta_puts = chain.get_high_theta_options('put')
    
    # Create option legs for strategy
    call_leg = chain.get_option_leg('call', 180.00)
    put_leg = chain.get_option_leg('put', 175.00)
    ```

11. **Best Practices**
    - Keep chain data up to date
    - Validate data before use
    - Use appropriate filtering methods
    - Monitor memory usage
    - Handle errors gracefully
    - Cache frequently used calculations
    - Use type hints for better code clarity
    - Document custom modifications

12. **Common Operations**
    ```python
    # Filtering options
    filtered_calls = chain.calls[
        (chain.calls['strike'] > spot_price) & 
        (chain.calls['implied_volatility'] > 0.3)
    ]
    
    # Calculating statistics
    avg_iv = chain.calls['implied_volatility'].mean()
    max_volume = chain.puts['volume'].max()
    
    # Finding specific options
    specific_call = chain.calls[
        chain.calls['strike'] == 180.00
    ].iloc[0]
    ```

This structure provides a robust foundation for options analysis and trading, with comprehensive data management and analysis capabilities.

---

## Option Chain Management

To fetch an option chain for a given underlying and expiry:

```python
# Assume you have a broker instance (Saxo, IB, etc.)
chain_data = await broker.get_options_chain('AAPL', expiry)

# Convert to OptionChain object (if needed)
from src.options.chain import OptionChain
option_chain = OptionChain.from_dict(chain_data)
```

---

## Strategy Analysis

Each `OptionStrategy` provides risk metrics and Greeks:

```python
risk_summary = strategy.get_risk_summary()
print(risk_summary)
```

You can also calculate metrics with live data:

```python
metrics = await helper.calculate_strategy_metrics(strategy, spot_price=150)
print(metrics)
```

---

## Rolling Strategies

The framework includes a `RollHelper` class to help manage rolling option strategies forward to new expiration dates. This is particularly useful for:
- Managing positions as they approach expiration
- Maintaining consistent exposure
- Adjusting positions based on market conditions

### Basic Usage

```python
from src.options.roll_helpers import RollHelper

# Initialize the helper with your broker
roll_helper = RollHelper(broker)

# Roll an entire strategy forward
new_strategy = await roll_helper.roll_strategy_forward(
    strategy=current_strategy,
    new_expiration=next_expiry,
    days_before_expiry=5,  # Roll 5 days before expiration
    min_dte=30,  # Minimum days to expiration
    max_dte=60,  # Maximum days to expiration
    adjust_strikes=True,  # Adjust strikes based on current price
    target_delta=None  # Optional target delta
)
```

### Strategy-Specific Rolling

The helper includes specialized methods for common strategies:

#### Vertical Spreads

```python
# Roll a vertical spread while maintaining width
new_spread = await roll_helper.roll_vertical_spread(
    strategy=current_spread,
    new_expiration=next_expiry,
    maintain_width=True  # Keep the same spread width
)
```

#### Iron Condors

```python
# Roll an iron condor while maintaining spread widths
new_condor = await roll_helper.roll_iron_condor(
    strategy=current_condor,
    new_expiration=next_expiry,
    maintain_widths=True  # Keep the same spread widths
)
```

### Rolling Parameters

When rolling strategies, you can control several aspects:

1. **Timing**
   - `days_before_expiry`: How many days before expiration to roll
   - `min_dte`: Minimum days to expiration for new position
   - `max_dte`: Maximum days to expiration for new position

2. **Strike Selection**
   - `adjust_strikes`: Whether to adjust strikes based on current price
   - `target_delta`: Target delta for the new position
   - `maintain_width`: Whether to maintain spread widths

3. **Position Management**
   - Maintains the same number of contracts
   - Preserves the strategy's risk profile
   - Adjusts for current market conditions

### Best Practices

1. **Timing**
   - Roll before gamma risk increases significantly
   - Consider rolling when 50% of the original time has passed
   - Avoid rolling too close to expiration

2. **Strike Selection**
   - Consider current market conditions
   - Maintain appropriate distance from current price
   - Adjust for changes in volatility

3. **Risk Management**
   - Monitor overall portfolio delta
   - Consider impact on margin requirements
   - Evaluate cost of rolling vs. closing position

4. **Market Conditions**
   - Consider volatility environment
   - Account for upcoming events
   - Evaluate market sentiment

### Example: Complete Roll Workflow

```python
from src.options.roll_helpers import RollHelper
from datetime import datetime, timedelta

# Initialize helpers
roll_helper = RollHelper(broker)

# Get current position
current_position = await broker.get_option_positions()

# Check if we need to roll
for position in current_position:
    dte = (position.expiration - datetime.now()).days
    
    if dte <= 5:  # Time to roll
        # Get next expiration
        next_expiry = await broker.get_next_expiration(
            symbol=position.symbol,
            min_dte=30,
            max_dte=60
        )
        
        # Roll the position
        new_strategy = await roll_helper.roll_strategy_forward(
            strategy=position,
            new_expiration=next_expiry,
            days_before_expiry=5,
            adjust_strikes=True
        )
        
        # Place the new position
        await broker.place_option_strategy(new_strategy)
        
        # Close the old position
        await broker.close_option_position(position)
```

---

## Implied Volatility Analysis

The framework includes a set of advanced options strategies in `src/options/advanced_strategies.py`. These strategies are more complex and often combine multiple basic strategies:

#### Iron Butterfly
A combination of a bull put spread and a bear call spread. Good for range-bound markets with limited risk and reward.

```python
from src.options.advanced_strategies import AdvancedOptionsStrategies

helper = AdvancedOptionsStrategies(broker)
strategy = helper.create_iron_butterfly(
    symbol='AAPL',
    expiration=expiry,
    put_lower=150,
    put_higher=155,
    call_lower=160,
    call_higher=165,
    quantity=1
)
```

#### Condor Spread
Similar to an iron butterfly but with a wider profit zone. Uses four different strikes and is good for range-bound markets with less precise price targets.

```python
strategy = helper.create_condor_spread(
    symbol='AAPL',
    expiration=expiry,
    put_lower=150,
    put_higher=155,
    call_lower=160,
    call_higher=165,
    quantity=1
)
```

#### Seagull Spread
Combines a bull put spread and a call spread. Has an asymmetric risk/reward profile and is good for moderately bullish outlooks.

```python
strategy = helper.create_seagull_spread(
    symbol='AAPL',
    expiration=expiry,
    put_strike=150,
    call_strike1=160,
    call_strike2=165,
    quantity=1
)
```

#### Risk Reversal
A combination of a long call and a short put, creating a synthetic long position. Good for bullish outlooks with reduced cost.

```python
strategy = helper.create_risk_reversal(
    symbol='AAPL',
    expiration=expiry,
    put_strike=150,
    call_strike=160,
    quantity=1
)
```

#### Box Spread
Combines a bull call spread and a bear put spread. Can be used for risk-free arbitrage when mispriced or for synthetic lending/borrowing.

```python
strategy = helper.create_box_spread(
    symbol='AAPL',
    expiration=expiry,
    put_lower=150,
    put_higher=155,
    call_lower=160,
    call_higher=165,
    quantity=1
)
```

#### Volatility Strategies
The framework includes two volatility-focused strategies:

1. **Volatility Skew Spread**
   - Exploits the volatility skew in the options market
   - Can be implemented with calls or puts
   - Good for volatility trading

```python
strategy = helper.create_volatility_skew_spread(
    symbol='AAPL',
    expiration=expiry,
    lower_strike=150,
    higher_strike=160,
    quantity=1,
    is_call=True
)
```

2. **Volatility Arbitrage**
   - Profits from volatility mispricing
   - Combines long and short positions
   - Good for market-neutral strategies

```python
strategy = helper.create_volatility_arbitrage(
    symbol='AAPL',
    expiration=expiry,
    strike=150,
    quantity=1
)
```

Each advanced strategy is implemented as a method in the `AdvancedOptionsStrategies` class, which takes a broker instance in its constructor. The strategies are well-documented with docstrings explaining their purpose and parameters.

When using these advanced strategies, it's important to:
1. Understand the risk/reward profile of each strategy
2. Monitor the Greeks and risk metrics closely
3. Have a clear exit strategy
4. Consider the impact of volatility changes
5. Be aware of assignment risk for short options

---

## Best Practices

1. **Strategy Selection**
   - Choose strategies based on market conditions
   - Consider volatility environment
   - Match strategy to your risk tolerance
   - Use appropriate position sizing

2. **Risk Management**
   - Set appropriate stop losses
   - Monitor position Greeks
   - Implement delta hedging when needed
   - Regular position review

3. **Market Analysis**
   - Monitor implied volatility
   - Track volatility skew
   - Watch for earnings events
   - Consider market sentiment

4. **Position Monitoring**
   - Track strategy performance
   - Monitor Greeks changes
   - Watch for early assignment risk
   - Regular rebalancing

## Examples

### Basic Strategy Analysis

```python
# Create option chain
chain = OptionChain(symbol='AAPL', expiration='2024-06-21')

# Get ATM options
atm_call = chain.get_atm_call()
atm_put = chain.get_atm_put()

# Create bull call spread
spread = {
    'legs': [
        {'type': 'call', 'strike': atm_call['strike'], 'action': 'buy', 'quantity': 1},
        {'type': 'call', 'strike': atm_call['strike'] + 5, 'action': 'sell', 'quantity': 1}
    ]
}

# Analyze spread
analysis = chain.analyze_strategy(spread)
print(f"Max Profit: ${analysis['max_profit']}")
print(f"Max Loss: ${analysis['max_loss']}")
print(f"Break-even: ${analysis['break_even']}")
```

### Rolling Strategy Example

```python
# Initialize roll helper
roll_helper = RollHelper(broker)

# Check if roll is needed
if roll_helper.should_roll_strategy(strategy, days_to_expiration=5):
    # Roll strategy forward
    new_strategy = await roll_helper.roll_strategy_forward(
        strategy,
        new_expiration='2024-07-19',
        adjust_strikes=True
    )
```

### Volatility Analysis

```python
# Get volatility skew
skew = chain.get_volatility_skew(option_type='call', delta_range=(0.1, 0.9))

# Get volatility smile
smile = chain.get_volatility_smile(option_type='put', moneyness_range=(-0.2, 0.2))

# Get term structure
term_structure = chain.get_term_structure(option_type='call', delta=0.5)

# Get volatility surface
surface = chain.get_implied_volatility_surface(option_type='call')
```

## Troubleshooting & FAQ

### Common Issues

1. **Data Retrieval Issues**
   - Check API connectivity
   - Verify symbol format
   - Ensure expiration date is valid
   - Check for market hours

2. **Strategy Analysis Issues**
   - Verify option chain data
   - Check strike prices
   - Ensure proper leg configuration
   - Validate expiration dates

3. **Rolling Issues**
   - Check days to expiration
   - Verify strike availability
   - Ensure sufficient liquidity
   - Check margin requirements

### Frequently Asked Questions

1. **How do I handle early assignment?**
   - Monitor dividend dates
   - Watch for deep ITM options
   - Consider rolling before assignment
   - Have a plan for assignment

2. **When should I roll a position?**
   - Based on days to expiration
   - When profit targets are reached
   - To adjust for market changes
   - To maintain risk parameters

3. **How do I choose the right strike?**
   - Consider your risk tolerance
   - Look at probability of profit
   - Check liquidity
   - Monitor Greeks

## Integrating Options into the Events System

### Event Types

1. **Option Chain Events**
   - Chain updates
   - Price changes
   - Volume changes
   - Open interest changes

2. **Strategy Events**
   - Position updates
   - P&L changes
   - Greek changes
   - Risk metric updates

3. **Rolling Events**
   - Roll triggers
   - Roll executions
   - Roll confirmations
   - Roll failures

### Event Handlers

```python
async def handle_option_chain_event(event):
    if event.type == 'chain_update':
        # Update option chain
        chain.update(event.data)
    elif event.type == 'price_change':
        # Handle price changes
        handle_price_change(event.data)

async def handle_strategy_event(event):
    if event.type == 'position_update':
        # Update strategy position
        strategy.update_position(event.data)
    elif event.type == 'risk_alert':
        # Handle risk alerts
        handle_risk_alert(event.data)

async def handle_roll_event(event):
    if event.type == 'roll_trigger':
        # Check if roll is needed
        if should_roll(event.data):
            # Execute roll
            await roll_strategy(event.data)
    elif event.type == 'roll_confirmation':
        # Handle roll confirmation
        handle_roll_confirmation(event.data)
```

### Event Subscriptions

```python
# Subscribe to option chain events
await event_system.subscribe('option_chain', handle_option_chain_event)

# Subscribe to strategy events
await event_system.subscribe('strategy', handle_strategy_event)

# Subscribe to roll events
await event_system.subscribe('roll', handle_roll_event)
```

---

This manual provides a comprehensive guide to options trading within the framework. For portfolio management across all asset classes, refer to the portfolio_manual.md. 