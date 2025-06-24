# Futures Trading Framework Manual

This document provides a guide to using the futures trading framework within the Quantify system. It covers fetching futures data, placing orders, and creating custom trading strategies.

## Core Components

The futures framework is built around two key components:

1.  **`FuturesHelper`**: Located in `src/algorithm_engine/helpers/futures_helper.py`, this class provides a high-level API for all futures-related operations, such as fetching contract chains and placing orders. It abstracts away the broker-specific details.
2.  **`BaseFuturesStrategy`**: Found in `src/futures/strategies/base_strategy.py`, this is the abstract base class that all futures strategies must inherit from. It ensures a consistent structure for strategy implementation and execution.

---

## 1. Fetching a Futures Chain

Before you can trade, you need to know which contracts are available. The `fetch_futures_chain.py` script is provided as an example of how to do this.

**Location:** `examples/futures/fetch_futures_chain.py`

This script connects to the broker and uses `FuturesHelper.get_futures_chain()` to retrieve a list of all available futures contracts for a specific underlying (e.g., WTI Crude Oil, 'WBS').

### Example Usage:

```bash
python3 examples/futures/fetch_futures_chain.py
```

The output will be a list of contracts, including their symbols, expiry dates, and other details. These symbols are required for placing orders.

---

## 2. Placing a Single Futures Order

The framework supports placing standard market and limit orders for individual futures contracts.

**Example Script:** `examples/futures/place_futures_order.py`

This script demonstrates how to:
- Connect to the broker.
- Instantiate the `FuturesHelper`.
- Place a market order for a specific futures contract symbol.

```python
# Example from place_futures_order.py
order_result = await futures_helper.place_order(
    symbol="WBSN5",  # A valid symbol from the futures chain
    side="buy",
    quantity=Decimal("1"),
    order_type="market"
)
```

---

## 3. Advanced Trading Strategies

The framework includes a library of advanced, multi-leg strategies for sophisticated trading.

### 3.1 Calendar Spread

A calendar spread (or time spread) involves buying a futures contract for one expiration and selling a contract for a different expiration on the **same underlying**. It's a common strategy for trading on the shape of the futures curve (contango or backwardation).

-   **Strategy Class:** `src.futures.strategies.calendar_spread.FuturesCalendarSpread`
-   **Example Script:** `examples/futures/run_calendar_spread_strategy.py`

#### Configuration Example:
```python
calendar_spread_config = {
    "underlying_symbol": "WBS",  # WTI Crude Oil
    "front_month_symbol": "WBS_CONT_2408", # Example: August 2024 contract
    "back_month_symbol": "WBS_CONT_2409",  # Example: September 2024 contract
    "quantity": "1",
    "spread_side": "buy"  # 'buy' or 'sell' the spread
}
```

### 3.2 Intermarket Spread

An intermarket spread involves simultaneously buying a futures contract for one commodity and selling a contract for a **different but related commodity** for the same contract month.

-   **Strategy Class:** `src.futures.strategies.intermarket_spread.IntermarketSpread`
-   **Example Script:** `examples/futures/run_intermarket_spread_strategy.py`

#### Configuration Example:
```python
intermarket_spread_config = {
    "leg_one_symbol": "WBSN5",   # WTI Crude Future
    "leg_two_symbol": "LCOQ5",  # Brent Crude Future
    "quantity": "1",
    "spread_side": "buy"  # Buy the spread -> Buy WTI, Sell Brent
}
```

### 3.3 Processing Spread (e.g., Crack Spread)

A processing spread is a multi-leg trade based on an economic relationship, such as the profit margin of processing a raw commodity into a refined one.

-   **Strategy Class:** `src.futures.strategies.processing_spread.ProcessingSpread`
-   **Example Script:** `examples/futures/run_crack_spread_strategy.py`

#### Configuration Example (3-2-1 Crack Spread):
```python
crack_spread_config = {
    "spread_name": "3-2-1 Crack Spread",
    "spread_side": "buy", # Buy the spread = refineries are profitable
    "legs": [
        { "symbol": "RBGV5", "ratio": 2, "side_factor": 1 }, # Gasoline
        { "symbol": "HON5", "ratio": 1, "side_factor": 1 },  # Heating Oil
        { "symbol": "WBSN5", "ratio": 3, "side_factor": -1 } # Crude Oil
    ]
}
```

### 3.4 Statistical Arbitrage (Pairs Trading)

This strategy executes a trade on a pair of instruments that are statistically determined to be related. The implementation assumes the statistical analysis (e.g., hedge ratio calculation) has already been performed.

-   **Strategy Class:** `src.futures.strategies.statistical_arbitrage.StatisticalArbitrage`
-   **Example Script:** `examples/futures/run_stat_arb_strategy.py`

#### Configuration Example (Gold vs. Silver):
```python
stat_arb_config = {
    "pair_name": "Gold-Silver Ratio",
    "leg_one_symbol": "GCZ5",
    "leg_two_symbol": "SIZ5",
    "hedge_ratio": "1.5",  # From external analysis
    "trade_signal": "buy" # From external analysis
}
```

### 3.5 Index Arbitrage (Futures vs. ETF)

This strategy attempts to profit from price discrepancies between a futures contract and its corresponding ETF.

**Note:** This implementation is conceptual. A true implementation would require a more generic `TradingHelper` that can handle orders for different asset types (Futures, Stocks) in a single strategy.

-   **Strategy Class:** `src.futures.strategies.index_arbitrage.IndexArbitrage`
-   **Example Script:** `examples/futures/run_index_arb_strategy.py`

#### Configuration Example (E-mini S&P 500 vs. SPY):
```python
index_arb_config = {
    "arbitrage_name": "E-mini S&P 500 vs SPY ETF",
    "trade_signal": "buy", # Buy the Future, Sell the ETF
    "leg_one": {
        "symbol": "ESZ5",
        "asset_type": "Future"
    },
    "leg_two": {
        "symbol": "SPY:ARCX",
        "asset_type": "Stock"
    }
}
```

---

## 4. Advanced Operations: Rolling and Expiration

The framework also provides helpers for common futures management tasks.

*(Note: These helpers need to be created)*

-   **Rolling Positions:** Use the `FuturesRollHelper` to automatically close an expiring contract and open a new one in a further-out month.
-   **Checking for Assignment:** Use the `FuturesAssignmentHelper` to monitor positions that are nearing their last trade date to avoid unwanted delivery.
