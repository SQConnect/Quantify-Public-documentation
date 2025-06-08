# Pattern Recognition Library Documentation

## Overview
The Pattern Recognition library provides comprehensive technical analysis capabilities for detecting various chart patterns and candlestick formations in financial market data. It is designed to work with OHLCV (Open, High, Low, Close, Volume) data and provides confidence scores for each detected pattern.

## Features

### Chart Patterns
1. **Head and Shoulders**
   - Classic reversal pattern
   - Detects both regular and inverse formations
   - Includes neckline calculation
   - Confidence based on symmetry and volume profile

2. **Double Top/Bottom**
   - Reversal patterns
   - Validates price level consistency
   - Includes neckline calculation
   - Confidence based on pattern completion and volume

3. **Breakouts**
   - Detects price breakouts from consolidation
   - Configurable threshold for breakout confirmation
   - Volume confirmation included
   - Direction-specific signals (up/down)

4. **Triangle Patterns**
   - Ascending Triangle
   - Descending Triangle
   - Symmetrical Triangle
   - Trendline-based detection
   - Volume profile analysis

5. **Wedge Patterns**
   - Rising Wedge
   - Falling Wedge
   - Slope-based detection
   - Volume confirmation

6. **Flag Patterns**
   - Bull Flag
   - Bear Flag
   - Trend context validation
   - Volume profile analysis

### Candlestick Patterns
1. **Doji**
   - Indecision pattern
   - Small body relative to range
   - Additional confirmation required
   - Position-based significance

2. **Hammer**
   - Bullish reversal pattern
   - Long lower shadow
   - Small upper shadow
   - Small body

3. **Hanging Man**
   - Bearish reversal pattern
   - Long lower shadow
   - Small upper shadow
   - Small body
   - Upper range position

## Usage

### Basic Usage
```python
from src.technical_analysis.pattern_recognition import PatternRecognition

# Initialize the pattern recognition engine
pattern_recognition = PatternRecognition(
    min_pattern_bars=5,
    max_pattern_bars=100
)

# Detect patterns in your data
patterns = pattern_recognition.detect_candlestick_patterns(data)
```

### Pattern Detection Methods
```python
# Chart Patterns
head_and_shoulders = pattern_recognition.detect_head_and_shoulders(data)
double_tops = pattern_recognition.detect_double_top(data)
breakouts = pattern_recognition.detect_breakouts(data)
triangles = pattern_recognition.detect_triangles(data)
wedges = pattern_recognition.detect_wedges(data)
flags = pattern_recognition.detect_flags(data)

# Candlestick Patterns
candlestick_patterns = pattern_recognition.detect_candlestick_patterns(data)
```

### Pattern Result Structure
Each detected pattern returns a `PatternResult` object with the following attributes:
```python
@dataclass
class PatternResult:
    pattern_type: str          # Type of pattern detected
    confidence: float          # Confidence score (0-1)
    start_idx: int            # Start index in data
    end_idx: int              # End index in data
    price_levels: Dict[str, float]  # Key price levels
    volume_profile: Optional[Dict[str, float]]  # Volume analysis
    additional_metrics: Optional[Dict[str, Any]]  # Additional pattern metrics
```

## Confidence Calculation

### Factors Considered
1. **Volume Confirmation**
   - Volume trend
   - Volume ratio to average
   - Volume profile consistency

2. **Price Action**
   - Pattern completion
   - Price level significance
   - Trend context

3. **Pattern-Specific Metrics**
   - Symmetry
   - Trendline quality
   - Support/Resistance levels

### Confidence Scores
- Range: 0.0 to 1.0
- Higher scores indicate stronger pattern formation
- Multiple factors weighted based on pattern type
- Minimum confidence threshold configurable

## Integration with Strategy Framework

The pattern recognition library is designed to work seamlessly with the strategy framework:

```python
from src.strategy_framework.pattern_based_strategy import PatternBasedStrategy

# Initialize strategy with pattern recognition
strategy = PatternBasedStrategy(
    broker=broker,
    symbols=['BTC/USD'],
    timeframe='1h',
    initial_capital=Decimal('10000')
)

# Pattern types available in strategy
strategy.parameters['pattern_types'] = [
    # Chart patterns
    'head_and_shoulders', 'inverse_head_and_shoulders',
    'double_top', 'double_bottom', 'breakout',
    'triangle_ascending', 'triangle_descending', 'triangle_symmetrical',
    'wedge_rising', 'wedge_falling',
    'flag_bull', 'flag_bear',
    # Candlestick patterns
    'doji', 'hammer', 'hanging_man'
]
```

## Best Practices

1. **Data Quality**
   - Use clean, normalized OHLCV data
   - Ensure sufficient historical data for pattern detection
   - Consider data frequency and timeframe

2. **Pattern Confirmation**
   - Use multiple timeframes for confirmation
   - Consider market context and trend
   - Validate with volume analysis

3. **Risk Management**
   - Use confidence scores for position sizing
   - Implement stop-loss and take-profit levels
   - Consider pattern-specific risk parameters

4. **Performance Optimization**
   - Adjust min/max pattern bars based on timeframe
   - Use appropriate confidence thresholds
   - Consider computational requirements

## Configuration Parameters

### Pattern Recognition Parameters
```python
pattern_recognition = PatternRecognition(
    min_pattern_bars=5,      # Minimum bars for pattern formation
    max_pattern_bars=100,    # Maximum bars for pattern detection
    doji_threshold=0.1,      # Maximum body size for Doji
    hammer_threshold=0.3     # Minimum shadow size for Hammer
)
```

### Strategy Parameters
```python
strategy.parameters = {
    'min_confidence': Decimal('0.7'),    # Minimum pattern confidence
    'stop_loss_pct': Decimal('0.02'),    # Stop loss percentage
    'take_profit_pct': Decimal('0.04'),  # Take profit percentage
    'max_position_size': Decimal('0.1')  # Maximum position size
}
```

## Contributing

To add new patterns or improve existing ones:

1. Add pattern detection method in `PatternRecognition` class
2. Implement confidence calculation
3. Add pattern validation criteria
4. Update documentation
5. Add unit tests

## License
This library is part of the Quantify project and is subject to its licensing terms. 