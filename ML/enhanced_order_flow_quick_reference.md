# Enhanced Order Flow Analysis - Quick Reference

## What's New

### ✅ Enhanced Market Data Subscriptions
- **Tick Data**: Individual trade executions from Kraken WebSocket
- **Quote Data**: Real-time bid/ask updates with spread analysis
- **Depth Data**: Full order book with large order detection
- **OHLC Data**: Traditional candlestick data

### ✅ Pure Listener Mode
- **No Trading**: Strategy only analyzes and trains ML models
- **Safe Operation**: Zero risk of accidental trades
- **Research Focus**: Perfect for strategy development and backtesting

### ✅ ML-Powered Analysis
- **XGBoost Models**: Three specialized models for pattern detection
- **Real-time Training**: Continuous learning from market data
- **Auto-retraining**: Models improve every 1,000 samples

## Configuration Files

### Strategy Config
```yaml
# config/strategies/enhanced_order_flow_strategy.yaml
symbols: ["BTC/USD", "ETH/USD"]
timeframes: ["1m", "5m"]

# Enhanced features
use_microstructure_signals: true
spread_threshold: 0.002
imbalance_threshold: 2.0
volume_spike_threshold: 3.0
price_momentum_threshold: 0.01
```

### ML Model Config
```yaml
# config/ml/order_flow_analyzer.yaml
model_type: "xgboost"
min_training_samples: 1000
retrain_interval: 1000
large_order_threshold: 1000
```

## Key Methods

### Strategy Status
```python
# Get ML training status
ml_status = strategy.get_ml_training_status()
print(f"Progress: {ml_status['training_progress_percent']}")
print(f"Model Trained: {ml_status['model_trained']}")

# Get comprehensive status
status = strategy.get_status()
print(f"Market Data: {status['market_data_status']}")
print(f"ML Status: {status['ml_training_status']}")
```

### Health Monitoring
```python
# Check if strategy is healthy
is_healthy = await strategy.is_healthy()

# Get event counts
print(f"Order Events: {len(strategy.order_events)}")
print(f"Tick Data: {len(strategy.tick_data.get('BTC/USD', []))}")
print(f"Depth Data: {len(strategy.depth_data.get('BTC/USD', []))}")
```

## Training Data Collection

### Automatic Collection
The strategy automatically collects training data from:

1. **Market Event Predictions**
   - Features from current market state
   - Actual impact from price movements
   - Estimated execution times

2. **Depth Pattern Analysis**
   - Large order detection
   - Order book imbalance
   - Pattern classification

### Training Timeline
- **0-1000 samples**: Data collection phase
- **1000 samples**: Initial model training
- **Every 1000 samples**: Model retraining
- **Continuous**: Feature importance updates

## Log Messages

### Training Progress
```
ML Training Status for BTC/USD:
  Model trained: False
  Training samples: 150/1000
  Progress: 15.0%
```

### Data Collection
```
Collected market-based training data:
  Predicted flow type: large_buy
  Actual impact: 0.0023
  Training samples: 151/1000 (15.1%)
  Time to execution: 2.0 minutes
```

### Model Training
```
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

## Order Flow Patterns

### Detected Patterns
1. **Large Buy/Sell**: Immediate impact, 1-3 min execution
2. **Iceberg Orders**: Medium impact, 10-20 min execution
3. **Cascades**: High volatility, 30 sec - 2 min execution
4. **Liquidation**: Extreme impact, 30 sec - 1 min execution
5. **Accumulation/Distribution**: Long-term, 20-60 min execution
6. **Normal Flow**: Minimal impact, 3-10 min execution

### Pattern Features
- **Volume Analysis**: Buy/sell ratios, imbalances, spikes
- **Price Impact**: Expected movement magnitude
- **Microstructure**: Spread, depth, order book imbalance
- **Timing**: Execution time predictions

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No training data | Insufficient market activity | Wait for more market events |
| Model not training | < 1000 samples | Monitor progress in logs |
| Poor predictions | Model needs retraining | Wait for auto-retrain |
| WebSocket errors | Connection issues | Check broker logs |

### Debug Commands
```python
# Enable debug logging
import logging
logging.getLogger('strategy.EnhancedOrderFlowStrategy').setLevel(logging.DEBUG)
logging.getLogger('src.ml.order_flow_analyzer').setLevel(logging.DEBUG)

# Check data quality
print(f"Market Data Buffer: {len(strategy.market_data_buffer)}")
print(f"Order Events: {len(strategy.order_events)}")
print(f"Training Samples: {len(strategy.order_flow_analyzer.training_data)}")
```

## Performance Expectations

### Initial Setup
- **Data Collection**: 10-15 minutes to reach 1000 samples
- **First Training**: 1-2 minutes for initial model
- **Prediction Quality**: Improves with more data

### Ongoing Operation
- **Retraining**: Every 1000 new samples
- **Prediction Accuracy**: 70-90% after sufficient training
- **Processing Latency**: < 100ms per prediction

### Resource Usage
- **Memory**: ~100-200MB for data buffers
- **CPU**: Low usage, spikes during training
- **Network**: WebSocket connections to Kraken

## Best Practices

### Configuration
- Start with default thresholds
- Adjust based on market conditions
- Monitor feature importance

### Monitoring
- Check training progress regularly
- Monitor prediction confidence
- Watch for data quality issues

### Maintenance
- Review logs for errors
- Check model performance metrics
- Consider retraining if accuracy drops

## Migration from Old System

### Key Changes
1. **No Trading Logic**: Removed all order placement
2. **Enhanced Data**: Added tick, quote, depth subscriptions
3. **ML Training**: Automatic training data collection
4. **Pure Listener**: Safe for research and development

### Configuration Updates
- Update broker config for new channels
- Adjust ML parameters as needed
- Set appropriate thresholds for your markets

### Testing
- Verify market data subscriptions
- Monitor training progress
- Check prediction quality
- Validate pattern detection 