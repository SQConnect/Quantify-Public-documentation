# The Quantify Options Trading Framework

This document provides a comprehensive guide to using the options trading module within the Quantify framework. It covers the core components, how to define and execute strategies, and advanced features like position management and optimization.

## 1. Overview

The options framework is designed to provide a robust set of tools for defining, executing, and managing single and multi-leg options strategies. It abstracts away broker-specific complexities, allowing you to focus on strategy logic.

The framework can be used in two primary ways:

1.  **Programmatic Strategy Execution**: This approach is used for creating and executing specific, well-defined options strategies on command. You instantiate a strategy class (e.g., `IronCondorStrategy`), define its parameters, and execute the trade. This is ideal for discretionary trading or for bots executing specific setups.
2.  **Event-Driven Strategies**: This approach is for building fully automated strategies that react to real-time market data. By inheriting from a specific base class, your strategy can listen to events (like new candle data) and make decisions to enter, exit, or manage options positions dynamically.

## 2. Core Components

These are the fundamental building blocks of the options module.

### `OptionLeg`
The most basic element, `OptionLeg`, is a data class that represents a single option contract. It holds all the essential information for one leg of a trade.

-   **File:** `src/options/option_leg.py`
-   **Key Attributes:**
    -   `symbol`: The broker-specific symbol for the option.
    -   `strike`: The strike price.
    -   `option_type`: `OptionType.CALL` or `OptionType.PUT`.
    -   `expiry`: The expiration `datetime`.
    -   `greeks`: A `Greeks` object containing values like Delta, Vega, etc.

### `OptionChain`
An `OptionChain` is a container that holds all the available `OptionLeg`s (both calls and puts) for a single underlying asset and a specific expiration date.

-   **File:** `src/options/chain.py`
-   **Key Methods:**
    -   `add_leg(leg)`: Adds an `OptionLeg` to the chain.
    -   `get_leg_by_strike(strike, option_type)`: Finds a specific leg within the chain.
    -   `sort_legs()`: Sorts calls and puts by strike price.

### `OptionsHelper`
The `OptionsHelper` is the primary interface for all interactions with the broker's options functionalities. It abstracts away the specific implementation details of a given broker (e.g., Saxo, Interactive Brokers), providing a consistent API for your strategies.

-   **File:** `src/options/helper.py`
-   **Key Methods:**
    -   `get_options_chain(underlying, expiry_date)`: Fetches the full option chain.
    -   `place_order(...)`: Places an order for a single option leg.
    -   `place_strategy_order(legs, price)`: Places a complex, multi-leg order for an entire strategy at a net debit/credit.
    -   `subscribe_to_greeks(leg)`: Subscribes to real-time Greeks updates for a leg.

### `Greeks`
A simple data class to store the Greek values for an option contract.

-   **File:** `src/options/greeks.py`
-   **Attributes:** `delta`, `gamma`, `theta`, `vega`, `rho`, `implied_volatility`.

## 3. Programmatic Strategy Execution

This is the most direct way to use the framework. You choose one of the pre-built strategy classes, provide the necessary parameters, and execute the trade.

### The Programmatic `BaseOptionsStrategy`
All programmatic strategy classes inherit from a base class that provides common functionality.

-   **File:** `src/options/strategies/base_strategy.py`
-   **Key Features:**
    -   It requires the strategy's parameters (underlying, expiry, etc.) upon initialization.
    -   It contains an abstract `_define_legs` method, which each child strategy must implement to construct its specific combination of options.
    -   It holds the final list of legs in the `self.legs` attribute, ready to be sent to the broker.

### Workflow
Executing a programmatic strategy follows these steps:

1.  **Instantiate Broker**: Create and connect a broker instance using the `BrokerFactory`.
2.  **Instantiate Strategy**: Choose a strategy from `src/options/strategies` and create an instance with its required parameters (e.g., strikes, expiry).
3.  **Assign OptionsHelper**: Manually assign the broker's `options_helper` to the strategy instance. This gives the strategy the ability to communicate with the broker.
4.  **Define Legs**: Call the asynchronous `strategy.define_legs()` method. The strategy will now use the `OptionsHelper` to fetch the option chain and find the specific contracts that match its defined parameters.
5.  **Place Order**: Pass the list of legs from `strategy.legs` to the `OptionsHelper.place_strategy_order()` method to execute the trade as a single multi-leg order.
6.  **Manage Position**: After execution, you can enter a loop to monitor the position, checking for early assignment or conditions to close the trade.

### Full Code Example

The following example demonstrates the complete workflow for executing a Short Put Spread.

-   **Source File:** `examples/options/run_short_put_spread_strategy.py`

```python
import asyncio
import logging
import sys
import os
from datetime import datetime

# Add project root to the Python path
project_root = os.path.abspath(os.path.join(os.path.dirname(__file__), '..', '..'))
if project_root not in sys.path:
    sys.path.insert(0, project_root)

from src.broker_interface.broker_factory import BrokerFactory
from src.core.config.config_manager import ConfigManager
from src.core.events import event_manager, OrderEvent
from src.options.strategies.short_put_spread import ShortPutSpreadStrategy
from src.broker_interface.models.orders import OrderStatus

# Configure basic logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

async def on_order_update(event: OrderEvent):
    """Event handler for printing received order updates."""
    order = event.order
    logger.info(
        f"--- ORDER UPDATE for {order.symbol} [ID: {order.order_id}] ---\n"
        f"Status: {order.status.value} ({order.sub_status.value})\n"
        f"Activity: {order.buy_sell} {order.amount} @ {order.price}\n"
        f"Filled: {order.filled_amount} @ {order.average_price}\n"
        f"----------------------------------------------------"
    )

async def main():
    """
    Initializes and runs the ShortPutSpreadStrategy.
    """
    # --- Configuration ---
    UNDERLYING_SYMBOL = "NVDA:xnas"
    # --- End Configuration ---

    broker = None
    try:
        # 1. Setup Broker
        config_manager = ConfigManager()
        saxo_config = config_manager.get_broker_config('saxo')
        broker = BrokerFactory.create_broker('saxo', saxo_config)
        
        event_manager.subscribe(OrderEvent, on_order_update)
        
        await broker.connect()
        await broker.subscribe_to_orders()
        
        # 2. Initialize the Strategy with its parameters
        strategy = ShortPutSpreadStrategy(
            underlying=UNDERLYING_SYMBOL,
            quantity=1,
            spread_width=5, # Sell a spread with a $5 width
            days_to_expiry=21,
            target_delta=-0.25,
            days_before_expiry_to_close=3
        )
        # 3. Assign the OptionsHelper from the broker to the strategy
        strategy.set_broker(broker)

        # 4. Define the strategy legs
        logger.info("Defining strategy legs...")
        await strategy.define_legs()
        
        logger.info(f"Legs defined: {strategy.legs}")
        
        # 5. Place the opening order
        # For this example, we'll request the current market price first
        price_info = await strategy.options_helper.get_strategy_price(strategy.legs)
        limit_price = price_info.get('NetPrice') # Use the NetPrice from the broker
        
        if limit_price is None:
            logger.error("Could not retrieve a valid price for the strategy. Aborting.")
            return

        logger.info(f"Placing opening order for Short Put Spread at limit price: {limit_price}")
        confirmation = await strategy.options_helper.place_strategy_order(strategy.legs, limit_price)
        logger.info(f"Opening order placed: {confirmation}")
        
        # 6. Management loop
        logger.info("Entering management loop... (Ctrl+C to exit)")
        strategy.order_id = confirmation.get('OrderId')
        while True:
            # Check if it's time to close the position
            if datetime.now() >= strategy.target_exit_date:
                logger.info("Target exit date reached. Closing position.")
                # Add logic here to place the closing order
                break

            # Check for assignment
            if await strategy.check_for_assignment():
                logger.warning("Strategy has been assigned! Exiting loop.")
                # Add logic here to handle the assigned position
                break
            
            logger.info("No management action needed yet. Sleeping for 1 hour.")
            await asyncio.sleep(3600) # Check once per hour

    except asyncio.CancelledError:
        logger.info("Run cancelled by user.")
    except Exception as e:
        logger.error(f"An error occurred: {e}", exc_info=True)
    finally:
        if broker and broker.is_connected:
            logger.info("Disconnecting from broker.")
            await broker.disconnect()
        logger.info("Script finished.")


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        logger.info("Script interrupted by user.")
```

## 4. Implemented Strategies

The framework comes with a variety of common, ready-to-use strategy implementations, located in `src/options/strategies/`.

#### `VerticalSpreadStrategy`
- **Description**: A generic class for all four types of vertical spreads. It buys one option and sells another of the same type and expiry, but different strikes.
- **`__init__` parameters**: `strategy_type` (e.g., `OptionStrategyType.BULL_CALL_SPREAD`), `underlying`, `expiry`, `strike1`, `strike2`, `quantity`.

#### `ShortPutSpreadStrategy`
- **Description**: A specialized vertical spread that specifically creates a short put spread (a credit spread). It identifies the legs based on a target delta and spread width.
- **`__init__` parameters**: `underlying`, `quantity`, `spread_width`, `days_to_expiry`, `target_delta`.

#### `StraddleStrategy`
- **Description**: A volatility strategy involving a call and a put with the same strike and expiry. Can be long (buy both) or short (sell both).
- **`__init__` parameters**: `strategy_type` (`LONG_STRADDLE` or `SHORT_STRADDLE`), `underlying`, `expiry`, `strike`, `quantity`.

#### `StrangleStrategy`
- **Description**: A volatility strategy similar to a straddle, but the call and put have different (typically OTM) strike prices.
- **`__init__` parameters**: `strategy_type` (`LONG_STRANGLE` or `SHORT_STRANGLE`), `underlying`, `expiry`, `call_strike`, `put_strike`, `quantity`.

#### `ButterflySpreadStrategy`
- **Description**: A neutral, limited-risk strategy involving three legs: buying the two outer "wings" and selling twice the number of contracts at the central "body" strike.
- **`__init__` parameters**: `strategy_type`, `underlying`, `expiry`, `lower_strike`, `middle_strike`, `upper_strike`, `quantity`.

#### `IronButterflyStrategy`
- **Description**: A four-leg strategy that combines a short ATM straddle with a long OTM strangle. It is a neutral, limited-risk strategy that profits from low volatility.
- **`__init__` parameters**: `underlying`, `expiry`, `atm_strike`, `wing_width`, `quantity`.

#### `IronCondorStrategy`
- **Description**: A popular neutral, limited-risk strategy consisting of four legs: a bull put spread and a bear call spread.
- **`__init__` parameters**: `underlying`, `expiry`, `buy_call_strike`, `sell_call_strike`, `sell_put_strike`, `buy_put_strike`, `quantity`.

#### `CalendarSpreadStrategy` (Time Spread)
- **Description**: A strategy that profits from the passage of time and changes in implied volatility. It involves selling a short-dated option and buying a longer-dated option of the same type and strike.
- **`__init__` parameters**: `strategy_type`, `underlying`, `long_leg_expiry`, `short_leg_expiry`, `strike`, `quantity`.

#### `DiagonalSpreadStrategy`
- **Description**: Similar to a calendar spread, but the two legs have different strike prices in addition to different expiries.
- **`__init__` parameters**: `strategy_type`, `underlying`, `long_leg_expiry`, `short_leg_expiry`, `long_leg_strike`, `short_leg_strike`, `quantity`.

#### `BackRatioStrategy`
- **Description**: A volatility strategy where an unequal number of options are bought and sold. For example, selling one ITM call and buying two OTM calls.
- **`__init__` parameters**: `strategy_type`, `underlying`, `expiry`, `short_strike`, `long_strike`, `ratio` (e.g., `(1, 2)`), `is_credit`.

## 5. Position Management & Utilities

The framework provides tools to help manage active positions.

### `RollHelper`
- **File:** `src/options/roll_helpers.py`
- **Purpose**: The `RollHelper` class provides high-level methods to roll an existing option position forward to a new expiration date. This is crucial for managing long-term strategies.
- **Key Methods**:
    - `roll_leg_forward(...)`: Closes a single option leg and opens a new one with a different strike and/or expiry in a single multi-leg transaction.
    - `roll_vertical_spread(...)`, `roll_iron_condor(...)`: Strategy-specific helpers that can roll an entire spread forward, intelligently maintaining the spread width if desired.

### Checking for Early Assignment
- **Method:** `BaseOptionsStrategy.check_for_assignment()`
- **Purpose**: This is a critical risk management function for any strategy that involves selling options. Early assignment occurs when the holder of an option you sold exercises their right to buy or sell the underlying stock *before* the expiration date. This can happen unexpectedly, instantly changing your position from an option spread to a stock position (either long or short), exposing you to significant new risks.
- **Usage**: The `check_for_assignment()` method should be called periodically within your management loop. It scans your account's positions for any new stock holdings that match the underlying of your options strategy, which would indicate an assignment has occurred.

```python
# In your management loop
if await strategy.check_for_assignment():
    logger.warning(f"Assignment detected for {strategy.underlying}! Halting strategy and manual intervention required.")
    # Implement logic to handle the new stock position (e.g., close it, hedge it)
    break
```

### Listening to Real-time Option Events (e.g., Greeks)
The framework allows you to subscribe to real-time data streams for specific option contracts, such as updates to their Greeks (Delta, Gamma, Vega, etc.). This is essential for strategies that need to react to changes in implied volatility or the option's sensitivity to price movements.

- **Workflow**:
    1.  Define an `async` event handler function that will process the incoming event (e.g., `on_greeks_update`).
    2.  Subscribe your handler to the appropriate event type using the global `event_manager`. For Greeks, this is `MarketDataEventType.GREEKS`.
    3.  After finding a specific `OptionLeg` you want to monitor, use the `OptionsHelper.subscribe_to_greeks(leg)` method.
    4.  The broker will begin pushing updates, and your event handler will be called automatically for each new update.

#### Example: Subscribing to Greeks Updates
The following example finds an option and prints its Greeks updates for 60 seconds.

```python
import asyncio
import logging
from datetime import datetime, timedelta
from src.core.events import event_manager, GreeksEvent, MarketDataEventType
from src.options.option_types import OptionType
# (Assuming broker and options_helper are already initialized and connected)

async def on_greeks_update(event: GreeksEvent):
    """Event handler for printing received Greeks updates."""
    logger.info(
        f"GREEKS UPDATE for Symbol {event.symbol}: "
        f"Delta={event.data['delta']:.4f}, "
        f"Gamma={event.data['gamma']:.4f}, "
        f"Vega={event.data['vega']:.4f}, "
        f"Theta={event.data['theta']:.4f}"
    )

async def run_greeks_subscription(options_helper, underlying, strike, option_type):
    # Register our event handler with the event manager
    event_manager.subscribe(MarketDataEventType.GREEKS, on_greeks_update)
    
    expiry_date = (datetime.now() + timedelta(days=60)).strftime('%Y-%m-%d')

    logger.info(f"Fetching chain for {underlying} on {expiry_date}...")
    chain = await options_helper.get_options_chain(underlying, expiry_date)
    target_leg = chain.get_leg_by_strike(strike, option_type)

    if not target_leg:
        logger.error(f"Could not find leg for strike {strike}. Exiting.")
        return

    logger.info(f"Subscribing to Greeks updates for {target_leg.symbol}...")
    success = await options_helper.subscribe_to_greeks(target_leg)

    if success:
        logger.info("Subscription successful. Waiting for updates for 60 seconds...")
        await asyncio.sleep(60)
    else:
        logger.error("Subscription failed.")

# To run this:
# asyncio.run(run_greeks_subscription(options_helper, "META:xnas", 500, OptionType.CALL))
```

## 6. Event-Driven & Automated Strategies

For fully automated trading, you can create strategies that react to market data.

### The Event-Driven `BaseOptionsStrategy`
- **File:** `src/options/base_strategy.py`
- **Purpose**: This base class inherits from the main framework's `BaseStrategy`. It is designed to be used within the application's main event loop.
- **Key Methods**:
    - `on_candle(candle_event)`: An abstract method that you must implement. This method is called every time a new candle of data is received for the underlying asset, allowing you to implement your trading logic.

### Example: `SyntheticStockStrategy`
- **File:** `src/options/strategies/synthetic_stock.py`
- **Description**: This is a powerful, fully implemented example of an event-driven strategy. A synthetic stock position (long call + short put) mimics owning the underlying stock.
- **Features**:
    - **Entry Logic**: Enters a position based on a moving average crossover signal received in `on_candle`.
    - **Exit Logic**: Manages the active position, closing it if take-profit or stop-loss levels are hit.
    - **Automated Rolling**: Automatically uses the `RollHelper` to roll the position forward as it nears expiration, making it a perpetual strategy.

This class serves as an excellent template for building your own complex, automated options strategies.

## 7. Advanced: Strategy Optimization

The framework includes a powerful `StrategyOptimizer` to backtest and find the optimal parameters for a given strategy based on historical data.

- **File:** `src/options/optimization.py`

### `StrategyOptimizer`
This class uses numerical optimization techniques (like `scipy.optimize.differential_evolution`) to find the best inputs for a strategy.

- **Workflow**:
    1.  Instantiate a strategy-specific optimizer (e.g., `IronCondorOptimizer`).
    2.  Provide it with historical price data for the underlying asset.
    3.  Call the `optimize_strategy` method, specifying your objective (e.g., maximize `sharpe_ratio` or `total_return`).
    4.  The optimizer will run many backtests to find the set of parameters (like strike distances) that best met your objective over the historical period.
- **Available Optimizers**:
    - `BullCallSpreadOptimizer`
    - `IronCondorOptimizer`

This tool allows you to rigorously test and refine your strategies before deploying them with live capital.

## 8. Creating a Custom Strategy

To create a new strategy, you create a new Python class that inherits from `BaseOptionsStrategy`.

### Steps:

1.  **Create the File:** Add a new file in `src/options/strategies/`, for example, `my_new_strategy.py`.

2.  **Define the Class:**
    *   Import `BaseOptionsStrategy`.
    *   Define your class: `class MyNewStrategy(BaseOptionsStrategy):`.
    *   Create the `__init__` method. It **must** call `super().__init__(...)` and pass through the `strategy_type`, `underlying`, etc.
    *   Add any custom parameters your strategy needs (e.g., `target_profit_pct`, `ma_window`).

3.  **Implement the Logic:**
    *   **`_define_legs(self)`:** This is often the most important method for programmatic strategies. Here, you will:
        *   Use `self.options_helper.get_options_chain(...)` to fetch available options.
        *   Implement your logic to select the specific legs for your strategy (e.g., find the strikes closest to a certain delta or price).
        *   Populate `self.legs` with a list of dictionaries, where each dictionary defines a leg, its side ('Buy' or 'Sell'), and quantity.
    *   **Event Handlers (Optional):** For event-driven strategies, you can implement methods like `on_candle(self, event)` or `on_order_update(self, event)` to have your strategy react to market data or order fills in real-time.

### Template for a New Programmatic Strategy

```python
import logging
from ..base_strategy import BaseOptionsStrategy
from src.options.option_types import OptionStrategyType

class MyNewStrategy(BaseOptionsStrategy):
    """
    Description of my new amazing strategy.
    """
    def __init__(self, underlying: str, expiry: datetime, quantity: int, custom_param: float):
        super().__init__(
            strategy_type=OptionStrategyType.BULL_CALL_SPREAD, # Or other appropriate type
            underlying=underlying,
            expiry=expiry,
            quantity=quantity
        )
        self.custom_param = custom_param
        self.log = logging.getLogger(__name__)

    async def _define_legs(self):
        """
        Fetches the option chain and constructs the strategy legs.
        """
        self.log.info("Defining legs for MyNewStrategy...")
        chain = await self.options_helper.get_options_chain(self.underlying, self.expiry_str)

        # --- Your logic to find the right strikes goes here ---
        # Example: find the ATM call
        current_price = await self.options_helper.broker.get_market_data(self.underlying)['mid']
        leg1 = self.options_helper.find_closest_strike(chain.calls, current_price)
        # ---

        if not leg1:
            raise ValueError("Could not find a suitable leg.")

        # Populate the legs for the order
        self.legs = [
            {"leg": leg1, "side": "Buy", "quantity": self.quantity},
        ]
        self.log.info("MyNewStrategy legs defined successfully.") 