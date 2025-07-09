# Order Flow Analyzer Guide

## Overview

The Order Flow Analyzer is a sophisticated machine learning system designed to detect and predict large order patterns in financial markets. It analyzes order flow data to identify various types of market microstructure patterns and provides predictions about where large orders will hit the market.

## Quick Start

### 1. Run the Strategy

The Order Flow Strategy is integrated into the main framework and can be run directly:

```bash
python main.py
```

The strategy is configured in `config/main.yaml` and will automatically:
- Collect order flow data from market events
- Analyze patterns using the ML model
- Generate trading signals based on predictions
- Collect training data from actual trades
- Retrain the model periodically

### 2. Model Storage

All trained models are automatically saved to `/data/models/` directory:
- Models persist across sessions
- Can be reused by other strategies
- Continuous learning from new data
- Automatic retraining when sufficient new data is collected

## Key Features

### 1. Order Flow Pattern Detection
- **Large Orders**: Identifies orders significantly larger than average market activity
- **Iceberg Orders**: Detects orders split into smaller pieces to avoid market impact
- **Stop Loss Cascades**: Recognizes cascading stop-loss orders that can cause rapid price movements
- **Liquidation Events**: Identifies forced selling/buying patterns
- **Accumulation/Distribution**: Detects gradual buying/selling patterns by large players

### 2. Machine Learning Models
- **Multiple Model Support**: XGBoost, LightGBM, Random Forest, and LSTM models
- **Feature Engineering**: Comprehensive feature extraction from order flow and market data
- **Continuous Learning**: Models retrain automatically with new data
- **Model Persistence**: Trained models are saved and can be reused across sessions

### 3. Prediction Capabilities
- **Flow Type Classification**: Predicts the type of order flow pattern
- **Impact Prediction**: Estimates the expected price impact of large orders
- **Timing Prediction**: Predicts when large orders will execute
- **Target Price Levels**: Identifies key price levels where orders are likely to hit

## Architecture

### Core Components

#### 1. OrderFlowAnalyzer
The main class that orchestrates the entire order flow analysis system.

```python
from src.ml.order_flow_analyzer import OrderFlowAnalyzer

config = {
    'model_type': 'xgboost',
    'large_order_threshold': 1000,
    'model_path': '/data/models/order_flow_model.pkl'
}

analyzer = OrderFlowAnalyzer(config)
```

#### 2. OrderFlowFeatureBuilder
Handles feature engineering from order flow and market data.

```python
from src.ml.order_flow_analyzer import OrderFlowFeatureBuilder

feature_builder = OrderFlowFeatureBuilder()
features = feature_builder.build_order_flow_features(order_events, market_data)
```

#### 3. OrderFlowEvent
Represents individual order flow events.

```python
from src.ml.order_flow_analyzer import OrderFlowEvent, OrderFlowType

event = OrderFlowEvent(
    timestamp=datetime.now(),
    symbol="ETH/USD",
    order_type="market",
    side="buy",
    quantity=1500,
    price=2000.0,
    order_id="order_123",
    is_large=True,
    flow_type=OrderFlowType.LARGE_BUY
)
```

### 4. Data Flow

```
Market Data → Order Flow Events → Feature Engineering → ML Model → Predictions → Trading Signals
     ↓              ↓                    ↓              ↓           ↓            ↓
  Candles      Order Events         Features      Predictions   Analysis    Strategy
```

## Configuration

### ML Configuration (`config/ml/order_flow_analyzer.yaml`)

```yaml
# Model Configuration
model_type: "xgboost"  # Options: xgboost, lightgbm, random_forest, lstm
sequence_length: 20
prediction_horizon: 5
min_training_samples: 1000
retrain_interval: 1000

# Order Flow Detection Parameters
large_order_threshold: 1000  # Minimum quantity to consider an order "large"
iceberg_detection_threshold: 0.8
cascade_detection_threshold: 0.7

# Model Storage
model_path: "/data/models/order_flow_model.pkl"

# Feature Engineering
use_volume_features: true
use_price_features: true
use_time_features: true
use_microstructure_features: true

# Analysis Windows (minutes)
analysis_windows: [5, 15, 30, 60]
```

### Strategy Configuration (`config/strategies/order_flow_strategy.yaml`)

```yaml
name: "Order Flow Strategy"
type: "src.strategy_framework.strategy_types.order_flow_strategy.OrderFlowStrategy"

# Order flow parameters
large_order_threshold: 1000
prediction_confidence_threshold: 0.6
max_order_events: 1000
max_market_data: 100

# Trading behavior
enable_trading: true
follow_large_orders: true
anticipate_icebergs: true
avoid_cascades: true
```

## Usage Examples

### 1. Basic Order Flow Analysis

```python
import asyncio
from src.ml.order_flow_analyzer import OrderFlowAnalyzer

async def analyze_order_flow():
    # Initialize analyzer
    config = {
        'model_type': 'xgboost',
        'large_order_threshold': 1000,
        'model_path': '/data/models/order_flow_model.pkl'
    }
    
    analyzer = OrderFlowAnalyzer(config)
    
    # Load pre-trained model
    await analyzer.load_model()
    
    # Analyze order flow
    prediction = await analyzer.predict_order_flow(order_events, market_data)
    
    print(f"Predicted Flow Type: {prediction.predicted_flow_type.value}")
    print(f"Confidence: {prediction.confidence:.3f}")
    print(f"Expected Impact: {prediction.expected_impact:.4f}")
    print(f"Target Levels: {prediction.target_price_levels}")

# Run analysis
asyncio.run(analyze_order_flow())
```

### 2. Training a New Model

```python
async def train_order_flow_model():
    # Initialize analyzer
    analyzer = OrderFlowAnalyzer(config)
    
    # Prepare training data
    training_data = [
        {
            'features': {
                'buy_volume_5m': 5000,
                'sell_volume_5m': 3000,
                'large_order_count_5m': 2,
                # ... more features
            },
            'targets': {
                'flow_type': 'large_buy',
                'impact': 0.02,
                'time_to_execution': 5.0
            }
        }
        # ... more training samples
    ]
    
    # Train model
    metrics = await analyzer.train(training_data)
    print(f"Training metrics: {metrics}")
    
    # Save model
    await analyzer.save_model()

# Train model
asyncio.run(train_order_flow_model())
```

### 3. Large Order Detection

```python
def detect_large_orders():
    analyzer = OrderFlowAnalyzer(config)
    
    # Detect large orders in order flow
    large_orders = analyzer.detect_large_orders(order_events)
    
    for order in large_orders:
        print(f"Large order detected: {order.order_id}")
        print(f"  Quantity: {order.quantity}")
        print(f"  Flow type: {order.flow_type.value}")
        print(f"  Confidence: {order.confidence:.3f}")
```

## Feature Engineering

The Order Flow Analyzer extracts comprehensive features from order flow and market data:

### Volume Features
- Buy/sell volume in different time windows
- Volume imbalance ratios
- Large order counts
- Order size distribution statistics

### Price Features
- Price impact of orders
- Volatility measures
- Price momentum
- High-low spreads

### Time Features
- Hour of day
- Day of week
- Market open/close status
- Order clustering metrics

### Microstructure Features
- Bid-ask spread estimates
- Market depth
- Price efficiency
- Volume-weighted average price (VWAP)

## Order Flow Types

### 1. Large Buy/Sell Orders
- **Description**: Single large orders that can move the market
- **Detection**: Orders above the large order threshold
- **Trading Strategy**: Follow the direction of large orders

### 2. Iceberg Orders
- **Description**: Large orders split into smaller pieces
- **Detection**: Pattern of similar-sized orders in short time windows
- **Trading Strategy**: Anticipate the direction and place orders ahead

### 3. Stop Loss Cascades
- **Description**: Cascading stop-loss orders triggered by price movements
- **Detection**: Rapid sequence of stop orders in the same direction
- **Trading Strategy**: Avoid trading during cascades, close existing positions

### 4. Liquidation Events
- **Description**: Forced selling/buying due to margin calls or other constraints
- **Detection**: Unusual volume spikes with price pressure
- **Trading Strategy**: Avoid trading, protect existing positions

### 5. Accumulation/Distribution
- **Description**: Gradual buying/selling by large players
- **Detection**: Consistent directional flow over extended periods
- **Trading Strategy**: Follow the accumulation/distribution pattern

## Integration with Trading Strategies

The Order Flow Analyzer is integrated into the `OrderFlowStrategy` class:

```python
from src.strategy_framework.strategy_types.order_flow_strategy import OrderFlowStrategy

# The strategy automatically:
# 1. Collects order flow data from market events
# 2. Analyzes patterns using the ML model
# 3. Generates trading signals based on predictions
# 4. Collects training data from actual trades
# 5. Retrains the model periodically
```

### Strategy Integration
- Automatically loaded via `config/main.yaml`
- Runs alongside other strategies
- Shares the same broker and event system
- Collects training data from actual trades

### Model Reuse
- Trained models saved to `/data/models/`
- Can be used by other strategies
- Continuous learning across sessions
- Persistent knowledge accumulation

## Model Performance Monitoring

### Key Metrics
- **Accuracy**: Classification accuracy for flow type prediction
- **Precision/Recall**: Performance on specific flow types
- **MSE**: Mean squared error for impact and timing predictions
- **Feature Importance**: Which features contribute most to predictions

### Continuous Learning
- Models automatically retrain with new data
- Performance metrics are tracked over time
- Feature importance is updated as patterns evolve

## Best Practices

### 1. Data Quality
- Ensure order flow data is accurate and timely
- Filter out noise and invalid orders
- Use appropriate time windows for analysis

### 2. Model Selection
- Start with XGBoost for good performance and interpretability
- Use LSTM for complex temporal patterns
- Experiment with different model types for your specific use case

### 3. Feature Engineering
- Include both order flow and market data features
- Use multiple time windows for comprehensive analysis
- Regularly review and update feature importance

### 4. Risk Management
- Set appropriate confidence thresholds for trading decisions
- Avoid trading during detected cascade events
- Monitor model performance and retrain as needed

### 5. Model Persistence
- Models are automatically saved to `/data/models/`
- Can be reused across sessions and strategies
- Continuous learning from new market data
- No need to retrain from scratch

### 6. Backtesting
- Test strategies on historical order flow data
- Validate predictions against actual market outcomes
- Use walk-forward analysis to avoid overfitting

## Troubleshooting

### Common Issues

1. **Low Prediction Confidence**
   - Check data quality and feature engineering
   - Ensure sufficient training data
   - Review model parameters

2. **Model Not Training**
   - Verify training data format
   - Check minimum training sample requirements
   - Ensure all required features are present

3. **Poor Prediction Accuracy**
   - Collect more training data
   - Review feature importance
   - Try different model types
   - Check for data leakage

4. **Model Loading Issues**
   - Verify model file path
   - Check model file integrity
   - Ensure compatible model versions

## Future Enhancements

### Planned Features
- **Real-time Order Book Analysis**: Direct integration with order book data
- **Multi-Asset Support**: Analysis across multiple instruments
- **Advanced Pattern Recognition**: Deep learning for complex patterns
- **Sentiment Integration**: Combine with news and social media sentiment
- **Portfolio Optimization**: Multi-strategy order flow integration

### Research Areas
- **Market Microstructure**: Advanced order book modeling
- **High-Frequency Patterns**: Ultra-short-term order flow analysis
- **Cross-Asset Correlation**: Order flow relationships between assets
- **Regime Detection**: Market regime-specific order flow patterns

## Conclusion

The Order Flow Analyzer provides a powerful foundation for understanding and predicting market microstructure patterns. By combining sophisticated feature engineering with machine learning models, it can identify opportunities and risks in order flow data that would be impossible to detect manually.

The system is designed to be:
- **Scalable**: Handles large volumes of order flow data
- **Adaptive**: Continuously learns from new market conditions
- **Interpretable**: Provides clear explanations for predictions
- **Persistent**: Models are saved to `/data/models/` for reuse
- **Integrable**: Works seamlessly with existing trading strategies

For questions or support, please refer to the test scripts and configuration examples provided in the codebase. 