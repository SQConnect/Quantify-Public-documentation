# Technical Indicators

This document provides an overview of the technical indicators available in the Quantify framework.

## Available Indicators

### Trend Indicators

1. **Moving Averages**
   - Simple Moving Average (SMA)
   - Exponential Moving Average (EMA)
   - Projected Moving Average (PMA)

2. **Directional Indicators**
   - Average Directional Index (ADX)
   - Aroon Indicator
   - True Strength Index (TSI)

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

All indicators are available in two forms:
1. Base indicators that work with raw price data
2. Candle indicators that work with candle data structures

### Example Usage

```python
from src.indicators.candle_indicators import (
    CandleRSI, CandleMACD, CandleBollingerBands,
    CandleStochasticOscillator, CandleATR, CandleIchimokuCloud,
    CandleEMA, CandleWilliamsR, CandleMomentum,
    CandleROC, CandleTSI, CandleADX, CandleAroon,
    CandleKeltnerChannels, CandleDonchianChannels,
    CandleVWAP, CandleOBV, CandleMFI, CandleVolatility,
    CandleVolumeIndicators, CandlePMA
)

class YourStrategy(BaseStrategy):
    def __init__(self, config_path: str):
        super().__init__(config_path)
        
        # Initialize indicators
        self.rsi = CandleRSI(period=14)
        self.macd = CandleMACD(fast_period=12, slow_period=26, signal_period=9)
        self.bb = CandleBollingerBands(period=20, num_std=2.0)
        self.stoch = CandleStochasticOscillator(k_period=14, d_period=3, slowing=3)
        self.atr = CandleATR(period=14)
        self.ichimoku = CandleIchimokuCloud()
        self.ema = CandleEMA(period=20)
        self.williams_r = CandleWilliamsR(period=14)
        self.momentum = CandleMomentum(period=14)
        self.roc = CandleROC(period=14)
        self.tsi = CandleTSI(first_period=25, second_period=13, signal_period=13)
        self.adx = CandleADX(period=14)
        self.aroon = CandleAroon(period=25)
        self.keltner = CandleKeltnerChannels(ema_period=20, atr_period=10, atr_multiplier=2.0)
        self.donchian = CandleDonchianChannels(period=20)
        self.vwap = CandleVWAP()
        self.obv = CandleOBV()
        self.mfi = CandleMFI(period=14)
        self.volatility = CandleVolatility(period=14)
        self.volume = CandleVolumeIndicators(period=20)
        self.pma = CandlePMA(period=20)
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