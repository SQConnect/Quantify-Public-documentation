# Candle Management Manual

## 1. Introduction

This manual provides a comprehensive guide to working with candle data in the Quantify trading framework. Candles are the cornerstone of most trading strategies, and this framework is designed to make accessing and managing them simple, efficient, and flexible.

The core of the system is the **`CandleManager`**, a centralized service that handles all candle creation, buffering, and resampling. This powerful component ensures that all strategies have access to a consistent, single source of truth for market data, eliminating data duplication and simplifying strategy logic.

## 2. Key Concepts

### The Centralized `CandleManager`

You will never need to interact with the `CandleManager` directly. It works silently in the background, managed by the framework. Here’s what it does for you:

- **Single Source of Truth**: All candle data for all symbols and timeframes is stored once in an efficient, in-memory buffer.
- **Automatic Candle Creation**: It ingests live tick or OHLC data from the broker and automatically builds candles for your desired timeframes.
- **On-the-Fly Resampling**: It can create any higher timeframe from a base timeframe. For example, if you need '5m' and '1h' candles, you only need to subscribe to '1m' data, and the `CandleManager` will build the larger timeframes for you in real-time.
- **Automatic Warm-Up**: When your strategy starts, the framework automatically pre-loads the `CandleManager`'s buffers with historical data, ensuring your indicators have enough data to be accurate from the very first trade.

### Stateless Strategies

Because the `CandleManager` handles all data storage, your strategies can remain **stateless** regarding candle data. You no longer need to maintain local lists or buffers of candles. You can simply request the data you need, when you need it.

## 3. Timeframes: Unlimited Flexibility

The framework offers complete flexibility for timeframes. While standard timeframes like '1m', '5m', '1h', '1d' are common, you are **not** restricted to them. You can define any custom timeframe in your strategy's configuration.

For example, if you want to work with **14-minute candles**, you simply specify it in your config. The `CandleManager` will automatically generate them for you from a smaller base timeframe.

```yaml
# In your strategy's config.yaml
timeframes: ['1m', '14m', '1h'] # '14m' is a valid custom timeframe
```

The framework will determine the smallest base timeframe required ('1m' in this case) and automatically subscribe to it, resampling to all other requested timeframes.

## 4. Accessing Candle Data in Your Strategy

Your strategy, which should inherit from `BaseStrategy`, has built-in helper methods to access the `CandleManager`'s data buffers. You should **never** store candles in your strategy instance.

### Key Helper Methods

- `self.get_candles(symbol, timeframe, lookback)`: Fetches a list of historical candles.
- `self.get_current_candle(symbol, timeframe)`: Fetches the most recent, still-forming candle.
- `self.get_candle_dataframe(symbol, timeframe, lookback)`: Fetches the candles as a pandas DataFrame, ready for analysis with libraries like TA-Lib or pandas_ta.

## 5. Practical Example: A Simple PMA Strategy

Let's look at a realistic example. Below is a complete `PMAStrategy` that trades based on a price crossover of a Projected Moving Average. Notice that it does **not** have an `__init__` method and does **not** store any candles locally.

### The Strategy Code

```python
# src/strategies/my_pma_strategy.py

from ..strategy_framework import BaseStrategy, CandleEvent, Signal
from ..indicators import ProjectedMovingAverage

class PMAStrategy(BaseStrategy):
    """
    A simple strategy that uses a Projected Moving Average (PMA)
    to make trading decisions. It inherits all the necessary components
    from BaseStrategy.
    """
    
    async def on_live_candle(self, event: CandleEvent):
        """This is the main logic loop for the strategy."""
        
        # We get the symbol, timeframe, and PMA period from the config file.
        # The framework guarantees we only receive events for the subscribed symbol/timeframe.
        symbol = self.config.get('symbol')
        timeframe = self.config.get('timeframe')
        pma_period = self.config.get('pma_period', 20)

        # Fetch the last `pma_period` candles from the CandleManager
        candles = self.get_candles(symbol, timeframe, lookback=pma_period)

        if len(candles) < pma_period:
            self.logger.info(f"Warming up... Need {pma_period} candles, have {len(candles)}.")
            return

        # Calculate the PMA value
        closes = [float(c.close) for c in candles]
        pma_indicator = ProjectedMovingAverage(period=pma_period)
        pma_value = pma_indicator.calculate(closes)

        if pma_value is None:
            return

        # Basic Crossover Logic
        current_price = float(event.candle.close)
        if current_price > pma_value:
            await self.place_order(
                symbol=symbol,
                side='buy',
                quantity=self.config.get('order_size', 1.0)
            )
        elif current_price < pma_value:
            await self.place_order(
                symbol=symbol,
                side='sell',
                quantity=self.config.get('order_size', 1.0)
            )
```

## 6. Buffer Preheating: Instant Strategy Start

One of the most powerful features of the centralized candle management system is **buffer preheating**. This allows your strategies to start trading immediately without waiting for live candles to accumulate.

### What is Preheating?

Preheating fills the `CandleManager`'s buffers with historical data from your broker before your strategy starts trading. This means:

- **No Warm-Up Wait**: Your strategy can start trading immediately with full historical context
- **Accurate Indicators**: All technical indicators have sufficient data to be accurate from the first candle
- **Backtesting Consistency**: The same historical data is available for both live trading and backtesting

### Automatic Preheating

You can enable automatic preheating in your strategy configuration:

```yaml
# In your strategy's config.yaml
strategy_id: "my_pma_strategy"
symbols: ["BTC/USD"]
timeframes: ["1m", "5m", "1h"]
auto_preheat: true  # Enable automatic preheating
preheat_lookback: 1000  # Number of candles to fetch per timeframe
```

When `auto_preheat: true` is set, the framework will automatically fetch historical data during strategy initialization.

### Manual Preheating

You can also preheat buffers manually using the command line:

```bash
# Preheat a specific symbol with multiple timeframes
python main.py preheat --symbol "BTC/USD" --timeframes 1m 5m 15m 1h --lookback 1000

# Preheat with default timeframes (1m, 5m, 15m, 1h)
python main.py preheat --symbol "ETH/USD" --lookback 500
```

### Programmatic Preheating

Within your strategy, you can preheat buffers programmatically:

```python
class MyStrategy(BaseStrategy):
    async def on_initialize(self):
        """Custom initialization hook."""
        
        # Preheat all strategy buffers
        results = await self.preheat_buffers(lookback_periods=1000)
        
        # Preheat specific symbol
        symbol_results = await self.preheat_symbol("BTC/USD", lookback_periods=500)
        
        # Preheat specific timeframe across all symbols
        timeframe_results = await self.preheat_timeframe("1h", lookback_periods=200)
        
        # Check if we have sufficient data
        if self.has_sufficient_data(min_candles=100):
            self.logger.info("Strategy has sufficient historical data to start trading")
        else:
            self.logger.info("Strategy will wait for more data to accumulate")
```

### Buffer Status Monitoring

You can monitor the status of your candle buffers:

```python
# Get status of all buffers
status = self.get_buffer_status()

# Check specific symbol/timeframe
btc_status = status.get("BTC/USD", {}).get("1h", {})
print(f"BTC/USD 1h buffer: {btc_status['buffer_utilization']} candles")
print(f"Time range: {btc_status['oldest_candle']} to {btc_status['newest_candle']}")
```

## 7. Best Practices

### Strategy Design

1. **Stay Stateless**: Never store candles in your strategy instance. Always use `self.get_candles()` to fetch data on-demand.

2. **Use Configuration**: Define your symbols and timeframes in your strategy's configuration file, not in code.

3. **Enable Preheating**: Use `auto_preheat: true` in your strategy config to ensure immediate trading capability.

4. **Check Data Sufficiency**: Use `self.has_sufficient_data()` to verify you have enough historical data before making trading decisions.

### Performance Considerations

1. **Efficient Lookbacks**: Only request the number of candles you actually need. Don't fetch 1000 candles if your indicator only needs 20.

2. **Reuse Data**: If you need the same data multiple times in a single candle event, fetch it once and reuse it.

3. **Monitor Buffer Status**: Use the buffer status methods to monitor memory usage and data freshness.

### Configuration Examples

```yaml
# Minimal configuration
strategy_id: "simple_strategy"
symbols: ["BTC/USD"]
timeframes: ["1m"]
auto_preheat: true

# Advanced configuration
strategy_id: "multi_timeframe_strategy"
symbols: ["BTC/USD", "ETH/USD"]
timeframes: ["1m", "5m", "15m", "1h", "4h"]
auto_preheat: true
preheat_lookback: 2000
min_candles: 100
```

## 8. Complete Example: Preheated PMA Strategy

Here's a complete example showing how to create a strategy that uses preheating for immediate trading capability.

### Strategy Configuration

```yaml
# config/strategies/preheated_pma_strategy.yaml

strategy_id: "preheated_pma_strategy"
strategy_class: "PreheatedPMAStrategy"
strategy_path: "src.strategies.preheated_pma_strategy"

# Data configuration
symbols: ["BTC/USD"]
timeframes: ["1m", "5m", "15m"]

# Preheating configuration
auto_preheat: true
preheat_lookback: 1000
min_candles: 50

# Strategy parameters
symbol: "BTC/USD"
primary_timeframe: "5m"
pma_period: 20
order_size: 0.01
```

### Strategy Implementation

```python
# src/strategies/preheated_pma_strategy.py

from ..strategy_framework import BaseStrategy, CandleEvent, Signal
from ..indicators import ProjectedMovingAverage

class PreheatedPMAStrategy(BaseStrategy):
    """
    A PMA strategy that uses preheating for immediate trading capability.
    """
    
    async def on_initialize(self):
        """Custom initialization with preheating verification."""
        
        # Check if we have sufficient data after auto-preheating
        if self.has_sufficient_data():
            self.logger.info("✅ Strategy has sufficient historical data - ready to trade immediately!")
        else:
            self.logger.info("⚠️ Strategy will wait for more data to accumulate")
            
        # Get buffer status for monitoring
        status = self.get_buffer_status()
        for symbol in self.symbols:
            for timeframe in self.timeframes:
                buffer_info = status.get(symbol, {}).get(timeframe, {})
                self.logger.info(f"Buffer {symbol}/{timeframe}: {buffer_info.get('buffer_utilization', '0/0')} candles")
    
    async def on_live_candle(self, event: CandleEvent):
        """Main trading logic - called immediately after preheating."""
        
        symbol = self.config.get('symbol')
        primary_timeframe = self.config.get('primary_timeframe', '5m')
        pma_period = self.config.get('pma_period', 20)
        
        # Only process events for our primary timeframe
        if event.timeframe != primary_timeframe:
            return
            
        # Fetch historical data (now immediately available due to preheating)
        candles = self.get_candles(symbol, primary_timeframe, lookback=pma_period)
        
        if len(candles) < pma_period:
            self.logger.warning(f"Insufficient data: {len(candles)}/{pma_period} candles")
            return
            
        # Calculate PMA
        closes = [float(c.close) for c in candles]
        pma_indicator = ProjectedMovingAverage(period=pma_period)
        pma_value = pma_indicator.calculate(closes)
        
        if pma_value is None:
            return
            
        # Trading logic
        current_price = float(event.candle.close)
        order_size = self.config.get('order_size', 0.01)
        
        if current_price > pma_value:
            self.logger.info(f"BUY signal: Price {current_price} > PMA {pma_value:.2f}")
            await self.place_tracked_order(
                symbol=symbol,
                order_type='market',
                side='buy',
                quantity=order_size
            )
        elif current_price < pma_value:
            self.logger.info(f"SELL signal: Price {current_price} < PMA {pma_value:.2f}")
            await self.place_tracked_order(
                symbol=symbol,
                order_type='market',
                side='sell',
                quantity=order_size
            )
```

### Usage

1. **Deploy the strategy**:
   ```bash
   python main.py deploy-strategy --config config/strategies/preheated_pma_strategy.yaml
   ```

2. **Start the framework**:
   ```bash
   python main.py start
   ```

3. **Monitor buffer status**:
   ```bash
   python main.py preheat --symbol "BTC/USD" --timeframes 1m 5m 15m --lookback 100
   ```

The strategy will start trading immediately with full historical context, no warm-up period required! 