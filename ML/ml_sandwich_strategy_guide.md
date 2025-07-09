# ML-Enhanced Sandwich Strategy Guide

## Overview

The ML-Enhanced Sandwich Strategy is an advanced version of the traditional sandwich strategy that uses machine learning to predict optimal order placement locations. This strategy combines the benefits of market making with intelligent order placement based on historical patterns and market conditions.

## Key Features

### 🤖 Machine Learning Integration
- **Order Placement Prediction**: Uses ML models to predict optimal buy/sell prices
- **Spread Multiplier Optimization**: Dynamically adjusts spread based on market conditions
- **Order Size Optimization**: Predicts optimal order sizes for different market conditions
- **Profitability Scoring**: Assigns confidence scores to predictions

### 📊 Advanced Features
- **Multiple ML Models**: Support for XGBoost, LightGBM, Random Forest, and LSTM
- **Feature Engineering**: Comprehensive feature set including technical indicators, regime detection, and time-based features
- **Continuous Learning**: Automatically retrains models with new data
- **Fallback Mechanism**: Falls back to rule-based strategy when ML confidence is low

### 🔄 Market Regime Awareness
- **Regime Detection**: Identifies trending, ranging, and volatile market conditions
- **Adaptive Behavior**: Adjusts strategy parameters based on current regime
- **Unknown Regime Handling**: Uses ML predictions even when regime is unclear

## Architecture

### Core Components

1. **OrderPlacementPredictor**: ML engine for predicting optimal order parameters
2. **FeatureBuilder**: Creates comprehensive feature sets from market data
3. **RegimeDetector**: Identifies market regimes for context-aware predictions
4. **Technical Indicators**: Provides input features for ML models

### Data Flow

```
Market Data → Feature Engineering → ML Prediction → Order Generation → Execution
     ↓              ↓                    ↓              ↓              ↓
  Candles    Technical Features    Buy/Sell Prices   Signals    Broker Orders
```

## Configuration

### Basic Configuration

```yaml
name: "ML Sandwich Strategy"
strategy_type: "ml_sandwich_strategy"
symbols: ["ETH/USD"]
timeframes: ["5m"]

# ML Configuration
use_ml_predictions: true
ml_confidence_threshold: 0.6
fallback_to_rule_based: true
collect_training_data: true
```

### ML Model Configuration

```yaml
ml_config:
  model_type: "xgboost"  # Options: xgboost, lightgbm, random_forest, lstm
  sequence_length: 20
  prediction_horizon: 5
  min_training_samples: 1000
  retrain_interval: 1000
  use_technical_indicators: true
  use_regime_features: true
  use_time_features: true
```

## Features Used by ML Models

### Technical Indicators
- **ATR (Average True Range)**: Volatility measurement
- **RSI (Relative Strength Index)**: Momentum oscillator
- **EMA (Exponential Moving Average)**: Trend following
- **ADX (Average Directional Index)**: Trend strength

### Price-Based Features
- **Price Changes**: 1m, 5m, 15m, 30m returns
- **Volatility**: Rolling standard deviation at multiple timeframes
- **Moving Averages**: SMA at 5, 15, 30 periods
- **Price vs MA Ratios**: Distance from moving averages

### Volume Features
- **Volume Moving Averages**: 5m and 15m
- **Volume Ratio**: Current volume vs average
- **Volume Trends**: Volume momentum

### Market Microstructure
- **High-Low Ratio**: Price range relative to close
- **Body Size**: Candle body relative to close
- **Recent Highs/Lows**: Distance from recent extremes

### Regime Features
- **One-Hot Encoded Regimes**: Trending up/down, ranging, volatile
- **Regime Duration**: Time in current regime
- **Regime Transitions**: Recent regime changes

### Time-Based Features
- **Hour of Day**: Market session awareness
- **Day of Week**: Weekly patterns
- **Seasonal Effects**: Time-based market behavior

## Usage

### 1. Setup Environment

```bash
# Install dependencies
pip install xgboost lightgbm scikit-learn tensorflow pandas numpy

# Set environment variables
export SAXO_API_KEY="your_api_key"
export SAXO_API_SECRET="your_api_secret"
```

### 2. Configure Strategy

The strategy is already configured in `config/strategies/ml_sandwich_strategy.yaml`. You can modify the settings as needed.

### 3. Run Strategy

The ML Sandwich Strategy is configured to run from the main framework. Simply start the main application:

```bash
python main.py
```

The strategy will be automatically loaded and started based on the configuration in `config/main.yaml`.

### 4. Monitor Performance

The strategy provides comprehensive logging including:
- ML prediction details
- Order placement decisions
- Performance metrics
- Feature importance rankings

## Model Training

### Initial Training

The strategy can be trained with historical data:

```python
# Train with 30 days of historical data
await train_model_with_historical_data(strategy, days_back=30)
```

### Continuous Learning

The strategy automatically:
1. **Collects Training Data**: Records actual order outcomes
2. **Retrains Models**: Updates models with new data every 1000 predictions
3. **Adapts to Market Changes**: Learns from changing market conditions

### Training Data Structure

```python
training_sample = {
    'features': {
        'current_price': 2000.0,
        'volatility_5m': 0.015,
        'rsi': 65.0,
        'regime_trending_up': 1.0,
        # ... more features
    },
    'targets': {
        'buy_price': 1995.0,
        'sell_price': 2005.0,
        'spread_multiplier': 1.8,
        'order_size': 0.15,
        'profitability': 0.75
    }
}
```

## Performance Monitoring

### Key Metrics

- **ML Prediction Accuracy**: How well predictions match actual outcomes
- **Confidence Scores**: Reliability of ML predictions
- **Feature Importance**: Which features drive predictions
- **Regime Performance**: Strategy performance by market regime

### Monitoring Dashboard

```python
status = strategy.get_status()
print(f"ML Model Trained: {status['ml_model_trained']}")
print(f"ML Predictions Used: {status['ml_predictions_used']}")
print(f"Total Signals: {status['total_signals']}")
```

## Advanced Configuration

### Model Selection

Choose the best model for your use case:

- **XGBoost**: Fast, good for most cases
- **LightGBM**: Memory efficient, good for large datasets
- **Random Forest**: Robust, handles outliers well
- **LSTM**: Best for time series patterns, requires more data

### Feature Engineering

Customize feature sets:

```yaml
ml_config:
  use_technical_indicators: true
  use_regime_features: true
  use_time_features: true
  use_volume_features: true
  use_microstructure_features: true
```

### Confidence Thresholds

Adjust ML usage:

```yaml
ml_confidence_threshold: 0.6  # Only use ML if confidence > 60%
fallback_to_rule_based: true  # Use rule-based when ML confidence is low
```

## Risk Management

### Built-in Safeguards

1. **Confidence Thresholds**: Only use high-confidence predictions
2. **Fallback Mechanisms**: Rule-based strategy when ML fails
3. **Position Limits**: Maximum position sizes
4. **Stop Losses**: Automatic loss protection
5. **Drawdown Limits**: Maximum acceptable losses

### Custom Risk Parameters

```yaml
max_position_size: 1000
stop_loss_pct: 0.05
take_profit_pct: 0.10
max_drawdown: 0.15
max_daily_loss: 0.05
risk_per_trade: 0.02
```

## Troubleshooting

### Common Issues

1. **Low ML Confidence**
   - Check feature quality
   - Increase training data
   - Adjust confidence threshold

2. **Poor Performance**
   - Review feature importance
   - Check market regime detection
   - Adjust model parameters

3. **Model Not Training**
   - Ensure sufficient historical data
   - Check feature availability
   - Verify data quality

### Debug Mode

Enable detailed logging:

```yaml
log_level: "DEBUG"
log_ml_predictions: true
log_training_data: true
```

## Best Practices

### 1. Data Quality
- Ensure clean, reliable market data
- Use appropriate timeframes for your strategy
- Validate technical indicators

### 2. Model Management
- Start with XGBoost for simplicity
- Monitor feature importance regularly
- Retrain models periodically

### 3. Risk Management
- Set appropriate position limits
- Use stop losses and take profits
- Monitor drawdown closely

### 4. Performance Optimization
- Use appropriate confidence thresholds
- Balance ML vs rule-based usage
- Monitor regime detection accuracy

## Future Enhancements

### Planned Features

1. **Ensemble Models**: Combine multiple ML models
2. **Online Learning**: Real-time model updates
3. **Cross-Asset Features**: Multi-asset correlations
4. **Sentiment Integration**: News and social media data
5. **Advanced Regime Detection**: More sophisticated regime identification

### Customization

The strategy is designed for easy extension:
- Add new features in `FeatureBuilder`
- Implement new ML models in `OrderPlacementPredictor`
- Customize regime detection logic
- Add new risk management rules

## Support

For questions and support:
1. Check the logs for detailed error messages
2. Review the configuration examples
3. Monitor the ML model performance metrics
4. Consult the troubleshooting section

The ML-Enhanced Sandwich Strategy represents a significant advancement in automated trading, combining the reliability of traditional market making with the intelligence of machine learning for optimal order placement. 