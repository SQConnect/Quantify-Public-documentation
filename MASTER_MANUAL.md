# Quantify: The Complete Guide

This document provides a comprehensive guide to the Quantify Trading Framework. It covers everything from initial setup to developing and deploying sophisticated, multi-asset trading strategies.

## 1. Introduction

Quantify is a sophisticated, event-driven trading analytics platform designed for the rapid development and deployment of algorithmic trading strategies. It combines advanced machine learning, sentiment analysis, real-time data processing, and comprehensive risk management to provide a complete solution for quantitative trading.

### Key Features
- **Event-Driven Architecture**: A powerful core event bus for handling everything from market data to order notifications, including DOM (Depth of Market) and advanced event types.
- **Centralized Strategy Management**: A robust framework for running multiple strategies concurrently, managed via a CLI or direct database commands.
- **Real-time Monitoring**: Live health, performance, and resource monitoring for all running strategies.
- **Multi-Asset Support**: Built-in support for equities, options, and futures.
- **Advanced Analysis**: Integrated tools for technical analysis, pattern recognition, options greeks, and order flow analysis.
- **Broker Integration**: A modular broker interface to connect to various exchanges and data providers, with broker-side normalization and sorting for futures chains.
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

#### Strategy Configuration (YAML)
Each strategy is defined by its own YAML configuration file, typically located in `config/strategies/`. This file contains all the parameters the strategy needs to operate, including new fields for DOM and broker-specific features.

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
    subscribe_dom: true  # <-- New: subscribe to DOM events
    dom_symbols:
      - "ETHUSD"
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

#### Key Event Types (see `src/data/event_types.py`)
- **`CandleEvent`**: Published when a new candlestick (Open, High, Low, Close) is available for an asset. This is the most common event for strategies to consume.
- **`DOMEvent`**: Published when new depth-of-market (order book) data is available. Used for order flow and advanced strategies.
- **`SignalEvent`**: Published by a `Strategy` when it identifies a trading opportunity. This event signals the desire to place an order.
- **`OrderEvent`**: Published by a `Broker` to provide updates on the status of an order (e.g., created, filled, cancelled).
- **`FillEvent`**: Published by a `Broker` when an order has been executed (fully or partially). This event contains the details of the trade.

#### The Event Flow
1. A **Broker** (e.g., `SaxoBroker`) receives raw data (candles, DOM, etc.).
2. The Broker parses and normalizes the data, then publishes a **`CandleEvent`** or **`DOMEvent`** to the event bus.
3. A **Strategy** subscribed to that event receives it in its `on_live_candle` or `on_dom_event` method.
4. The Strategy's logic analyzes the event. If a trade is warranted, it creates and publishes a **`SignalEvent`**.
5. The **Execution Manager** receives the `SignalEvent` and translates it into an **`OrderEvent`** sent to the broker.
6. The **Broker** places the trade and publishes **`OrderEvent`** and **`FillEvent`** updates back to the bus.
7. The **Portfolio Manager** and the original **Strategy** receive these fill updates to track the position.

#### Event Subscription Pattern
- **Strategies and components now subscribe to events by event type (e.g., `'depth'` for DOM, `'closed'` for candles) rather than by topic.**
- Example: `self.event_manager.subscribe('depth', '*', self.on_dom_event, timeframe='*')`

### 3.3. Strategy Configuration

Each strategy is defined by its own YAML configuration file, typically located in `config/strategies/`. This file contains all the parameters the strategy needs to operate, including DOM and broker-specific fields.

---

## 4. Writing Your First Strategy

Now that you understand the core concepts, let's build a trading strategy. All strategies in the framework inherit from the `BaseStrategy` class, which provides the core logic for handling data, managing state, and interacting with the event bus.

### 4.1. The BaseStrategy Class

The `BaseStrategy` (`src/strategy_framework/base_strategy.py`) is an abstract class that you will extend. It handles the "plumbing" so you can focus on trading logic.

Key responsibilities of `BaseStrategy`:
- Automatically subscribing to the data feeds defined in your config (including DOM if enabled).
- Managing a "warm-up" period to pre-load historical data.
- Providing separate handlers for historical and live data.
- Exposing a simple interface for placing orders.

### 4.2. Handling Data: Historical vs. Live

To ensure indicators have enough data to be accurate from the start, the framework enforces a "warm-up" period. During this phase, historical data is fed to your strategy. Once the warm-up is complete for all required timeframes, the strategy transitions to handling live data.

You must implement key methods to handle this:

**`on_live_candle(self, event: CandleEvent)`**
- **When it's called**: For every new live candle after warm-up.
- **What to do here**: This is where your main trading logic lives. Analyze the latest candle, check your indicators, and decide whether to place a trade.

**`on_dom_event(self, event: DOMEvent)`**
- **When it's called**: For every new DOM (order book) event if DOM subscription is enabled.
- **What to do here**: Analyze order flow, depth, and liquidity for advanced strategies.

### 4.3. Working with Candle and DOM Data

Inside your strategy, you have access to a rich set of candle and DOM data. The `BaseStrategy` provides access to recent candles and order book events for each symbol and timeframe combination.

```python
# Get the most recent candle for a symbol/timeframe
latest_candle = self.get_latest_candle(symbol, timeframe)

# Get the last 100 candles as a list
df = self.get_dataframe(symbol, timeframe, lookback=100)

# DOM event handler
async def on_dom_event(self, event: DOMEvent):
    print(f"Received DOM event for {event.symbol}: {event}")
```

### 4.4. Using Technical Indicators

The framework includes a rich library of technical indicators that are easy to integrate into your strategy.

1.  **Import and Initialize**: In your strategy's `__init__` method, import the candle indicators and initialize the ones you need.
2.  **Update with Candles**: In `on_live_candle`, pass the new candle data to your indicator instances to keep them updated.
3.  **Check Values**: In `on_live_candle`, check the latest value of your indicator to make trading decisions.

```python
from src.indicators.candle_indicators import CandleRSI, CandleEMA

class MyMomentumStrategy(BaseStrategy):
    def __init__(self, instance_id, config):
        super().__init__(instance_id, config)
        self.rsi = CandleRSI(period=14)
        self.ema_fast = CandleEMA(period=50)
        self.ema_slow = CandleEMA(period=200)

    async def on_live_candle(self, event: CandleEvent):
        candle_data = event.candle
        self.rsi.add_candle(candle_data)
        self.ema_fast.add_candle(candle_data)
        self.ema_slow.add_candle(candle_data)

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

### 4.6. Placing Orders

When your logic determines it's time to trade, you need to generate a `SignalEvent`. The `ExecutionManager` will listen for this event and handle the order placement with the appropriate broker.

To make this easy, the `BaseStrategy` provides a helper method: `_generate_signal`.

```python
# Inside your on_live_candle method...
if crossover_buy_signal and rsi_oversold:
    await self._generate_signal(
        signal_type="LONG",       # "LONG", "SHORT", "EXIT_LONG", "EXIT_SHORT"
        target_price=event.candle['close']
    )
```

---

## 5. Connecting to Brokers

The framework provides a unified interface for interacting with different exchanges and brokers. The `BrokerFactory` is used to create broker instances, which all share a common interface for placing orders and receiving data.

### 5.1. Broker Factory
The `BrokerFactory` (`src/broker_interface/broker_factory.py`) simplifies the creation of broker instances. It reads the configuration, handles authentication, and returns a ready-to-use broker object. In general, you won't interact with this directly; the `StrategyFactory` uses it to provide the correct broker to your strategy at runtime.

### 5.2. Supported Brokers
The framework supports several brokers, including:
- **Kraken**: For cryptocurrencies.
- **Saxo Bank**: For stocks, options, futures, forex, bonds, ETFs.
- **Interactive Brokers**: For stocks, options, and futures.
- **Binance**: For cryptocurrencies.
- **MockBroker**: A simulated broker for testing without connecting to a live exchange.

#### Broker-Side Normalization and Sorting
- **Futures chains are now always sorted by expiry in the broker.** Strategies should not sort chains themselves.
- **DOM data is normalized in the broker and published as `DOMEvent` with `event_type='depth'`.**

Configuration for each broker is handled via environment variables (`.env` file) and your strategy's YAML file.

---

## 6. Portfolio & Risk Management

The portfolio management system (`src/portfolio/portfolio_manager.py`) provides a unified view of your positions, cash, and risk across all connected brokers and asset classes.

### 6.1. Position Management
You can track your open positions across all accounts. The `PortfolioManager` automatically synchronizes with the brokers to keep your positions up to date.

### 6.2. Cash Management
The system tracks cash balances in multiple currencies and provides tools for currency conversion and managing cash utilization.

### 6.3. Strategy Portfolio Access

All strategies that inherit from `BaseStrategy` automatically get access to portfolio information through `self.portfolio`. This provides a simple, unified interface to query your positions and cash regardless of which broker you're using.

#### Basic Usage in Strategies

```python
class MyStrategy(BaseStrategy):
    async def run(self):
        # Check if you own something
        invested = await self.portfolio.is_invested("AAPL")
        
        # Get position size
        size = await self.portfolio.get_position_size("AAPL")  # returns 0.0 if no position
        
        # Get available cash 
        cash = await self.portfolio.get_available_cash()  # respects 5% reserve by default
        
        # Get portfolio summary
        summary = await self.portfolio.get_portfolio_summary()
```

#### Available Methods

**Position Info:**
- `await self.portfolio.is_invested(symbol)` → bool
- `await self.portfolio.position_exists(symbol)` → bool  
- `await self.portfolio.get_position_size(symbol)` → float
- `await self.portfolio.get_position_value(symbol)` → float

**Cash Info:**
- `await self.portfolio.get_available_cash()` → float (after reserve)
- `await self.portfolio.get_total_cash()` → float (before reserve)
- `await self.portfolio.get_buying_power()` → float

**Advanced:**
- `await self.portfolio.check_margin()` → dict with margin info
- `await self.portfolio.get_currencies()` → list of currencies with balances
- `await self.portfolio.get_options_expiration_dates(symbol)` → list of dates (if broker supports options)
- `await self.portfolio.liquidate_portfolio(emergency=False)` → dict with liquidation results

The PositionManager automatically adapts to your broker - Saxo strategies get options functionality, crypto strategies get multi-currency support, etc. All methods return sensible defaults (0.0, False, empty dict) if something goes wrong.

### 6.4. Risk Management
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

- **`FuturesHelper`**: This is the main interface for fetching futures contract chains and placing orders. Chains are always sorted by expiry in the broker.
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

## 8.1. Three-Tier Democratic AI Voting System

The Quantify framework implements a sophisticated three-tier democratic AI voting system that combines ensemble learning, cross-model voting, and reinforcement learning for robust decision-making.

### 8.1.1. System Architecture

The voting system operates on three tiers:

#### **Tier 1: Ensemble Models (Futures Price Prediction)**
- **Multiple LSTM Models**: Standard, Bidirectional, and Deep architectures
- **Individual Model Votes**: Each LSTM model casts its own vote
- **Ensemble Vote**: Combined prediction from all models with uncertainty quantification
- **Real-time Adaptation**: Models automatically retrain based on performance

#### **Tier 2: Cross-Model Voting (Democratic Process)**
- **All Models Vote**: Order flow, regime detection, anomaly detection, dark pool analysis, AND ensemble models
- **Democratic Process**: Each model gets equal voting rights initially
- **Vote Tallying**: Aggregates all votes by topic with confidence weighting
- **Transparent Decision Making**: Complete visibility into how each model voted

#### **Tier 3: RL Agent (Final Decision Maker)**
- **Learns from All Votes**: Takes the vote tally as input to state vector
- **Makes Final Decision**: Uses learned policy to decide action (buy/sell/hold)
- **Feedback Loop Learning**: Learns from trading outcomes and vote accuracy
- **Adaptive Policy**: Continuously improves decision-making based on results
- **Risk Management**: Considers confidence levels and uncertainty scores

### 8.1.2. How the Voting System Works

The voting system is **fully automatic** and **seamlessly integrated** with the prediction process. Here's how it works:

#### **Automatic Voting Design:**
```python
# ✅ Automatic voting - No client intervention needed
prediction = await futures_client.predict_price(ticker, market_data)
# Votes are automatically cast in the background!
```

**How it works:**
1. **Client makes prediction**: Calls `predict_price()` with market data
2. **Automatic vote casting**: Handler automatically casts votes for ensemble and individual models
3. **Voting system integration**: Votes are stored in the voting system
4. **RL agent access**: RL agent can access all votes for decision making
5. **Transparent process**: Client gets prediction, voting happens seamlessly

#### **Why This Design is Better:**
1. **Zero Client Code**: No additional client code required for voting
2. **Guaranteed Voting**: Every prediction automatically casts votes
3. **Seamless Integration**: Voting is transparent to the client
4. **Reliable**: No risk of forgetting to cast votes
5. **Production Ready**: Works out of the box without manual intervention

### 8.1.3. Implementation Details

The voting system is **automatically integrated** into the prediction process:

```python
class FuturesPricePredictionHandler(BaseHandler):
    async def _handle_predict_price(self, ticker: str, predictor, payload):
        # 1. Make prediction
        prediction = predictor.predict(market_data)
        
        # 2. Automatically cast votes (no client involvement needed)
        auto_cast_enabled = self.config.get('auto_cast_votes', True)
        if auto_cast_enabled:
            await self._auto_cast_votes(ticker, prediction, predictor, market_data)
        
        # 3. Return prediction to client
        return {'success': True, 'prediction': prediction.to_dict()}
    
    async def _auto_cast_votes(self, ticker: str, prediction, predictor, market_data):
        """Automatically cast votes when prediction is made."""
        # Cast ensemble vote
        ensemble_vote = Vote(
            voter_name=f"FuturesPricePrediction_Ensemble_{ticker}",
                            vote_topic=f"trade_decision_{ticker}",
            prediction=prediction.position_flag,
            confidence=prediction.confidence_score
        )
        
        # Cast individual model votes (if multiple models)
        for model_name in predictor.models.keys():
            individual_pred = predictor._get_individual_model_prediction(model_name, market_data)
            if individual_pred:
                individual_vote = Vote(
                    voter_name=f"FuturesPricePrediction_{model_name}_{ticker}",
                    vote_topic=f"trade_decision_{ticker}",
                    prediction=individual_pred['position_flag'],
                    confidence=individual_pred['confidence']
                )
                # Cast vote automatically
                await voting_client.cast_vote(individual_vote)
```

### 8.1.4. Usage Examples

#### **Simple Usage (Automatic Voting):**
```python
# Client just makes prediction - voting happens automatically
prediction = await futures_client.predict_price("BTC-USD", market_data)
print(f"Prediction: {prediction['position_flag']}")
print(f"Confidence: {prediction['confidence_score']}")
# Votes are automatically cast in the background!
```

#### **RL Agent Integration:**
```python
# RL agent automatically accesses all votes for decision making
rl_decision = await rl_client.decide_action({
    'ticker': 'BTC-USD',
    'vote_topic': 'futures_price_direction',
    'market_data': market_data
})

print(f"RL Agent Decision: {rl_decision['action']}")
print(f"Confidence: {rl_decision['confidence']}")
print(f"Reasoning: {rl_decision['reasoning']}")
```

#### **Vote Monitoring (Optional):**
```python
# Monitor votes if needed (optional)
vote_tally = await voting_client.tally_votes(f"trade_decision_{ticker}")
print(f"Total votes: {vote_tally['total_votes']}")
print(f"Futures prediction votes: {len(vote_tally['futures_price_votes'])}")
```

### 8.1.5. Key Benefits

1. **Zero Client Code**: No additional client code required for voting
2. **Automatic Integration**: Votes cast immediately when predictions are made
3. **RL Agent Ready**: Seamless integration with existing RL agent
4. **Transparent Process**: Complete visibility into decision-making
5. **Scalable Architecture**: Easy to add new models to the voting system
6. **Production Ready**: Works out of the box without manual intervention
7. **Risk Management**: Uncertainty quantification and confidence weighting

### 8.1.6. Configuration

The voting system is configured in `config/main.yaml`:

```yaml
ml_service:
  futures_price_prediction:
    # Ensemble configuration
    ensemble_size: 3
    model_types: ['standard', 'bidirectional', 'deep']
    
    # Voting configuration
    voting_config:
      auto_cast_votes: true  # Enable automatic voting
      vote_topic: "trade_decision_{ticker}"  # Dynamic vote topic per ticker
      include_individual_votes: true  # Cast votes from individual ensemble models
    
    # Performance monitoring
    adaptation_threshold: 0.1
    performance_window: 100
```

### 8.1.7. Complete Workflow

Here's the complete workflow from prediction to RL agent decision:

1. **Client makes prediction**:
   ```python
   prediction = await futures_client.predict_price("BTC-USD", market_data)
   ```

2. **Automatic vote casting** (happens in background):
   - Ensemble vote is cast
   - Individual model votes are cast
   - Votes stored in voting system

3. **RL agent accesses votes**:
   ```python
   rl_decision = await rl_client.decide_action({
       'ticker': 'BTC-USD',
       'vote_topic': 'futures_price_direction'
   })
   ```

4. **RL agent makes final decision**:
   - Converts vote tally to state vector
   - Applies learned policy
   - Returns final action (buy/sell/hold)
   - **Learns from feedback**: Trading outcomes improve future decisions

This creates a **truly democratic AI trading system** where multiple intelligent agents vote on decisions, and a reinforcement learning agent learns from their collective wisdom and trading outcomes to continuously improve decision-making.

### 8.1.9. Fully Self-Learning ML Chain

The Quantify framework implements a **complete autonomous self-learning ML chain** where every component learns and adapts:

#### **Multi-Level Autonomous Learning:**

**🔹 Level 1: Individual Model Learning**
- **LSTM Models**: Automatically retrain based on new data and performance
- **Feature Engineering**: Adapts to changing market patterns
- **Model Architecture**: Can switch between different LSTM types based on performance
- **Hyperparameter Optimization**: Self-adjusts learning rates, batch sizes, etc.

**🔹 Level 2: Ensemble Learning**
- **Model Weighting**: Automatically adjusts weights based on individual model performance
- **Ensemble Composition**: Can add/remove models based on effectiveness
- **Uncertainty Quantification**: Learns to measure prediction confidence
- **Adaptive Voting**: Adjusts voting strategy based on market conditions

**🔹 Level 3: Cross-Model Voting Learning**
- **Vote Reliability**: Learns which models are most reliable in different conditions
- **Confidence Weighting**: Adjusts vote importance based on historical accuracy
- **Market Regime Adaptation**: Different voting strategies for different market conditions
- **Model Diversity**: Maintains diverse model types for robust predictions

**🔹 Level 4: RL Agent Learning**
- **Policy Optimization**: Continuously improves decision-making policy
- **Reward Learning**: Adapts reward function based on trading outcomes
- **State Representation**: Learns optimal ways to represent market state
- **Action Selection**: Optimizes action selection based on vote patterns

**🔹 Level 5: System-Level Adaptation**
- **Performance Monitoring**: Tracks accuracy, profit factor, Sharpe ratio
- **Automatic Retraining**: Triggers retraining when performance degrades
- **Resource Optimization**: Manages memory and computational resources
- **Risk Management**: Learns optimal risk parameters

#### **Complete Autonomous Cycle:**

```python
# The system runs this cycle continuously:
while True:
    # 1. Collect market data
    market_data = get_latest_market_data()
    
    # 2. Individual models learn and predict
    for model in ensemble_models:
        model.adapt_to_new_data(market_data)
        prediction = model.predict(market_data)
    
    # 3. Ensemble learns optimal combination
    ensemble.learn_optimal_weights(predictions, historical_accuracy)
    ensemble_prediction = ensemble.combine_predictions(predictions)
    
    # 4. Voting system learns from all models
    voting_system.cast_votes(all_model_predictions)
    voting_system.learn_vote_reliability(historical_outcomes)
    
    # 5. RL agent learns optimal decision policy
    rl_agent.observe_votes(vote_tally)
    action = rl_agent.decide_action(vote_tally)
    
    # 6. Execute trade and observe outcome
    trade_result = execute_trade(action)
    
    # 7. All components learn from outcome
    for model in ensemble_models:
        model.learn_from_outcome(trade_result)
    
    voting_system.learn_from_outcome(trade_result)
    rl_agent.learn_from_outcome(trade_result)
    
    # 8. System adapts overall strategy
    system.adapt_strategy(performance_metrics)
```

#### **Self-Learning Capabilities:**

**🎯 Autonomous Decision Making:**
- No human intervention required for learning
- Self-optimizing parameters and strategies
- Automatic adaptation to market changes

**🧠 Continuous Intelligence:**
- Gets smarter with every trade
- Learns from both successes and failures
- Adapts to new market conditions

**📈 Performance Optimization:**
- Self-monitoring and self-improving
- Automatic retraining when needed
- Resource optimization and management

**🔄 Full Feedback Loop:**
- Every component learns from outcomes
- Cross-component learning and adaptation
- System-wide optimization

This creates a **truly autonomous, self-learning AI trading system** that continuously improves itself at every level without human intervention.

### 8.1.8. Automatic Feedback Loop Learning

The system implements a **fully automatic feedback loop** that eliminates the need for client-side reward handling:

#### **Automatic Learning Process:**

1. **Prediction Tracking**: Every prediction is automatically tracked for feedback
2. **Vote Collection**: RL agent receives votes from all models
3. **Decision Making**: Applies current policy to make trading decision
4. **Action Execution**: Execute trade based on decision
5. **Outcome Capture**: Trade results automatically captured from broker
6. **Reward Calculation**: System automatically calculates reward based on P&L
7. **Automatic Feedback**: Feedback automatically provided to all components
8. **Policy Update**: RL agent updates policy automatically
9. **Model Adaptation**: Ensemble models adapt automatically

#### **Zero Client Intervention Required:**

**❌ Old Way (Client Responsibility):**
```python
# Client had to manually:
1. Track predictions
2. Monitor trade outcomes  
3. Calculate rewards
4. Send feedback to RL agent
5. Update model performance
6. Handle learning cycles
```

**✅ New Way (Fully Automatic):**
```python
# Client only needs to:
1. Make prediction
2. Execute trade
3. That's it! Everything else is automatic
```

#### **Automatic Reward Calculation:**
```python
def _calculate_trade_reward(self, trade_result: Dict[str, Any]) -> float:
    """Calculate reward based on trade outcome."""
    pnl = trade_result.get('pnl', 0.0)
    action = trade_result.get('action', '')
    
    # Base reward on P&L
    if pnl > 0:
        reward = min(1.0, pnl * 10)  # Scale positive P&L to reward
    elif pnl < 0:
        reward = max(-1.0, pnl * 10)  # Scale negative P&L to penalty
    else:
        reward = 0.0
    
    # Adjust reward based on action type
    if action in ['buy', 'sell'] and abs(pnl) > 0.001:  # Significant trade
        reward *= 1.2  # Boost reward for decisive actions
    elif action == 'hold' and abs(pnl) < 0.001:  # Good hold decision
        reward *= 0.8  # Moderate reward for avoiding losses
    
    return reward
```

#### **Automatic Feedback Provision:**
```python
# System automatically provides feedback to:
1. RL Agent: Updates Q-values and policy
2. Ensemble Models: Updates performance metrics
3. Voting System: Learns vote reliability
4. System Level: Adapts overall strategy
```

#### **Example Automatic Learning Cycle:**
```python
# 1. Make prediction (automatically tracked)
prediction = await futures_client.predict_price("BTC-USD", market_data)

# 2. Execute trade based on prediction
action = prediction.rl_decision or prediction.position_flag
trade_result = await execute_trade(action)

# 3. Provide trade outcome (automatic feedback triggered)
await futures_client.provide_automatic_feedback("BTC-USD", {
    'trade_id': 'trade_123',
    'action': action,
    'pnl': 0.025,  # 2.5% profit
    'timestamp': '2024-01-01T10:00:00'
})

# 4. System automatically:
#    - Calculates reward (+0.25)
#    - Updates RL agent policy
#    - Adapts ensemble models
#    - Improves voting system
#    - Triggers model retraining if needed
```

#### **Benefits of Automatic Feedback:**
- **✅ Zero Client Code**: No feedback handling required
- **✅ No Risk of Forgetting**: Automatic tracking ensures no missed feedback
- **✅ Simplified Architecture**: Single function handles all feedback types
- **✅ Clean Code**: No complex multi-function chains
- **✅ Easy Maintenance**: Simple, readable feedback logic
- **✅ Consistent Rewards**: Standardized reward calculation
- **✅ Self-Improving**: Models automatically get better
- **✅ Production Ready**: Reliable and robust
- **✅ Scalable**: Works for any number of trades

---

## 9. Visualization & Dashboards

Understanding your strategy's performance and market data is critical. The `UnifiedVisualizer` (`src/visualization/unified_visualizer.py`) uses Plotly to create rich, interactive charts.

### 9.1. Chart Types
You can plot:
- **Candlestick charts** with trade entry/exit markers.
- **Technical indicators** overlaid on the price or in subplots.
- **Elliott Wave** patterns and Fibonacci levels.
- **Strategy performance**, including equity curves and drawdown.
- **Order Flow/DOM**: Visualize order book depth and synthetic orders (if enabled).

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

### 10.4. Enhanced Logging Features

The framework now includes enhanced logging capabilities:

- **Performance Logging**: Automatic performance metrics tracking with detailed P&L calculations
- **Execution Logging**: Comprehensive order and trade execution logging
- **Real-time Log Viewer**: CLI-based log viewer for monitoring live trading (`src/cli/log_viewer.py`)
- **Structured Logging**: JSON-formatted logs for easier parsing and analysis
- **Header Debugging**: Advanced broker request/response header logging for troubleshooting API issues

---

## 11. Command Line Interface (CLI)

The Quantify framework provides comprehensive command-line interfaces for all major operations. See the dedicated [CLI Manual](cli_manual.md) for complete documentation.

### 11.1. Framework CLI

The main CLI through `main.py` provides:

- **Framework Management**: Start, stop, and monitor the trading framework
- **Strategy Management**: Deploy, start, stop, and restart individual strategies
- **Interactive Mode**: Real-time monitoring and management interface
- **Daemon Mode**: Background execution with proper process management
- **Status Monitoring**: View framework and strategy health in real-time

```bash
# Start the framework
python main.py start

# Deploy a new strategy
python main.py deploy-strategy --config config/strategies/my_strategy.yaml

# Enter interactive mode
python main.py interactive

# Run as daemon
python main.py start --daemon --pidfile /var/run/quantify.pid
```

### 11.2. Backtesting CLI

Comprehensive backtesting capabilities through `python -m backtesting.cli`:

- **Single Strategy Backtests**: Test individual strategies
- **Strategy Comparison**: Compare multiple strategies side-by-side
- **Walk-Forward Optimization**: Optimize parameters over time
- **Parameter Sweeps**: Test multiple parameter combinations
- **Portfolio Analysis**: Multi-strategy portfolio testing
- **Performance Attribution**: Detailed performance analysis

```bash
# Run a single backtest
python -m backtesting.cli single --strategy momentum --symbols AAPL --start-date 2023-01-01 --end-date 2023-12-31

# Compare strategies
python -m backtesting.cli compare --strategies config/strategies.json --symbols AAPL MSFT

# Optimize parameters
python -m backtesting.cli wfo --strategy momentum --parameters config/param_ranges.json
```

### 11.3. Log Viewer CLI

Real-time log monitoring with filtering capabilities:

```bash
# Follow all logs
python src/cli/log_viewer.py --follow

# Filter by strategy
python src/cli/log_viewer.py --strategy "MyStrategy" --follow

# Show only errors
python src/cli/log_viewer.py --level ERROR --follow
```

---

## 12. Broker Integrations

### 12.1. Saxo Bank Integration

The framework includes comprehensive Saxo Bank integration with advanced features:

#### Features:
- **Multi-Asset Support**: Stocks, options, futures, forex, bonds, ETFs
- **Real-time Data**: WebSocket-based streaming market data
- **Options Trading**: Full options chain access and multi-leg strategy support
- **Advanced Order Types**: Support for algorithmic orders and complex strategies
- **Market Data Entitlements**: Proper handling of market data permissions
- **Error Handling**: Comprehensive error handling with correlation tracking
- **DOM Support**: Real-time DOM (order book) data, normalized and published as `DOMEvent` with `event_type='depth'`.
- **Futures Chain Sorting**: All futures chains are sorted by expiry in the broker.

#### Configuration:
```yaml
broker:
  name: "saxo"
  config:
    base_url: "https://gateway.saxobank.com/sim/openapi"
    client_id: "${SAXO_CLIENT_ID}"
    client_secret: "${SAXO_CLIENT_SECRET}"
    refresh_token: "${SAXO_REFRESH_TOKEN}"
```

#### Key Components:
- **SaxoBroker**: Main broker interface with WebSocket streaming
- **SaxoOptions**: Specialized options trading handler
- **Market Data Management**: Subscription-based real-time data
- **Error Tracking**: X-Correlation header tracking for support

#### Options Strategies:
The Saxo integration supports complex options strategies:
- Iron Condors
- Calendar Spreads
- Vertical Spreads
- Straddles and Strangles
- Custom multi-leg strategies

See `examples/options/` for detailed implementation examples.

### 12.2. Market Data Troubleshooting

When experiencing market data access issues:

1. **Check Entitlements**: Verify your account has the required market data subscriptions
2. **Symbol Format**: Use the correct symbol format for each exchange
3. **Market Hours**: Ensure markets are open for real-time data
4. **Headers Logging**: Enhanced logging captures X-Correlation headers for support
5. **Account Type**: Verify you're using the correct account (simulation vs live)

---

## 13. Database Operations and Command Processing

### 13.1. Database Command System

The framework includes a sophisticated command processing system that allows external control of strategies:

```python
from src.database.command_client import CommandClient

# Submit a command to stop a strategy
await client.submit_command(
    command='stop-strategy',
    strategy_id='MyStrategy_001',
    parameters={'reason': 'manual_stop'}
)
```

### 13.2. Supported Commands

- **deploy-strategy**: Deploy a new strategy from configuration
- **start-strategy**: Start a paused strategy
- **stop-strategy**: Stop a running strategy
- **restart-strategy**: Restart a strategy with new parameters
- **update-params**: Update strategy parameters without restart

### 13.3. Command Processing

Commands are processed asynchronously by the Strategy Factory:
- Commands are queued in the database
- Processed in order with proper error handling
- Results and errors are logged and stored
- Failed commands can be retried

---

## 14. Performance Monitoring and Analytics

### 14.1. Real-time Performance Tracking

The framework provides comprehensive performance monitoring:

- **P&L Tracking**: Real-time profit and loss calculations
- **Position Monitoring**: Live position updates across all brokers
- **Risk Metrics**: Real-time risk calculations and limit monitoring
- **Performance Attribution**: Detailed breakdown of returns by strategy and asset

### 14.2. Interactive Performance Dashboard

Access through the interactive CLI mode:

```bash
python main.py interactive
```

Features:
- Live strategy performance metrics
- Real-time position monitoring
- Interactive command execution
- Log viewing and filtering
- Health status monitoring

### 14.3. Performance Data Export

Export performance data for external analysis:
- CSV exports for spreadsheet analysis
- JSON exports for programmatic access
- Database queries for custom reporting
- Integration with external monitoring systems

---

## 15. Troubleshooting and Support

### 15.1. Common Issues

**Market Data Access:**
- Verify broker entitlements and permissions
- Check symbol format and exchange codes
- Ensure proper authentication credentials
- Review market hours and data availability

**Strategy Deployment:**
- Validate YAML configuration syntax
- Check required parameters and dependencies
- Verify broker connectivity
- Review log files for specific errors

**Performance Issues:**
- Monitor resource usage (CPU, memory)
- Check database performance and optimization
- Review network connectivity and latency
- Optimize strategy logic and data handling

### 15.2. Debugging Tools

- **Enhanced Logging**: Comprehensive logging with correlation tracking
- **Log Viewer**: Real-time log monitoring and filtering
- **Interactive Mode**: Live debugging and command execution
- **Performance Profiling**: Built-in performance monitoring
- **Database Inspector**: Direct database query capabilities

### 15.3. Getting Support

When contacting broker support or debugging issues:
- Include X-Correlation headers from logs
- Provide specific timestamps and error messages
- Include relevant configuration details
- Use the enhanced logging for detailed error tracking

---

## 16. Best Practices and Recommendations

### 16.1. Development Workflow

1. **Use Configuration Templates**: Generate configurations using the CLI tools
2. **Validate Before Deploy**: Always validate configurations before deployment
3. **Test with Backtesting**: Thoroughly test strategies before live deployment
4. **Monitor Performance**: Use real-time monitoring during live trading
5. **Implement Proper Risk Management**: Define and enforce risk limits
6. **Use Version Control**: Track configuration and strategy changes
7. **Regular Monitoring**: Use the interactive CLI for ongoing monitoring

### 16.2. Production Deployment

1. **Daemon Mode**: Run the framework as a daemon in production
2. **Process Management**: Use proper PID files and signal handling
3. **Log Rotation**: Configure appropriate log rotation and retention
4. **Monitoring Setup**: Implement external monitoring and alerting
5. **Backup Strategy**: Regular database and configuration backups
6. **Disaster Recovery**: Plan for system failure scenarios

### 16.3. Security Considerations

1. **API Key Management**: Use environment variables for sensitive data
2. **Access Control**: Limit database and system access
3. **Network Security**: Secure network connections and API endpoints
4. **Audit Logging**: Maintain comprehensive audit trails
5. **Regular Updates**: Keep dependencies and systems updated

---

## 17. Additional Resources

### 17.1. Documentation

- **[CLI Manual](cli_manual.md)**: Comprehensive CLI reference
- **[Options Framework Manual](options_framework_manual.md)**: Options trading guide
- **[Strategy Guide](strategy_guide.md)**: Strategy development reference
- **[Brokers Manual](brokers.md)**: Broker integration guide
- **[Configuration Guide](configuration.md)**: Configuration reference

### 17.2. Examples

The `examples/` directory contains:
- Options strategy implementations
- Futures trading examples
- Machine learning strategy examples
- Backtesting workflow examples
- Configuration templates

### 17.3. Support Resources

- Framework documentation in `docs/`
- Example configurations in `config/`
- Test cases for reference implementations
- Community discussions and support forums

This master manual provides a comprehensive overview of the Quantify Trading Framework. For specific implementation details, refer to the specialized manuals and example code in the framework repository. 