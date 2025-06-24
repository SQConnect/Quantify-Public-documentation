# Quantify: The Complete Guide

This document provides a comprehensive guide to the Quantify Trading Framework. It covers everything from initial setup to developing and deploying sophisticated, multi-asset trading strategies.

## 1. Introduction

Quantify is a sophisticated, event-driven trading analytics platform designed for the rapid development and deployment of algorithmic trading strategies. It combines advanced machine learning, sentiment analysis, real-time data processing, and comprehensive risk management to provide a complete solution for quantitative trading.

### Key Features
- **Event-Driven Architecture**: A powerful core event bus for handling everything from market data to order notifications.
- **Centralized Strategy Management**: A robust framework for running multiple strategies concurrently, managed via a CLI or direct database commands.
- **Real-time Monitoring**: Live health, performance, and resource monitoring for all running strategies.
- **Multi-Asset Support**: Built-in support for equities, options, and futures.
- **Advanced Analysis**: Integrated tools for technical analysis, pattern recognition, and options greeks.
- **Broker Integration**: A modular broker interface to connect to various exchanges and data providers.
- **Database Control**: Interact with and control strategies by inserting commands into a central database.

---

## 2. Getting Started

This section will walk you through setting up the Quantify framework on your local machine.

### 2.1. Installation

1.  **Clone the repository**:
    ```bash
    git clone https://github.com/yourusername/quantify.git
    cd quantify
    ```

2.  **Create a virtual environment** (recommended):
    ```bash
    python -m venv venv
    source venv/bin/activate  # On Windows: venv\Scripts\activate
    ```

3.  **Install dependencies**:
    ```bash
    pip install -r requirements.txt
    ```

### 2.2. Configuration

#### Environment Variables for API Keys
The framework loads sensitive API keys from environment variables. Start by copying the example file:
```bash
cp .env.example .env
```
Now, edit the `.env` file to add your credentials for the brokers you intend to use (e.g., Kraken, Saxo, Binance).

```env
# .env file
# Kraken API
KRAKEN_API_KEY=your_kraken_api_key
KRAKEN_API_SECRET=your_kraken_api_secret

# Saxo API
SAXO_CLIENT_ID=your_saxo_client_id
SAXO_CLIENT_SECRET=your_saxo_client_secret

# Binance API
BINANCE_API_KEY=your_binance_api_key
BINANCE_API_SECRET=your_binance_api_secret
```
**Security Note**: Never commit your `.env` file or API keys to version control.

#### Main Configuration File (`config/main.yaml`)
The main configuration file, `config/main.yaml`, is the control center for the framework. It defines which strategies to load, how the database is configured, and which brokers to connect to.

```yaml
# config/main.yaml

# List of strategy configuration files to load on startup
strategies:
  - "config/strategies/eth_elliott.yaml"
  - "config/strategies/short_put_spread.yaml"

# Database configuration
database:
  enabled: true
  path: "data/trading_framework.db"

# Broker and execution settings
execution:
  enabled: true
```

### 2.3. Running the Framework

The framework is operated via the command-line interface (CLI) in `main.py`.

**To start the framework**, run the following command. This will initialize the `StrategyFactory`, load the strategies listed in `main.yaml`, and start processing data.
```bash
python main.py
```

You will see logs indicating that the strategies have been deployed and are connecting to their required brokers.

---

## 3. Core Concepts

This section explains the fundamental building blocks of the Quantify framework.

### 3.1. The Strategy Factory

The **Strategy Factory** (`src/strategy_factory/strategy_factory.py`) is the central orchestrator of the system. It is responsible for:
- **Loading** strategy configurations from YAML files.
- **Instantiating** strategy objects.
- **Managing** the entire lifecycle of a strategy: deploying, running, pausing, and stopping.
- **Connecting** strategies with their required brokers.
- **Monitoring** the health and resource usage of each strategy.

When `main.py` is executed, it delegates all strategy management tasks to the Strategy Factory.

### 3.2. The Event-Driven Architecture

The framework is built on an **event-driven architecture**. Instead of components calling each other directly, they communicate by publishing and subscribing to events on a central event bus. This creates a loosely coupled and highly scalable system.

A `Strategy` subscribes to the events it cares about, and the framework's core systems (like a `Broker`) publish events as they occur.

#### Key Event Types (`src/core/events.py`)
- **`OHLCEvent`**: Published when a new candlestick (Open, High, Low, Close) is available for an asset. This is the most common event for strategies to consume.
- **`SignalEvent`**: Published by a `Strategy` when it identifies a trading opportunity. This event signals the desire to place an order.
- **`OrderEvent`**: Published by a `Broker` to provide updates on the status of an order (e.g., created, filled, cancelled).
- **`FillEvent`**: Published by a `Broker` when an order has been executed (fully or partially). This event contains the details of the trade.

#### The Event Flow
1. A **Broker** (e.g., `KrakenBroker`) receives raw data.
2. The Broker parses the data and publishes an **`OHLCEvent`** to the event bus.
3. A **Strategy** subscribed to that event receives it in its `on_candle` (or similar) method.
4. The Strategy's logic analyzes the event. If a trade is warranted, it creates and publishes a **`SignalEvent`**.
5. The **Execution Manager** receives the `SignalEvent` and translates it into an **`OrderEvent`** sent to the broker.
6. The **Broker** places the trade and publishes **`OrderEvent`** and **`FillEvent`** updates back to the bus.
7. The **Portfolio Manager** and the original **Strategy** receive these fill updates to track the position.

### 3.3. Strategy Configuration

Each strategy is defined by its own YAML configuration file, typically located in `config/strategies/`. This file contains all the parameters the strategy needs to operate.

Example: `config/strategies/eth_elliott.yaml`
```yaml
strategy:
  name: "ElliottWaveStrategy"
  class: "ElliottWaveStrategy"
  module: "src.strategies.elliott_wave_strategy"
  instance_id: "ETH_ELLIOTT_01"
  broker: "kraken"
  # Strategy-specific parameters
  params:
    symbol: "ETH/USD"
    timeframe: "1m"
    wave_lookback: 100
    risk_per_trade: 0.01
    min_volume: 10
```

This configuration tells the Strategy Factory everything it needs to know to load and run the strategy.

---

## 4. Writing Your First Strategy

Now that you understand the core concepts, let's build a trading strategy. All strategies in the framework inherit from the `BaseStrategy` class, which provides the core logic for handling data, managing state, and interacting with the event bus.

### 4.1. The BaseStrategy Class

The `BaseStrategy` (`src/strategy_framework/base_strategy.py`) is an abstract class that you will extend. It handles the "plumbing" so you can focus on trading logic.

Key responsibilities of `BaseStrategy`:
- Automatically subscribing to the data feeds defined in your config.
- Managing a "warm-up" period to pre-load historical data.
- Providing separate handlers for historical and live data.
- Exposing a simple interface for placing orders.

### 4.2. Handling Data: Historical vs. Live

To ensure indicators have enough data to be accurate from the start, the framework enforces a "warm-up" period. During this phase, historical data is fed to your strategy. Once the warm-up is complete for all required timeframes, the strategy transitions to handling live data.

You must implement two key methods to handle this:

**`on_historical_candle(self, event: OHLCEvent)`**
- **When it's called**: For every single candle during the initial warm-up phase.
- **What to do here**: Use this data to populate your indicators and data structures. For example, if you're using a 20-period moving average, this is where you feed it the first 20+ candles.
- **What NOT to do here**: Do not generate trading signals or place orders.

**`on_candle(self, event: OHLCEvent)`**
- **When it's called**: Only *after* the warm-up period is complete. This method is called for every new live candle. (Note: The method is named `on_candle`, not `on_live_candle`).
- **What to do here**: This is where your main trading logic lives. Analyze the latest candle, check your indicators, and decide whether to place a trade.

### 4.3. Working with Candle Data

Inside your strategy, you have access to a rich set of candle data. The `BaseStrategy` provides a `self.datastreams` object, which is a dictionary holding `DataStream` objects for each symbol and timeframe combination.

```python
# Get the most recent candle for a symbol/timeframe
latest_candle = self.datastreams[("ETH/USD", "1m")].get_latest_candle()

# Get the last 100 candles as a list
candles = self.datastreams[("ETH/USD", "1m")].get_candles(lookback=100)

# Get the last 100 candles as a Pandas DataFrame for analysis
df = self.datastreams[("ETH/USD", "1m")].get_candles_as_df(lookback=100)
# df columns: ['timestamp', 'open', 'high', 'low', 'close', 'volume']
```

### 4.4. Using Technical Indicators

The framework includes a rich library of technical indicators that are easy to integrate into your strategy.

1.  **Import and Initialize**: In your strategy's `__init__` method, import the candle indicators and initialize the ones you need. They are found in `src/indicators/candle_indicators.py`.
2.  **Update with Candles**: In `on_historical_candle` and `on_candle`, pass the new candle data to your indicator instances to keep them updated.
3.  **Check Values**: In `on_candle`, check the latest value of your indicator to make trading decisions.

```python
# In your strategy file...
from src.indicators.candle_indicators import CandleRSI, CandleEMA

class MyMomentumStrategy(BaseStrategy):
    def __init__(self, instance_id, config):
        super().__init__(instance_id, config)
        self.rsi = CandleRSI(period=14)
        self.ema_fast = CandleEMA(period=50)
        self.ema_slow = CandleEMA(period=200)

    async def on_historical_candle(self, event: OHLCEvent):
        # Update indicators with historical data
        candle_data = event.candle
        self.rsi.add_candle(candle_data)
        self.ema_fast.add_candle(candle_data)
        self.ema_slow.add_candle(candle_data)

    async def on_candle(self, event: OHLCEvent):
        # Update indicators with live data
        candle_data = event.candle
        self.rsi.add_candle(candle_data)
        self.ema_fast.add_candle(candle_data)
        self.ema_slow.add_candle(candle_data)

        # Check if indicators are ready (warmed up)
        if not self.rsi.is_ready() or not self.ema_slow.is_ready():
            return

        # --- Trading Logic ---
        fast_value = self.ema_fast.value
        slow_value = self.ema_slow.value
        rsi_value = self.rsi.value

        if fast_value > slow_value and rsi_value < 30:
            # Generate a buy signal
            pass
```

### 4.5. Recognizing Chart Patterns

For more complex analysis, you can use the built-in pattern recognition library (`src/technical_analysis/pattern_recognition.py`).

1.  **Initialize the Engine**: Create an instance of `PatternRecognition`.
2.  **Feed it Data**: On each new candle, get a DataFrame of your recent candles and pass it to one of the detection methods.
3.  **Act on Results**: The detector will return a result object if a pattern is found, including a confidence score.

```python
from src.technical_analysis.pattern_recognition import PatternRecognition

# In your strategy...
class MyPatternStrategy(BaseStrategy):
    def __init__(self, instance_id, config):
        super().__init__(instance_id, config)
        self.pattern_recognizer = PatternRecognition()

    async def on_candle(self, event: OHLCEvent):
        # Get candles as a DataFrame
        df = self.datastreams[(event.symbol, event.timeframe)].get_candles_as_df()
        
        if len(df) < 50: # Need enough data for patterns
            return

        # Detect patterns
        head_and_shoulders = self.pattern_recognizer.detect_head_and_shoulders(df)
        if head_and_shoulders and head_and_shoulders.confidence > 0.7:
            print(f"Detected Head and Shoulders with confidence: {head_and_shoulders.confidence}")
            # Generate a sell signal
```

### 4.6. Placing Orders

When your logic determines it's time to trade, you need to generate a `SignalEvent`. The `ExecutionManager` will listen for this event and handle the order placement with the appropriate broker.

To make this easy, the `BaseStrategy` provides a helper method: `_generate_signal`.

```python
# Inside your on_candle method...
if crossover_buy_signal and rsi_oversold:
    await self._generate_signal(
        signal_type="LONG",       # "LONG", "SHORT", "EXIT_LONG", "EXIT_SHORT"
        target_price=event.candle['close']
    )
```

This is the final piece. The signal will be picked up, converted to an order, and sent to the exchange. Your strategy will then receive `FillEvent` updates to track the position.

---

## 5. Connecting to Brokers

The framework provides a unified interface for interacting with different exchanges and brokers. The `BrokerFactory` is used to create broker instances, which all share a common interface for placing orders and receiving data.

### 5.1. Broker Factory
The `BrokerFactory` (`src/broker_interface/broker_factory.py`) simplifies the creation of broker instances. It reads the configuration, handles authentication, and returns a ready-to-use broker object. In general, you won't interact with this directly; the `StrategyFactory` uses it to provide the correct broker to your strategy at runtime.

### 5.2. Supported Brokers
The framework supports several brokers, including:
- **Kraken**: For cryptocurrencies.
- **Saxo Bank**: For stocks, forex, commodities, and options.
- **Interactive Brokers**: For stocks, options, and futures.
- **Binance**: For cryptocurrencies.
- **MockBroker**: A simulated broker for testing without connecting to a live exchange.

Configuration for each broker is handled via environment variables (`.env` file) and your strategy's YAML file.

---

## 6. Portfolio & Risk Management

The portfolio management system (`src/portfolio/portfolio_manager.py`) provides a unified view of your positions, cash, and risk across all connected brokers and asset classes.

### 6.1. Position Management
You can track your open positions across all accounts. The `PortfolioManager` automatically synchronizes with the brokers to keep your positions up to date.

### 6.2. Cash Management
The system tracks cash balances in multiple currencies and provides tools for currency conversion and managing cash utilization.

### 6.3. Risk Management
A key feature is the ability to define and enforce portfolio-level risk limits. These can be configured globally and include:
- **Max Position Size**: Limit the size of any single position relative to the total portfolio value.
- **Max Cash Utilization**: Ensure a certain percentage of cash is held in reserve.
- **Max Leverage**: Prevent excessive use of margin.
- **Max Drawdown**: A hard stop to prevent catastrophic losses.

If a new trade would violate one of these rules, the framework can automatically block it.

---

## 7. Advanced Strategy Guides

Beyond simple strategies based on indicators, the framework has specialized support for complex options and futures strategies.

### 7.1. Options Strategies

The options framework (`src/options/`) provides a powerful set of tools for defining and executing multi-leg options trades.

- **Core Components**: It includes classes for `OptionLeg`, `OptionChain`, and a powerful `OptionsHelper` that abstracts broker communications.
- **Programmatic Execution**: You can instantiate pre-built strategies like `IronCondorStrategy` or `CalendarSpreadStrategy`, define their parameters (e.g., target delta, days to expiry), and execute them programmatically. This is perfect for bots executing specific, known setups.
- **Ready-to-use Strategies**: The framework includes implementations for dozens of common strategies, including Verticals, Straddles, Iron Condors, Butterflies, and more. See `src/options/strategies/` for a full list.

**Example**: For a full walkthrough, see the `run_short_put_spread_strategy.py` script in `examples/options/`.

### 7.2. Futures Strategies

The futures framework (`src/futures/`) provides tools for trading futures contracts, including complex multi-leg spreads.

- **`FuturesHelper`**: This is the main interface for fetching futures contract chains and placing orders.
- **Built-in Spreads**: The framework has built-in support for common futures spreads:
    - **Calendar Spreads**: Trading the futures curve shape.
    - **Intermarket Spreads**: Trading related commodities (e.g., WTI vs. Brent Crude).
    - **Processing Spreads**: Trading economic relationships (e.g., the Crack Spread).
    - **Statistical Arbitrage**: Trading pairs based on cointegration.

**Examples**: See the scripts in `examples/futures/` for detailed implementations of each spread type.

---

## 8. Machine Learning Strategies

The framework includes a comprehensive machine learning module (`src/ml/`) for building and deploying predictive models.

- **Feature Engineering**: A `FeatureBuilder` helps create features from price data (returns, volatility, technical indicators) and even news data (sentiment scores, news counts).
- **Supervised Learning**: You can train traditional ML models (Random Forest, XGBoost, etc.) to predict outcomes like price direction. The `ModelBuilder` class handles data preparation, training, and evaluation.
- **Reinforcement Learning**: For fully autonomous strategies, you can train a Deep Q-Learning agent (`DQNAgent`) in a simulated `TradingEnvironment`. The agent learns an optimal trading policy through trial and error.

**Example**: See the `rl_trader.py` and `model_builder.py` files for details on the implementation.

---

## 9. Visualization & Dashboards

Understanding your strategy's performance and market data is critical. The `UnifiedVisualizer` (`src/visualization/unified_visualizer.py`) uses Plotly to create rich, interactive charts.

### 9.1. Chart Types
You can plot:
- **Candlestick charts** with trade entry/exit markers.
- **Technical indicators** overlaid on the price or in subplots.
- **Elliott Wave** patterns and Fibonacci levels.
- **Strategy performance**, including equity curves and drawdown.

### 9.2. Exporting
Charts can be saved as interactive **HTML** files or as static **PNG** images, perfect for reports and analysis.

### 9.3. Interactive Dashboard
The visualizer can also launch a web-based dashboard using Dash, allowing you to interactively explore your data, select date ranges, and view performance metrics in real-time.

**Example**: See `visualize_strategy.py` in `src/examples/` for a demonstration.

---

## 10. Logging and Debugging

A robust logging system is crucial for monitoring and debugging. The framework uses a custom `logger_factory` to create structured, consistent logs.

### 10.1. Getting a Logger
To get a logger in your strategy or component, use the factory:
```python
from src.logging_stats.logger_factory import get_logger

logger = get_logger(__name__, "MyStrategyComponent")
```

### 10.2. Log Format
Logs are formatted to include a timestamp, log level, the module and component name, the thread ID, and the message. This makes it easy to trace the execution flow of a specific strategy instance.
```
2025-06-02 22:07:43 | INFO | MyStrategyComponent | Thread-8533597952 | Received new candle for ETH/USD
```

### 10.3. Best Practices
- **Use Component Names**: Always provide a component name to distinguish logs from different parts of the application.
- **Log Exceptions Correctly**: When catching an exception, use `logger.error("...", exc_info=True)` to include the full stack trace.
- **Log Strategy State**: Periodically log key information about your strategy's state, such as current position, indicator values, and P&L.
- **Log Files**: Logs are automatically created and rotated in the `logs/` directory. 