# Technical Indicators

This document provides an overview of the technical indicators available in the Quantify framework.

## Available Indicators

### Trend Indicators

1. **Moving Averages**
   - Simple Moving Average (SMA)
   - Exponential Moving Average (EMA)
   - Projected Moving Average (PMA)
   - Projected Moving Average (ProjectedMovingAverage)

2. **Directional Indicators**
   - Average Directional Index (ADX)
   - Aroon Indicator
   - True Strength Index (TSI)
   - Parabolic SAR (PSAR)

3. **Channel Indicators**
   - Bollinger Bands
   - Keltner Channels
   - Donchian Channels

### Momentum Indicators

1. **Oscillators**
   - Relative Strength Index (RSI)
   - Stochastic Oscillator
   - Williams %R
   - Money Flow Index (MFI)

2. **Rate of Change**
   - Momentum
   - Rate of Change (ROC)
   - Williams %R

### Volume Indicators

1. **Volume Analysis**
   - On-Balance Volume (OBV)
   - Volume Weighted Average Price (VWAP)
   - Volume Indicators (Multiple)

### Volatility Indicators

1. **Volatility Measures**
   - Average True Range (ATR)
   - Volatility Indicator

### Japanese Candlestick Indicators

1. **Ichimoku Cloud**
   - Complete Ichimoku system
   - Support/Resistance levels
   - Trend direction

## Usage in Strategies

All indicators work with candle data structures and are designed to be used in trading strategies.

### Example Usage

```python
from src.indicators.candle_indicators import (
    RSI, MACD, BollingerBands,
    StochasticOscillator, ATR, IchimokuCloud,
    EMA, SMA, WilliamsR, Momentum,
    ROC, TSI, ADX, Aroon,
    KeltnerChannels, DonchianChannels,
    VWAP, OBV, MFI, Volatility,
    VolumeIndicators, PMA, ProjectedMovingAverage
)

class YourStrategy(BaseStrategy):
    def __init__(self, config_path: str):
        super().__init__(config_path)
        
        # Initialize indicators
        self.rsi = RSI(period=14)
        self.macd = MACD(fast_period=12, slow_period=26, signal_period=9)
        self.bb = BollingerBands(period=20, num_std=2.0)
        self.stoch = StochasticOscillator(k_period=14, d_period=3, slowing=3)
        self.atr = ATR(period=14)
        self.ichimoku = IchimokuCloud()
        self.ema = EMA(period=20)
        self.sma = SMA(period=20)
        self.williams_r = WilliamsR(period=14)
        self.momentum = Momentum(period=14)
        self.roc = ROC(period=14)
        self.tsi = TSI(first_period=25, second_period=13, signal_period=13)
        self.adx = ADX(period=14)
        self.aroon = Aroon(period=25)
        self.keltner = KeltnerChannels(ema_period=20, atr_period=10, atr_multiplier=2.0)
        self.donchian = DonchianChannels(period=20)
        self.vwap = VWAP()
        self.obv = OBV()
        self.mfi = MFI(period=14)
        self.volatility = Volatility(period=14)
        self.volume = VolumeIndicators(period=20)
        self.pma = PMA(period=20)
        self.projected_ma = ProjectedMovingAverage(period=20)
```

## Indicator Parameters

Each indicator has its own set of parameters that can be customized. Here are the common parameters:

- `period`: The lookback period for calculations
- `fast_period`: The shorter period for dual-period indicators
- `slow_period`: The longer period for dual-period indicators
- `signal_period`: The period for signal line calculations
- `num_std`: Number of standard deviations for band calculations
- `k_period`: The %K period for stochastic calculations
- `d_period`: The %D period for stochastic calculations
- `slowing`: The slowing period for stochastic calculations
- `first_period`: First smoothing period for TSI
- `second_period`: Second smoothing period for TSI
- `ema_period`: Period for EMA calculations
- `atr_period`: Period for ATR calculations
- `atr_multiplier`: Multiplier for ATR in Keltner Channels

## Best Practices

1. **Parameter Selection**
   - Use longer periods for longer timeframes
   - Adjust parameters based on market volatility
   - Test different parameter combinations

2. **Indicator Combinations**
   - Combine trend and momentum indicators
   - Use volume indicators to confirm signals
   - Consider market conditions when selecting indicators

3. **Signal Confirmation**
   - Wait for multiple indicator confirmations
   - Consider price action and market structure
   - Use volume to validate signals

## Performance Considerations

1. **Calculation Efficiency**
   - Indicators are optimized for performance
   - Calculations are done incrementally
   - Memory usage is minimized

2. **Real-time Updates**
   - Indicators update with each new candle
   - Historical data is maintained efficiently
   - Predictions are available immediately

## Error Handling

All indicators include proper error handling:
- Invalid parameters are caught and logged
- Missing data is handled gracefully
- Edge cases are properly managed

## Future Enhancements

Planned improvements:
1. Additional indicator types
2. Machine learning integration
3. Custom indicator creation
4. Performance optimizations
5. More documentation and examples

## Regime Detector

The `RegimeDetector` is a powerful tool for identifying the current market regime, which can be crucial for selecting the appropriate trading strategy. It is designed to be generic and can be used by any strategy that provides the necessary market metrics.

### Implementation

The `RegimeDetector` is located in `src.technical_analysis.regime_detector.py`. It analyzes a combination of volatility and trend indicators to classify the market into one of the following states:

- **volatile**: The market is experiencing high volatility, typically identified by a normalized ATR value exceeding a defined threshold.
- **trending_up**: The market is in a clear uptrend, identified by an ADX value above a trend threshold and a fast EMA above a slow EMA.
- **trending_down**: The market is in a clear downtrend, with ADX above the threshold and a fast EMA below a slow EMA.
- **ranging**: The market is not exhibiting a strong trend or high volatility.

### Usage

To use the `RegimeDetector`, first instantiate it with a configuration dictionary. Then, call the `detect_regime` method with a dictionary of market metrics.

#### Initialization

```python
from src.technical_analysis.regime_detector import RegimeDetector

# Configuration for the detector
detector_config = {
    'regime_window': 100,
    'volatility_threshold': 0.02,
    'trend_threshold': 25  # ADX threshold
}

regime_detector = RegimeDetector(detector_config)
```

#### Detecting the Regime

The `detect_regime` method expects a single dictionary containing all the necessary metrics.

```python
metrics = {
    "candle_buffer": self.candle_buffer,
    "atr": self.atr.get_value(),
    "ema_fast": self.ema_fast.get_value(),
    "ema_slow": self.ema_slow.get_value(),
    "adx": self.adx.get_value(),
    "last_close": self.last_price
}

current_regime = await regime_detector.detect_regime(metrics)
```

### Configuration Parameters

| Parameter              | Type  | Default | Description                                                                                             |
| ---------------------- | ----- | ------- | ------------------------------------------------------------------------------------------------------- |
| `regime_window`        | `int` | `100`   | The number of candles to consider for trend detection.                                                  |
| `volatility_threshold` | `float` | `0.02`  | The normalized ATR threshold to identify a volatile market.                                             |
| `trend_threshold`      | `float` | `25`    | The ADX value above which a market is considered to be trending.                                        |

### Required Metrics

The `detect_regime` method requires the following keys in its `metrics` dictionary:

| Key             | Type                  | Description                                                                                             |
| --------------- | --------------------- | ------------------------------------------------------------------------------------------------------- |
| `candle_buffer` | `List[dict]`          | A list of candle data dictionaries.                                                                     |
| `atr`           | `Optional[float]`     | The current ATR value.                                                                                  |
| `adx`           | `Optional[float]`     | The current ADX value.                                                                                  |
| `ema_fast`      | `Optional[float]`     | The value of the fast exponential moving average.                                                       |
| `ema_slow`      | `Optional[float]`     | The value of the slow exponential moving average.                                                       |
| `last_close`    | `Optional[float]`     | The most recent closing price. If not provided, it will be inferred from the `candle_buffer`.           |

To use the indicators, simply import the desired class from `src.technical_analysis.technical_indicators` and instantiate it with the required parameters. The `update` method should be called with new market data, and the `get_value` or `get_values` methods can be used to retrieve the calculated indicator values. 