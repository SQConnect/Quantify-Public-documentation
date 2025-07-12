# Order Flow Analysis System

## Overview

The Order Flow Analysis System is a sophisticated machine learning-based approach to analyzing market microstructure and predicting order flow patterns. It operates as a **pure listener/analyzer strategy** that collects rich market data, trains ML models, and provides insights without executing trades.

## Key Features

- **Enhanced Market Data**: Real-time tick, quote, depth, and OHLC data
- **ML-Powered Analysis**: XGBoost-based order flow pattern detection
- **Pure Listener Mode**: No trading execution, focused on analysis and ML training
- **Real-time Training**: Continuous model improvement with new market data
- **Multi-pattern Detection**: Large orders, icebergs, cascades, accumulation/distribution

## Architecture

```
Market Data Sources → Event Processing → Feature Engineering → ML Training → Pattern Detection
     ↓                      ↓                    ↓                    ↓                    ↓
  Kraken WebSocket    Event Handlers      Feature Builder      OrderFlowAnalyzer    Predictions
  (tick/quote/depth)  (tick/quote/depth)  (technical +        (XGBoost models)     (flow types)
                      (OHLC events)       microstructure)     (classification +    (impact +
                                                                 regression)        timing)
```

## Market Data Subscriptions

### 1. Tick Data (Trade Executions)
- **Source**: Kraken WebSocket `trade` channel
- **Data**: Price, volume, timestamp, side, order type
- **Usage**: Volume spike detection, momentum analysis, trade clustering

### 2. Quote Data (Bid/Ask Updates)
- **Source**: Kraken WebSocket `quote` channel  
- **Data**: Bid/ask prices, sizes, spread analysis
- **Usage**: Spread anomaly detection, size imbalance analysis

### 3. Depth Data (Order Book)
- **Source**: Kraken WebSocket `book` channel
- **Data**: Full order book levels, bid/ask stacks
- **Usage**: Large order detection, order book imbalance, market depth

### 4. OHLC Data (Candlesticks)
- **Source**: Traditional candle manager
- **Data**: Open, high, low, close, volume
- **Usage**: Technical analysis, price patterns, volatility

## Order Flow Patterns

### Detected Patterns

1. **Large Buy/Sell Orders**
   - Threshold-based detection
   - Immediate price impact prediction
   - Short execution time (1-3 minutes)

2. **Iceberg Orders**
   - Pattern recognition in order flow
   - Large orders split into smaller pieces
   - Medium execution time (10-20 minutes)

3. **Stop Loss/Take Profit Cascades**
   - Cascade detection algorithms
   - High volatility prediction
   - Very short execution time (30 seconds - 2 minutes)

4. **Liquidation Events**
   - Forced selling patterns
   - Extreme price impact
   - Immediate execution (30 seconds - 1 minute)

5. **Accumulation/Distribution**
   - Gradual buying/selling patterns
   - Long-term price impact
   - Extended execution time (20-60 minutes)

6. **Normal Flow**
   - Regular market activity
   - Minimal price impact
   - Standard execution time (3-10 minutes)

## Machine Learning Models

### Model Architecture

The system uses **XGBoost** (configurable) with three specialized models:

1. **Flow Type Classifier**
   - **Input**: Market features (volume, price, microstructure)
   - **Output**: Order flow pattern type (10 classes)
   - **Metric**: Accuracy

2. **Impact Predictor**
   - **Input**: Market features
   - **Output**: Expected price impact (percentage)
   - **Metric**: Mean Squared Error

3. **Execution Time Predictor**
   - **Input**: Market features
   - **Output**: Time to pattern completion (minutes)
   - **Metric**: Mean Squared Error

### Feature Engineering

#### Volume Features
- Buy/sell volume ratios (5m, 15m, 30m, 60m windows)
- Volume imbalances and momentum
- Large order counts and clustering

#### Price Features
- Price impact calculations
- Momentum indicators
- Volatility measures

#### Microstructure Features
- Bid-ask spread analysis
- Order book imbalance
- Market depth metrics
- Size imbalance ratios

#### Time Features
- Market hours, day-of-week patterns
- Time-based volatility
- Seasonal patterns

## Training Data Collection

### Pure Listener Mode Training

Since the strategy doesn't execute trades, training data is collected from:

1. **Market Event Predictions**
   - Features built from current market state
   - Actual impact calculated from price movements
   - Execution time estimated based on pattern type

2. **Depth Pattern Analysis**
   - Large order detection in order book
   - Order book imbalance calculations
   - Pattern classification from microstructure

### Training Process

```
Market Events → Feature Building → Pattern Detection → Sample Collection → Model Training
     ↓              ↓                    ↓                    ↓                    ↓
  Real-time      Technical +        Flow Type         Training Sample    XGBoost Models
  Data Stream    Microstructure     Classification    Addition           (1000 samples)
```

### Training Configuration

```yaml
# config/ml/order_flow_analyzer.yaml
model_type: "xgboost"
sequence_length: 20
prediction_horizon: 5
min_training_samples: 1000
retrain_interval: 1000
large_order_threshold: 1000
iceberg_detection_threshold: 0.8
cascade_detection_threshold: 0.7
```

## Configuration

### Strategy Configuration

```yaml
# config/strategies/enhanced_order_flow_strategy.yaml
symbols: ["BTC/USD", "ETH/USD"]
timeframes: ["1m", "5m"]

# Enhanced market data settings
use_microstructure_signals: true
spread_threshold: 0.002
imbalance_threshold: 2.0
volume_spike_threshold: 3.0
price_momentum_threshold: 0.01

# ML training settings
prediction_confidence_threshold: 0.6
max_order_events: 1000
max_depth_history: 100
```

### Broker Configuration

```yaml
# config/brokers/kraken.yaml
ws_url: "wss://ws.kraken.com/v2"
paper_trading: true
channels: ["trade", "book", "ohlc"]  # Enhanced channels
```

## Usage

### Starting the Strategy

```python
from src.strategy_framework.strategy_factory import StrategyFactory
from src.core.config import ConfigManager

# Load configuration
config_manager = ConfigManager("config/strategies/enhanced_order_flow_strategy.yaml")

# Create and start strategy
strategy = StrategyFactory.create_strategy("enhanced_order_flow", config_manager)
await strategy.start()
```

### Monitoring Training Progress

```python
# Get ML training status
ml_status = strategy.get_ml_training_status()
print(f"Training Progress: {ml_status['training_progress_percent']}")
print(f"Model Trained: {ml_status['model_trained']}")
print(f"Samples: {ml_status['training_samples']}/{ml_status['min_training_samples']}")

# Get detailed status
status = strategy.get_status()
print(f"Market Data: {status['market_data_status']}")
print(f"ML Status: {status['ml_training_status']}")
```

### Log Output Examples

```
ML Training Status for BTC/USD:
  Model trained: False
  Training samples: 150/1000
  Progress: 15.0%

Collected market-based training data:
  Predicted flow type: large_buy
  Actual impact: 0.0023
  Training samples: 151/1000 (15.1%)
  Time to execution: 2.0 minutes

STARTING ML MODEL TRAINING
============================================================
Training order flow analyzer with 1000 samples
Model type: xgboost
Sequence length: 20
Prediction horizon: 5 minutes

ML MODEL TRAINING COMPLETED
============================================================
Training completed successfully!
Models trained: ['flow_type', 'impact', 'time_to_execution']
Training metrics: {'flow_type_accuracy': 0.87, 'impact_mse': 0.0001, 'time_to_execution_mse': 2.3}
```

## Performance Monitoring

### Key Metrics

1. **Training Progress**
   - Samples collected vs. required
   - Model training status
   - Retraining frequency

2. **Market Data Quality**
   - Event counts by type
   - Data freshness
   - Connection stability

3. **Prediction Quality**
   - Confidence scores
   - Pattern detection accuracy
   - Impact prediction error

4. **System Health**
   - Memory usage
   - Processing latency
   - Error rates

### Health Checks

```python
# Check strategy health
is_healthy = await strategy.is_healthy()

# Get comprehensive status
status = strategy.get_status()
print(f"Strategy State: {status['state']}")
print(f"Event Counts: {status['order_events_count']}")
print(f"Predictions: {status['predictions_count']}")
```

## Troubleshooting

### Common Issues

1. **No Training Data Collection**
   - Check market data subscriptions
   - Verify event handlers are working
   - Ensure sufficient market activity

2. **Model Not Training**
   - Verify minimum sample requirement (1000)
   - Check feature engineering
   - Monitor training logs

3. **Poor Predictions**
   - Check model training metrics
   - Verify feature quality
   - Consider retraining with more data

4. **Market Data Issues**
   - Check WebSocket connections
   - Verify channel subscriptions
   - Monitor broker logs

### Debug Commands

```python
# Enable debug logging
import logging
logging.getLogger('strategy.EnhancedOrderFlowStrategy').setLevel(logging.DEBUG)
logging.getLogger('src.ml.order_flow_analyzer').setLevel(logging.DEBUG)

# Check data buffers
print(f"Order Events: {len(strategy.order_events)}")
print(f"Market Data: {len(strategy.market_data_buffer)}")
print(f"Tick Data: {len(strategy.tick_data.get('BTC/USD', []))}")
```

## Future Enhancements

### Planned Features

1. **Advanced Pattern Detection**
   - Machine learning-based pattern recognition
   - Adaptive threshold adjustment
   - Multi-timeframe analysis

2. **Enhanced Feature Engineering**
   - Deep learning features
   - Sentiment analysis integration
   - Cross-asset correlations

3. **Real-time Model Updates**
   - Online learning capabilities
   - Adaptive model architecture
   - Performance-based retraining

4. **Advanced Analytics**
   - Pattern backtesting
   - Performance attribution
   - Risk analysis integration

## Conclusion

The Order Flow Analysis System provides a comprehensive, ML-powered approach to market microstructure analysis. Operating in pure listener mode, it offers deep insights into market dynamics while continuously improving through real-time learning.

The system is designed for research, analysis, and strategy development, providing the foundation for advanced trading strategies while maintaining the safety of a non-executing environment. 