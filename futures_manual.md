# Futures Trading Framework Manual

This document provides a guide to using the futures trading framework within the Quantify system. It covers fetching futures data, placing orders, rolling contracts, and creating custom trading strategies.

## Core Components

The futures framework is built around a few key components:

1.  **`FuturesHelper`**: Located in `src/futures/futures_helper.py`, this class provides a high-level API for all futures-related operations. It abstracts away broker-specific details, allowing strategies to be broker-agnostic.
2.  **`FuturesRollHelper`**: Located in `src/futures/roll_helper.py`, this class provides functionality to automate the process of rolling a futures position from an expiring contract to a new one.
3.  **`BaseFuturesStrategy`**: An abstract base class in `src/futures/base_strategy.py` that all futures strategies should inherit from. It provides common functionalities and ensures a consistent structure.
4.  **Broker Implementations**: Each broker (`Saxo`, `Kraken`, etc.) has its own implementation of the futures-related methods defined in `BaseBroker`.

---

## 1. Understanding the Futures Chain

The futures chain is a list of all available contracts for a given underlying asset. Each contract in the chain is represented as a dictionary with a standardized structure.

### Futures Chain Data Structure

When you fetch a futures chain, you will get a list of dictionaries, where each dictionary contains the following keys:

-   `symbol` (str): The broker-specific trading symbol for the contract (e.g., 'NQU5').
-   `uic` (str): The Unique Instrument Code, a broker-specific identifier.
-   `description` (str): A human-readable description of the contract.
-   `expiry_date` (datetime): The expiration date of the contract.
-   `asset_type` (str): The type of asset (e.g., 'ContractFutures').
-   `exchange` (str): The exchange where the contract is traded.

---

## 2. The FuturesHelper

The `FuturesHelper` is the primary interface for interacting with futures markets. It simplifies common tasks and should be used within all futures strategies.

### Key Methods

-   **`get_futures_chain(underlying_symbol: str)`**: Fetches the futures chain for the specified underlying.
-   **`get_contract_details(symbol: str)`**: Retrieves detailed information for a single futures contract.
-   **`place_order(...)`**: Places an order for a single futures contract.
-   **`get_continuous_future(underlying_symbol: str)`**: Gets the front-month contract for a given underlying.
-   **`print_futures_chain(underlying_symbol: str)`**: A utility function that fetches and prints the futures chain in a readable format to the console.

---

## 3. Rolling Futures with FuturesRollHelper

Rolling a futures position is a critical operation to avoid expiration and maintain a continuous position. The `FuturesRollHelper` automates this process.

### `roll_position(from_symbol: str, to_symbol: str, quantity: Decimal)`

This is the core method of the helper. It performs the following actions:
1.  Finds the existing position for the `from_symbol`.
2.  Places an opposing market order to close the current position.
3.  Places a new market order to open a position in the `to_symbol`.

The helper is injected into your strategy via the `BaseFuturesStrategy`'s `set_futures_helper` method, so you can access it directly with `self.roll_helper`.

---

## 4. Building a Custom Futures Strategy

All futures strategies should inherit from `BaseFuturesStrategy`. The following example demonstrates how to build a strategy that trades a calendar spread and uses the roll helper.

**Example Strategy: `nq_test_strategy.py`**

This strategy demonstrates several key concepts:
-   Placing a multi-leg spread by submitting two separate orders.
-   Monitoring the status of multiple active orders.
-   Using the `FuturesRollHelper` to roll the position after a set duration.

### Step 1: Initialization and Setup

In the `__init__` and `initialize` methods, you set up the strategy's parameters and schedule any recurring tasks. The `roll_helper` is automatically initialized in the base class.

```python
# From src/futures/strategies/nq_test_strategy.py

class NQTestStrategy(BaseFuturesStrategy):
    def __init__(self, ...):
        super().__init__(...)
        self.underlying_symbol = self.config.get('underlying_symbol', 'NQ')
        self.hold_duration = timedelta(minutes=self.config.get('hold_duration_minutes', 30))
        self._position_entry_time: datetime | None = None
        self._active_order_ids: List[str] = []

    def set_futures_helper(self, helper: 'FuturesHelper') -> None:
        """Injects the futures helper dependency."""
        super().set_futures_helper(helper)
```

### Step 2: Placing a Spread Trade

Since the broker's multi-leg order endpoint does not support futures, the correct way to place a spread is by submitting two separate orders.

```python
# From _place_new_trade method

# Get the full futures chain
chain = await self.futures_helper.get_futures_chain(self.underlying_symbol)
chain.sort(key=lambda x: x['expiry_date'])
front_month = chain[0]
second_month = chain[1]

# Define the legs for the spread order
legs = [
    {'symbol': front_month['symbol'], 'side': OrderSide.BUY, 'quantity': 1},
    {'symbol': second_month['symbol'], 'side': OrderSide.SELL, 'quantity': 1}
]

# Place separate orders for each leg
for leg in legs:
    order = await self.broker.place_order(
        symbol=leg['symbol'],
        side=leg['side'],
        quantity=leg['quantity'],
        order_type='market'
    )
    if order and order.get('OrderId'):
        self._active_order_ids.append(order['OrderId'])
```

### Step 3: Rolling the Position

When it's time to roll, the strategy identifies the current and next-month contracts and calls the `roll_helper`.

```python
# From _roll_position method

async def _roll_position(self):
    """Rolls the front-month contract to the next month."""
    self.logger.info("Hold duration expired. Attempting to roll position...")
    chain = await self.futures_helper.get_futures_chain(self.underlying_symbol)
    if not chain or len(chain) < 2:
        self.logger.error("Not enough contracts in the chain to roll.")
        return

    chain.sort(key=lambda x: x['expiry_date'])
    from_contract = chain[0]
    to_contract = chain[1]

    await self.roll_helper.roll_position(
        from_symbol=from_contract['symbol'],
        to_symbol=to_contract['symbol'],
        quantity=1
    )
    self._position_entry_time = None  # Reset timer after rolling
```

This updated structure provides a comprehensive guide to using the futures trading framework, from understanding the data to implementing advanced rolling logic in a custom strategy.
