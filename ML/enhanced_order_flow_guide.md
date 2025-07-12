# Enhanced Order Flow Strategy Guide

## Overview

The Enhanced Order Flow Strategy is a sophisticated trading strategy that leverages real-time tick, quote, and depth data from the Kraken broker to identify and capitalize on order flow patterns. This strategy combines traditional order flow analysis with advanced market microstructure signals for improved trading performance.

## Key Features

### 🎯 Multi-Data Source Analysis
- **Tick Data**: Real-time price and volume updates
- **Quote Data**: Bid/ask spreads and size imbalances
- **Depth Data**: Order book structure and large order detection
- **OHLC Data**: Traditional candle patterns and volume analysis

### 🔍 Advanced Pattern Detection
- **Volume Spikes**: Detect unusual trading activity
- **Price Momentum**: Identify trending price movements
- **Spread Analysis**: Monitor bid/ask spread dynamics
- **Size Imbalances**: Detect bid/ask size asymmetries
- **Large Orders**: Identify institutional order flow
- **Order Book Imbalances**: Analyze depth structure

### 🤖 Machine Learning Integration
- **Order Flow Prediction**: ML-based pattern recognition
- **Confidence Scoring**: Risk-adjusted signal generation
- **Continuous Learning**: Model retraining with new data
- **Feature Engineering**: Advanced market microstructure features

## Configuration

### Basic Configuration

```yaml
strategy:
  name: "enhanced_order_flow"
  type: "order_flow"
  symbols: ["BTC/USD", "ETH/USD"]
  broker: "kraken"
  
  # Market data configuration
  market_data:
    max_tick_history: 1000
    max_quote_history: 500
    max_depth_history: 100
    max_order_events: 1000
    max_market_data: 100
```

### Microstructure Parameters

```yaml
microstructure:
  spread_threshold: 0.001      # 0.1% - wide spread detection
  imbalance_threshold: 2.0     # 2:1 ratio - size imbalance
  volume_spike_threshold: 3.0  # 3x average - volume spike
  price_momentum_threshold: 0.005  # 0.5% - momentum detection
```

### Trading Parameters

```yaml
trading:
  enable_trading: true
  follow_large_orders: true
  anticipate_icebergs: true
  avoid_cascades: true
  use_microstructure_signals: true
  
  # Risk management
  stop_loss_pct: 0.02
  take_profit_pct: 0.04
  max_position_size: 0.1
```

## Signal Types

### 1. Volume Spike Signals
**Trigger**: Volume exceeds threshold (default: 3x average)
**Action**: Follow the direction of the spike
**Confidence**: Based on volume ratio
**Use Case**: Institutional activity detection

```python
# Example volume spike detection
if current_volume > avg_volume * volume_spike_threshold:
    signal = Signal(
        action="buy" if price > bid else "sell",
        reason="volume_spike",
        confidence=min(0.9, volume_ratio / 5.0)
    )
```

### 2. Price Momentum Signals
**Trigger**: Price change exceeds threshold (default: 0.5%)
**Action**: Follow the momentum direction
**Confidence**: Based on momentum magnitude
**Use Case**: Trend following

```python
# Example momentum detection
if abs(price_change) > price_momentum_threshold:
    signal = Signal(
        action="buy" if price_change > 0 else "sell",
        reason="price_momentum",
        confidence=min(0.8, abs(price_change) / 0.01)
    )
```

### 3. Spread Analysis Signals
**Trigger**: Spread exceeds threshold (default: 0.1%)
**Action**: Avoid trading or use straddle strategies
**Confidence**: Based on spread magnitude
**Use Case**: Volatility detection

```python
# Example spread analysis
if spread_pct > spread_threshold:
    # Avoid trading during wide spreads
    logger.info("Wide spread detected - avoiding trades")
```

### 4. Size Imbalance Signals
**Trigger**: Bid/ask size ratio exceeds threshold (default: 2:1)
**Action**: Trade in the direction of larger size
**Confidence**: Based on imbalance ratio
**Use Case**: Order flow direction

```python
# Example size imbalance detection
if bid_size / ask_size > imbalance_threshold:
    signal = Signal(
        action="buy",
        price=ask_price,  # Pay the ask
        reason="size_imbalance_bid_dominance"
    )
```

### 5. Large Order Signals
**Trigger**: Large orders detected in order book
**Action**: Follow the direction of large orders
**Confidence**: Based on number of large orders
**Use Case**: Institutional order flow

```python
# Example large order detection
large_bids = [bid for bid in bids if size > large_order_threshold]
if len(large_bids) > len(large_asks):
    signal = Signal(
        action="buy",
        reason="large_buy_orders"
    )
```

### 6. Order Book Imbalance Signals
**Trigger**: Total bid/ask volume ratio exceeds threshold
**Action**: Trade in the direction of volume imbalance
**Confidence**: Based on imbalance ratio
**Use Case**: Market structure analysis

```python
# Example order book imbalance
imbalance = total_bid_size / total_ask_size
if imbalance > imbalance_threshold:
    signal = Signal(
        action="buy",
        reason="order_book_bid_imbalance"
    )
```

## Usage Examples

### Conservative Trading
```yaml
# Focus on high-confidence signals only
prediction_confidence_threshold: 0.8
use_microstructure_signals: false
max_position_size: 0.05
```

### Aggressive Trading
```yaml
# Use all available signals
prediction_confidence_threshold: 0.4
use_microstructure_signals: true
max_position_size: 0.2
volume_spike_threshold: 2.0
price_momentum_threshold: 0.003
```

### Market Making
```yaml
# Focus on microstructure signals
use_microstructure_signals: true
follow_large_orders: false
anticipate_icebergs: false
spread_threshold: 0.0005
imbalance_threshold: 1.5
```

## Running the Strategy

### 1. Set Environment Variables
```bash
export KRAKEN_API_KEY="your_api_key"
export KRAKEN_API_SECRET="your_api_secret"
```

### 2. Load Configuration
```python
import yaml

with open('config/strategies/enhanced_order_flow_strategy.yaml', 'r') as f:
    config = yaml.safe_load(f)
```

### 3. Initialize Strategy
```python
from strategy_framework.strategy_types.order_flow_strategy import OrderFlowStrategy

strategy = OrderFlowStrategy(
    strategy_id="my_enhanced_order_flow",
    config=config['strategy'],
    event_manager=event_manager,
    candle_manager=candle_manager,
    scheduler=scheduler,
    broker=broker
)

await strategy.initialize()
```

### 4. Monitor Performance
```python
# Get strategy status
status = strategy.get_status()

# Check market data
market_data = status['market_data_status']
print(f"Tick events: {market_data['tick_data_count']}")
print(f"Quote events: {market_data['quote_data_count']}")
print(f"Depth events: {market_data['depth_data_count']}")

# Check predictions
print(f"Predictions: {status['predictions_count']}")
if status['last_prediction']:
    pred = status['last_prediction']
    print(f"Last prediction: {pred['predicted_flow_type']} (confidence: {pred['confidence']:.3f})")
```

## Testing

### Run the Test Script
```bash
python tests/test_enhanced_order_flow_strategy.py
```

### Expected Output
```
✅ Enhanced Order Flow Strategy working correctly!
Total tick events: 150
Total quote events: 75
Total depth events: 25
Final order events: 200
Final predictions: 5
```

## Performance Monitoring

### Key Metrics
- **Order Flow Accuracy**: Prediction vs actual outcome
- **Signal Generation Rate**: Signals per hour
- **Microstructure Detection Rate**: Events per minute
- **Prediction Confidence**: Average confidence scores
- **Trading Performance**: P&L and Sharpe ratio

### Alerts
```yaml
alerts:
  low_confidence_predictions: 0.3
  high_prediction_error: 0.1
  low_signal_generation: 10  # signals per hour
  high_microstructure_events: 100  # events per minute
```

## Best Practices

### 1. Data Quality
- Monitor data feed health
- Validate event timestamps
- Check for data gaps

### 2. Risk Management
- Use position sizing based on confidence
- Implement stop losses
- Monitor drawdowns

### 3. Model Maintenance
- Retrain models regularly
- Monitor prediction accuracy
- Update feature engineering

### 4. Market Conditions
- Adjust thresholds for different market conditions
- Disable signals during high volatility
- Monitor correlation between signals

## Troubleshooting

### Common Issues

1. **No Market Data Received**
   - Check API credentials
   - Verify WebSocket connection
   - Check symbol format (e.g., "BTC/USD")

2. **Low Signal Generation**
   - Lower confidence thresholds
   - Adjust microstructure parameters
   - Check market activity

3. **High False Positives**
   - Increase confidence thresholds
   - Add additional filters
   - Review signal logic

4. **Performance Issues**
   - Reduce data history sizes
   - Optimize event handlers
   - Use async processing

### Debug Mode
```python
# Enable debug logging
logging.getLogger('strategy_framework.strategy_types.order_flow_strategy').setLevel(logging.DEBUG)

# Check individual components
print(f"Tick data: {len(strategy.tick_data['BTC/USD'])}")
print(f"Quote data: {len(strategy.quote_data['BTC/USD'])}")
print(f"Depth data: {len(strategy.depth_data['BTC/USD'])}")
```

## Advanced Features

### Custom Signal Generation
```python
# Add custom signal logic
async def custom_signal_generator(self, symbol: str, data: Dict) -> Optional[Signal]:
    # Your custom logic here
    if custom_condition:
        return Signal(
            symbol=symbol,
            action="buy",
            reason="custom_signal",
            confidence=0.7
        )
    return None
```

### Feature Engineering
```python
# Add custom features to ML model
def custom_features(self, tick_data: List, quote_data: List) -> Dict:
    return {
        'custom_metric_1': calculate_metric_1(tick_data),
        'custom_metric_2': calculate_metric_2(quote_data)
    }
```

## Conclusion

The Enhanced Order Flow Strategy provides a comprehensive framework for analyzing and trading based on market microstructure. By combining multiple data sources with advanced pattern recognition, it offers sophisticated trading capabilities while maintaining risk management and performance monitoring.

For more information, see:
- [Market Data Subscription Plan](market_data_subscription_plan.md)
- [Enhanced Market Data Guide](enhanced_market_data_guide.md)
- [Kraken Broker Documentation](../src/broker_interface/kraken_broker.py) 