# Machine Learning Manual

## Table of Contents
1. [Introduction](#introduction)
2. [Model Types](#model-types)
3. [Feature Engineering](#feature-engineering)
4. [Model Training](#model-training)
5. [Reinforcement Learning](#reinforcement-learning)
6. [Model Evaluation](#model-evaluation)
7. [Integration with Trading](#integration-with-trading)
8. [Advanced Features](#advanced-features)
9. [Best Practices](#best-practices)
10. [Examples](#examples)

## Introduction

The machine learning system provides powerful tools for building and training models to assist in trading decisions. It includes both traditional supervised learning models and reinforcement learning capabilities for automated trading strategies.

## Model Types

The framework supports several types of models:

```python
from src.ml.model_builder import ModelType, ModelConfig

# Available model types
model_types = [
    ModelType.RANDOM_FOREST,      # Random Forest Classifier
    ModelType.GRADIENT_BOOSTING,  # Gradient Boosting Classifier
    ModelType.XGBOOST,           # XGBoost Classifier
    ModelType.LIGHTGBM,          # LightGBM Classifier
    ModelType.LSTM               # Long Short-Term Memory Network
]

# Configure model
config = ModelConfig(
    model_type=ModelType.XGBOOST,
    target_column='price_direction',
    prediction_horizon=5,        # Predict 5 periods ahead
    sequence_length=10,          # Look back 10 periods
    test_size=0.2,
    validation_size=0.1,
    n_estimators=100,
    learning_rate=0.01,
    max_depth=5
)
```

## Feature Engineering

### Technical Features

Build technical indicators and features:

```python
from src.ml.model_builder import FeatureBuilder

# Initialize feature builder
feature_builder = FeatureBuilder()

# Build technical features
technical_features = feature_builder.build_technical_features(
    df=price_data,
    indicators=['rsi', 'macd', 'bollinger_bands']
)

# Available features include:
# - Returns and log returns
# - Volatility
# - Volume metrics
# - Technical indicators
# - Lagged features
```

### News Features

Process news data and sentiment:

```python
# Build news features
news_features = feature_builder.build_news_features(
    news_items=news_data,
    time_index=price_data.index
)

# Available features include:
# - News count
# - Average sentiment
# - Sentiment volatility
# - Positive/negative news ratios
```

### Feature Combination

Combine technical and news features:

```python
# Combine features
combined_features = feature_builder.combine_features(
    technical_features,
    news_features
)

# Features are automatically:
# - Aligned by timestamp
# - Scaled appropriately
# - Combined into a single DataFrame
```

## Model Training

### Supervised Learning

Train traditional ML models:

```python
from src.ml.model_builder import ModelBuilder

# Initialize model builder
model_builder = ModelBuilder(config)

# Prepare data
X, y, feature_names = model_builder.prepare_data(
    technical_data=price_data,
    news_data=news_data,
    target_data=target_series
)

# Build and train model
model_builder.build_model()
metrics = model_builder.train(X, y)

# Access training metrics
print(f"Accuracy: {metrics['accuracy']}")
print(f"Precision: {metrics['precision']}")
print(f"Recall: {metrics['recall']}")
print(f"F1 Score: {metrics['f1']}")
```

### Model Persistence

Save and load models:

```python
# Save model
model_builder.save_model('models/price_prediction.pkl')

# Load model
model_builder.load_model('models/price_prediction.pkl')

# Get feature importance
importance = model_builder.get_feature_importance()
```

## Reinforcement Learning

### Trading Environment

Create a custom trading environment:

```python
from src.ml.rl_trader import TradingEnvironment, RLConfig

# Initialize environment
env = TradingEnvironment(
    data=price_data,
    initial_balance=100000.0,
    transaction_fee=0.001
)

# Environment features:
# - Price data
# - Technical indicators
# - Account balance
# - Current position
# - Unrealized P&L
```

### DQN Agent

Train a Deep Q-Learning agent:

```python
from src.ml.rl_trader import DQNAgent, RLTradingStrategy

# Configure RL agent
config = RLConfig(
    state_size=env.observation_space.shape[0],
    action_size=env.action_space.n,
    hidden_size=128,
    learning_rate=0.001,
    gamma=0.99,
    epsilon=1.0,
    epsilon_min=0.01,
    epsilon_decay=0.995
)

# Initialize trading strategy
strategy = RLTradingStrategy(
    data=price_data,
    config=config,
    initial_balance=100000.0
)

# Train strategy
history = strategy.train(episodes=1000)

# Access training history
rewards = history['rewards']
losses = history['losses']
```

## Model Evaluation

### Performance Metrics

Evaluate model performance:

```python
# Supervised learning metrics
metrics = {
    'accuracy': accuracy_score(y_true, y_pred),
    'precision': precision_score(y_true, y_pred),
    'recall': recall_score(y_true, y_pred),
    'f1': f1_score(y_true, y_pred)
}

# RL metrics
rl_metrics = {
    'total_reward': sum(rewards),
    'average_reward': np.mean(rewards),
    'max_reward': max(rewards),
    'final_balance': strategy.balance
}
```

### Cross-Validation

Perform time series cross-validation:

```python
from sklearn.model_selection import TimeSeriesSplit

# Initialize time series split
tscv = TimeSeriesSplit(n_splits=5)

# Perform cross-validation
for train_idx, test_idx in tscv.split(X):
    X_train, X_test = X[train_idx], X[test_idx]
    y_train, y_test = y[train_idx], y[test_idx]
    
    # Train and evaluate
    model_builder.build_model()
    metrics = model_builder.train(X_train, y_train)
```

## Integration with Trading

### Making Predictions

Use models for trading decisions:

```python
# Supervised learning prediction
prediction = model_builder.predict(new_data)

# RL strategy prediction
action = strategy.predict(current_state)

# Available actions:
# 0: Hold
# 1: Buy
# 2: Sell
```

### Real-time Updates

Update models with new data:

```python
# Update feature builder
feature_builder.update_scalers(new_data)

# Update model predictions
new_features = feature_builder.build_technical_features(new_data)
prediction = model_builder.predict(new_features)
```

## Advanced Features

### Market Microstructure Features

Build features from order book data:

```python
from src.ml.advanced_features import AdvancedFeatureBuilder

# Initialize advanced feature builder
advanced_builder = AdvancedFeatureBuilder()

# Build market microstructure features
microstructure_features = advanced_builder.build_market_microstructure_features(
    order_book_data=order_book_data,
    window_sizes=[5, 10, 20]
)

# Available features include:
# - Order book imbalance
# - Relative spread
# - Market depth
# - Price impact
```

### Cross-Asset Features

Build features from multiple assets:

```python
# Build cross-asset features
cross_asset_features = advanced_builder.build_cross_asset_features(
    price_data_dict={
        'SPY': spy_data,
        'QQQ': qqq_data,
        'IWM': iwm_data
    },
    window_sizes=[5, 10, 20, 50]
)

# Available features include:
# - Cross-asset correlations
# - Cointegration metrics
# - Cross-asset momentum
```

### Market Regime Features

Build features for market regime detection:

```python
# Build market regime features
regime_features = advanced_builder.build_market_regime_features(
    price_data=price_data,
    window_sizes=[20, 50, 100]
)

# Available features include:
# - Volatility regimes
# - Trend strength (ADX)
# - Mean reversion metrics
```

### Risk-Aware Reinforcement Learning

Implement risk-aware trading strategies:

```python
from src.ml.risk_aware_rl import RiskAwareRLAgent, RiskConstraints

# Define risk constraints
risk_constraints = RiskConstraints(
    max_var=0.02,          # Maximum Value at Risk (2%)
    max_drawdown=0.1,      # Maximum drawdown (10%)
    max_leverage=2.0,      # Maximum leverage
    min_sharpe=0.5,        # Minimum Sharpe ratio
    max_correlation=0.7,   # Maximum correlation with market
    max_position_size=0.2  # Maximum position size (20% of portfolio)
)

# Initialize risk-aware agent
risk_agent = RiskAwareRLAgent(
    state_size=state_size,
    action_size=action_size,
    risk_constraints=risk_constraints
)

# Train with risk awareness
history = risk_agent.train(episodes=1000)

# Get risk report
risk_report = risk_agent.get_risk_report()
```

### Model Monitoring and Interpretability

Monitor model performance and interpret predictions:

```python
from src.ml.model_monitoring import ModelMonitor, MonitoringConfig

# Configure monitoring
monitoring_config = MonitoringConfig(
    performance_threshold=0.1,    # Threshold for performance drift
    feature_drift_threshold=0.1,  # Threshold for feature drift
    retraining_threshold=0.2,     # Threshold for triggering retraining
    min_samples=100,             # Minimum samples for drift detection
    window_size=1000             # Window size for rolling calculations
)

# Initialize model monitor
monitor = ModelMonitor(
    model=model,
    config=monitoring_config,
    feature_names=feature_names
)

# Check for performance drift
drift_detected, drift_metrics = monitor.check_performance_drift(
    new_data=new_data,
    new_labels=new_labels
)

# Check for feature drift
feature_drift_detected, feature_drift_metrics = monitor.check_feature_drift(
    new_data=new_data
)

# Get feature importance
importance = monitor.get_feature_importance(
    data=new_data,
    method='shap'  # or 'lime'
)

# Get prediction confidence
confidence = monitor.get_prediction_confidence(new_data)

# Get comprehensive model report
report = monitor.get_model_report()

# Check if retraining is needed
should_retrain = monitor.should_retrain()
```

## Best Practices

1. **Data Preparation**
   - Clean and normalize data
   - Handle missing values appropriately
   - Use appropriate scaling methods
   - Consider time series characteristics

2. **Feature Engineering**
   - Start with basic technical indicators
   - Add domain-specific features
   - Consider feature interactions
   - Monitor feature importance

3. **Model Selection**
   - Choose model based on data characteristics
   - Consider computational requirements
   - Balance complexity and performance
   - Use appropriate validation methods

4. **Training Process**
   - Use appropriate train/test splits
   - Implement early stopping
   - Monitor for overfitting
   - Regularize when necessary

5. **Evaluation**
   - Use multiple metrics
   - Consider transaction costs
   - Account for market impact
   - Monitor model drift

6. **Risk Management**
   - Set appropriate risk constraints
   - Monitor portfolio metrics
   - Implement position sizing
   - Track correlation exposure

7. **Model Monitoring**
   - Track performance drift
   - Monitor feature drift
   - Check prediction confidence
   - Implement automated retraining

## Examples

### Complete ML Pipeline

```python
from src.ml.model_builder import ModelBuilder, ModelConfig, ModelType
from src.ml.risk_aware_rl import RLTradingStrategy, RLConfig, RiskConstraints
from src.ml.model_monitoring import ModelMonitor, MonitoringConfig

# 1. Supervised Learning Pipeline
def supervised_learning_pipeline():
    # Configure model
    config = ModelConfig(
        model_type=ModelType.XGBOOST,
        target_column='price_direction',
        prediction_horizon=5
    )
    
    # Initialize model builder
    model_builder = ModelBuilder(config)
    
    # Prepare data
    X, y, features = model_builder.prepare_data(
        technical_data=price_data,
        news_data=news_data,
        target_data=target_series
    )
    
    # Train model
    model_builder.build_model()
    metrics = model_builder.train(X, y)
    
    # Save model
    model_builder.save_model('models/price_prediction.pkl')
    
    return model_builder, metrics

# 2. Reinforcement Learning Pipeline
def reinforcement_learning_pipeline():
    # Configure RL agent
    config = RLConfig(
        state_size=state_size,
        action_size=action_size,
        hidden_size=128
    )
    
    # Define risk constraints
    risk_constraints = RiskConstraints(
        max_var=0.02,
        max_drawdown=0.1,
        max_leverage=2.0
    )
    
    # Initialize trading strategy
    strategy = RLTradingStrategy(
        data=price_data,
        config=config,
        risk_constraints=risk_constraints,
        initial_balance=100000.0
    )
    
    # Train strategy
    history = strategy.train(episodes=1000)
    
    # Save strategy
    strategy.save_strategy('models/rl_strategy.pkl')
    
    return strategy, history

# 3. Model Monitoring Pipeline
def monitoring_pipeline(model, feature_names):
    # Configure monitoring
    monitoring_config = MonitoringConfig(
        performance_threshold=0.1,
        feature_drift_threshold=0.1,
        retraining_threshold=0.2
    )
    
    # Initialize monitor
    monitor = ModelMonitor(
        model=model,
        config=monitoring_config,
        feature_names=feature_names
    )
    
    # Monitor model
    drift_detected, drift_metrics = monitor.check_performance_drift(
        new_data=new_data,
        new_labels=new_labels
    )
    
    # Get model report
    report = monitor.get_model_report()
    
    # Check if retraining is needed
    should_retrain = monitor.should_retrain()
    
    return monitor, report

# 4. Combined Approach
def combined_trading_system():
    # Get supervised learning predictions
    sl_model, _ = supervised_learning_pipeline()
    sl_prediction = sl_model.predict(new_data)
    
    # Get RL strategy actions
    rl_strategy, _ = reinforcement_learning_pipeline()
    rl_action = rl_strategy.predict(current_state)
    
    # Monitor models
    sl_monitor, sl_report = monitoring_pipeline(
        sl_model,
        feature_names
    )
    
    rl_monitor, rl_report = monitoring_pipeline(
        rl_strategy.model,
        feature_names
    )
    
    # Combine predictions with monitoring
    if (sl_prediction > 0.7 and 
        rl_action == 1 and 
        sl_monitor.get_prediction_confidence(new_data).mean() > 0.8):
        return 'BUY'
    elif (sl_prediction < 0.3 and 
          rl_action == 2 and 
          sl_monitor.get_prediction_confidence(new_data).mean() > 0.8):
        return 'SELL'
    else:
        return 'HOLD'
```

### Feature Engineering Example

```python
def build_advanced_features():
    # Initialize feature builder
    advanced_builder = AdvancedFeatureBuilder()
    
    # Build technical features
    technical_features = advanced_builder.build_technical_features(
        df=price_data,
        indicators=[
            'rsi', 'macd', 'bollinger_bands',
            'stochastic', 'atr', 'adx'
        ]
    )
    
    # Build market microstructure features
    microstructure_features = advanced_builder.build_market_microstructure_features(
        order_book_data=order_book_data,
        window_sizes=[5, 10, 20]
    )
    
    # Build cross-asset features
    cross_asset_features = advanced_builder.build_cross_asset_features(
        price_data_dict={
            'SPY': spy_data,
            'QQQ': qqq_data,
            'IWM': iwm_data
        },
        window_sizes=[5, 10, 20, 50]
    )
    
    # Build market regime features
    regime_features = advanced_builder.build_market_regime_features(
        price_data=price_data,
        window_sizes=[20, 50, 100]
    )
    
    # Build sentiment features
    sentiment_features = advanced_builder.build_sentiment_features(
        news_data=news_data,
        social_data=social_data,
        window_sizes=[1, 3, 5, 10]
    )
    
    # Combine all features
    combined_features = pd.concat([
        technical_features,
        microstructure_features,
        cross_asset_features,
        regime_features,
        sentiment_features
    ], axis=1)
    
    # Scale features
    scaled_features = advanced_builder.scale_features(combined_features)
    
    return scaled_features
```

## Model Training and Live Trading Deployment

### Training and Validation Process

```python
from src.ml.model_builder import ModelBuilder, ModelConfig
from src.ml.model_monitoring import ModelMonitor, MonitoringConfig

def train_and_validate_model():
    # 1. Split data into training, validation, and test sets
    train_data = data.loc[:'2022-12-31']
    val_data = data.loc['2023-01-01':'2023-06-30']
    test_data = data.loc['2023-07-01':]
    
    # 2. Configure model
    config = ModelConfig(
        model_type=ModelType.XGBOOST,
        target_column='price_direction',
        prediction_horizon=5,
        sequence_length=10,
        test_size=0.2,
        validation_size=0.1
    )
    
    # 3. Train model
    model_builder = ModelBuilder(config)
    model_builder.build_model()
    train_metrics = model_builder.train(
        X_train=train_data[features],
        y_train=train_data[target]
    )
    
    # 4. Validate on out-of-sample data
    val_predictions = model_builder.predict(val_data[features])
    val_metrics = calculate_metrics(val_data[target], val_predictions)
    
    # 5. Final test on unseen data
    test_predictions = model_builder.predict(test_data[features])
    test_metrics = calculate_metrics(test_data[target], test_predictions)
    
    return model_builder, train_metrics, val_metrics, test_metrics
```

### Live Trading Deployment

```python
class LiveTradingModel:
    def __init__(
        self,
        model,
        feature_builder,
        monitoring_config: MonitoringConfig
    ):
        self.model = model
        self.feature_builder = feature_builder
        self.monitor = ModelMonitor(
            model=model,
            config=monitoring_config,
            feature_names=feature_builder.feature_names
        )
        self.prediction_history = []
        self.performance_history = []
    
    def prepare_live_features(self, market_data: Dict) -> pd.DataFrame:
        """Prepare features for live prediction."""
        # Build technical features
        technical_features = self.feature_builder.build_technical_features(
            df=market_data['price_data']
        )
        
        # Build market microstructure features
        microstructure_features = self.feature_builder.build_market_microstructure_features(
            order_book_data=market_data['order_book']
        )
        
        # Combine features
        features = pd.concat([
            technical_features,
            microstructure_features
        ], axis=1)
        
        return features
    
    def make_prediction(self, market_data: Dict) -> Dict:
        """Make prediction for live trading."""
        # Prepare features
        features = self.prepare_live_features(market_data)
        
        # Get prediction
        prediction = self.model.predict(features)
        
        # Get prediction confidence
        confidence = self.monitor.get_prediction_confidence(features)
        
        # Check for model drift
        drift_detected, drift_metrics = self.monitor.check_feature_drift(features)
        
        # Store prediction
        self.prediction_history.append({
            'timestamp': market_data['timestamp'],
            'prediction': prediction,
            'confidence': confidence.mean(),
            'drift_detected': drift_detected
        })
        
        return {
            'prediction': prediction,
            'confidence': confidence.mean(),
            'drift_detected': drift_detected,
            'drift_metrics': drift_metrics
        }
    
    def update_performance(self, trade_result: Dict):
        """Update model performance with trade results."""
        self.performance_history.append(trade_result)
        
        # Check if retraining is needed
        if self.monitor.should_retrain():
            self.retrain_model()
    
    def retrain_model(self):
        """Retrain model with new data."""
        # Get recent data
        recent_data = self.get_recent_data()
        
        # Retrain model
        self.model.fit(
            recent_data['features'],
            recent_data['target']
        )
        
        # Update feature builder
        self.feature_builder.update_scalers(recent_data['features'])
        
        # Reset monitor
        self.monitor = ModelMonitor(
            model=self.model,
            config=self.monitor.config,
            feature_names=self.feature_builder.feature_names
        )

# Example usage in live trading
def live_trading_example():
    # 1. Train and validate model
    model_builder, train_metrics, val_metrics, test_metrics = train_and_validate_model()
    
    # 2. Initialize live trading model
    live_model = LiveTradingModel(
        model=model_builder.model,
        feature_builder=model_builder.feature_builder,
        monitoring_config=MonitoringConfig(
            performance_threshold=0.1,
            feature_drift_threshold=0.1,
            retraining_threshold=0.2
        )
    )
    
    # 3. Live trading loop
    while True:
        # Get market data
        market_data = get_live_market_data()
        
        # Make prediction
        prediction = live_model.make_prediction(market_data)
        
        # Execute trade if confidence is high enough
        if prediction['confidence'] > 0.8 and not prediction['drift_detected']:
            trade_result = execute_trade(prediction['prediction'])
            live_model.update_performance(trade_result)
        
        # Wait for next update
        time.sleep(60)  # 1-minute intervals
```

### Best Practices for Live Trading

1. **Data Quality**
   - Use high-quality, clean data for training
   - Implement robust data validation
   - Handle missing data appropriately
   - Consider data latency in live trading

2. **Model Validation**
   - Use proper train/validation/test splits
   - Validate on out-of-sample data
   - Test on different market regimes
   - Consider transaction costs in validation

3. **Risk Management**
   - Start with small position sizes
   - Implement stop-loss mechanisms
   - Monitor model confidence
   - Track prediction accuracy

4. **Monitoring and Maintenance**
   - Monitor model drift
   - Track feature importance
   - Implement automated retraining
   - Keep detailed performance logs

5. **Deployment Strategy**
   - Start with paper trading
   - Gradually increase position sizes
   - Monitor system performance
   - Have fallback mechanisms

### Example Deployment Workflow

```python
def deployment_workflow():
    # 1. Initial Training
    model, metrics = train_and_validate_model()
    
    # 2. Paper Trading Phase
    paper_trading_results = paper_trading_phase(
        model=model,
        duration_days=30,
        initial_balance=100000.0
    )
    
    # 3. Small Live Trading
    if paper_trading_results['sharpe_ratio'] > 1.0:
        small_live_results = live_trading_phase(
            model=model,
            duration_days=30,
            initial_balance=10000.0,
            max_position_size=0.1
        )
    
    # 4. Full Deployment
    if small_live_results['sharpe_ratio'] > 1.5:
        full_deployment = live_trading_phase(
            model=model,
            duration_days=None,  # Indefinite
            initial_balance=100000.0,
            max_position_size=0.2
        )
    
    return {
        'paper_trading': paper_trading_results,
        'small_live': small_live_results,
        'full_deployment': full_deployment
    }
```

---

This manual provides a comprehensive guide to the machine learning capabilities of the framework. For specific trading strategies and portfolio management, refer to the respective manuals (options_manual.md and portfolio_manual.md). 