# Advanced ML Features Guide

## Table of Contents
1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Core Features](#core-features)
4. [Order Book Imbalance Momentum](#order-book-imbalance-momentum)
5. [Microstructure Regime Classification](#microstructure-regime-classification)
6. [Liquidity Provision Patterns](#liquidity-provision-patterns)
7. [Microstructure Prediction](#microstructure-prediction)
8. [Liquidity Shock Detection](#liquidity-shock-detection)
9. [Advanced ML Feature Engine](#advanced-ml-feature-engine)
8. [Integration with Strategies](#integration-with-strategies)
9. [Configuration](#configuration)
10. [Usage Examples](#usage-examples)
11. [Best Practices](#best-practices)

## Overview

The Advanced ML Features library provides three cutting-edge machine learning modules designed to extract high-alpha signals from market microstructure data. These features work on top of existing trained models and provide sophisticated analysis capabilities that can significantly enhance trading performance.

### Key Components

1. **Order Book Imbalance Momentum**: Weighted imbalance analysis across multiple depth levels with exponential decay
2. **Microstructure Regime Classification**: Identify accumulation/distribution/neutral regimes using HMM and gradient boosting
3. **Liquidity Provision Patterns**: Detect when market makers are "stepping away" before significant moves
4. **Microstructure Prediction**: LSTM/Transformer models for predicting order book shape changes 1-10 seconds ahead
5. **Liquidity Shock Detection**: Anomaly detection for unusual order book states and whale movements
6. **Advanced ML Feature Engine**: Orchestrates all modules and generates interaction features

### Why These Features Matter

These advanced features address critical gaps in traditional technical analysis:
- **Traditional indicators lag** - These features use forward-looking order book data
- **Price-only analysis misses liquidity** - These features analyze market depth and structure
- **Static models miss regime changes** - These features detect market microstructure shifts
- **Individual signals lack context** - These features combine multiple signals for higher confidence

## Quick Start

### 1. Enable Advanced Features

```python
from src.ml.order_flow_analyzer import OrderFlowAnalyzer

# Initialize with advanced features enabled
config = {
    'model_type': 'xgboost',
    'large_order_threshold': 1000,
    'enable_advanced_features': True,
    'advanced_features_config_path': 'config/ml/advanced_features_config.yaml'
}

analyzer = OrderFlowAnalyzer(config)
```

### 2. Get Advanced Analysis

```python
# Get comprehensive advanced analysis
advanced_analysis = analyzer.get_advanced_analysis(
    order_book_data=order_book,
    trade_data=recent_trades,
    market_data=market_data
)

print(f"Market Regime: {advanced_analysis['regime']['current_regime']}")
print(f"Imbalance Score: {advanced_analysis['imbalance']['weighted_imbalance']:.4f}")
print(f"Liquidity Stress: {advanced_analysis['liquidity']['liquidity_stress']:.2f}")
```

### 3. Get High-Alpha Opportunities

```python
# Get actionable trading opportunities
opportunities = analyzer.get_alpha_opportunities(
    order_book_data=order_book,
    trade_data=recent_trades,
    market_data=market_data
)

for opportunity in opportunities:
    print(f"Signal: {opportunity['signal']}")
    print(f"Confidence: {opportunity['confidence']:.3f}")
    print(f"Expected Return: {opportunity['expected_return']:.4f}")
    print(f"Risk Level: {opportunity['risk_level']}")
```

## Core Features

### Architecture Overview

```
Order Book Data → [Imbalance Module] → Feature Vector
Trade Data      → [Regime Module]    → Feature Vector  → [Feature Engine] → Alpha Signals
Market Data     → [Liquidity Module] → Feature Vector
```

### Feature Integration

The advanced features seamlessly integrate with existing order flow analysis:

```python
# Enhanced prediction with advanced features
prediction = analyzer.predict_order_flow(order_events, market_data)

# Now includes advanced features if enabled:
# - Regime-adjusted confidence scores
# - Liquidity-aware target levels
# - Imbalance-enhanced timing predictions
```

## Order Book Imbalance Momentum

### What It Does

Calculates sophisticated imbalance metrics that go beyond simple bid/ask ratios:

- **Weighted Imbalance**: Weights orders by their distance from mid-price
- **Exponential Decay**: Maintains historical memory with exponential decay
- **Momentum Scoring**: Detects acceleration in imbalance trends
- **Predictive Signals**: Generates forward-looking imbalance indicators

### Key Features

```python
from src.ml.advanced_features import OrderBookImbalanceMomentum

# Initialize module
imbalance_analyzer = OrderBookImbalanceMomentum(config)

# Calculate imbalance features
features = imbalance_analyzer.calculate_imbalance_features(order_book)

# Available features:
# - weighted_imbalance: Distance-weighted bid/ask imbalance
# - momentum_score: Rate of change in imbalance
# - exponential_decay_imbalance: Historical memory component
# - spread_impact: Imbalance effect on spread
```

### Configuration

```yaml
order_book_imbalance:
  max_depth_levels: 10          # How deep into order book to analyze
  distance_decay_rate: 0.1      # Weight decay by price distance
  momentum_window: 20           # Lookback for momentum calculation
  exponential_decay_factor: 0.95 # Historical memory decay
  min_update_interval: 1.0      # Minimum seconds between updates
```

### Interpretation

- **Weighted Imbalance > 0**: More buying pressure (bullish)
- **Weighted Imbalance < 0**: More selling pressure (bearish)
- **High Momentum Score**: Accelerating imbalance trend
- **Spread Impact**: How imbalance affects bid-ask spread

## Microstructure Regime Classification

### What It Does

Identifies market regimes using sophisticated statistical models:

- **Accumulation Regime**: Large players quietly building positions
- **Distribution Regime**: Large players quietly reducing positions
- **Neutral Regime**: Normal market activity
- **Trending Regime**: Strong directional movement
- **Volatile Regime**: High volatility, choppy conditions

### Key Features

```python
from src.ml.advanced_features import MicrostructureRegimeClassifier

# Initialize classifier
regime_classifier = MicrostructureRegimeClassifier(config)

# Classify current regime
regime_analysis = regime_classifier.classify_regime(
    order_data=recent_orders,
    trade_data=recent_trades,
    market_data=market_data
)

# Available insights:
# - current_regime: Current market regime
# - regime_probability: Confidence in classification
# - regime_duration: How long in current regime
# - transition_probability: Likelihood of regime change
```

### Regime Characteristics

**Accumulation Regime**:
- Large orders with minimal price impact
- Gradual volume increase
- Low cancellation rates
- Tight spreads

**Distribution Regime**:
- Large orders with controlled price impact
- Gradual volume decrease
- Moderate cancellation rates
- Widening spreads

**Trending Regime**:
- Strong price momentum
- High volume
- Low cancellation rates
- Directional order flow

**Volatile Regime**:
- High price volatility
- Erratic volume
- High cancellation rates
- Wide spreads

### Configuration

```yaml
regime_classification:
  hmm_components: 5             # Number of hidden states
  feature_window: 100           # Data points for feature calculation
  min_regime_duration: 30       # Minimum regime duration (minutes)
  transition_smoothing: 0.1     # Smoothing factor for regime transitions
  gb_n_estimators: 100          # Gradient boosting trees
  gb_max_depth: 6               # Maximum tree depth
```

## Liquidity Provision Patterns

### What It Does

Detects when market makers are "stepping away" before significant moves:

- **Spread Monitoring**: Tracks bid-ask spread changes
- **Depth Analysis**: Monitors order book depth reduction
- **Market Maker Confidence**: Assesses MM willingness to provide liquidity
- **Liquidity Stress**: Measures overall market liquidity health

### Key Features

```python
from src.ml.advanced_features import LiquidityProvisionPatterns

# Initialize analyzer
liquidity_analyzer = LiquidityProvisionPatterns(config)

# Analyze liquidity patterns
liquidity_analysis = liquidity_analyzer.analyze_liquidity_patterns(
    order_book_data=order_book,
    trade_data=recent_trades
)

# Available insights:
# - spread_z_score: Spread deviation from normal
# - depth_reduction: Order book depth changes
# - mm_confidence: Market maker confidence level
# - liquidity_stress: Overall liquidity stress score
```

### Interpretation

- **High Spread Z-Score**: Spreads unusually wide
- **Depth Reduction**: Market makers pulling liquidity
- **Low MM Confidence**: Market makers uncertain
- **High Liquidity Stress**: Potential for large price moves

### Configuration

```yaml
liquidity_patterns:
  spread_window: 50             # Window for spread analysis
  depth_levels: 5               # Order book levels to analyze
  mm_confidence_threshold: 0.3  # Threshold for low MM confidence
  liquidity_stress_threshold: 0.7 # Threshold for high stress
  update_frequency: 5           # Update frequency (seconds)
```

## Microstructure Prediction

### What It Does

Uses advanced deep learning models to predict order book microstructure changes:

- **LSTM Networks**: Sequence prediction of order book shapes
- **Transformer Models**: Multi-level price impact forecasting
- **Spread Evolution**: Predicts bid-ask spread changes
- **Order Arrival Rates**: Forecasts order flow patterns
- **Multi-Horizon**: Predictions from 1-10 seconds ahead

### Key Features

```python
from src.ml.advanced_features import MicrostructurePrediction

# Initialize predictor
predictor = MicrostructurePrediction(config)

# Make predictions
prediction_result = predictor.predict_microstructure(order_book_data)

# Available predictions:
# - predicted_spreads: Future spread values
# - predicted_mid_prices: Future mid-price movements
# - predicted_order_arrival_rates: Expected order frequency
# - predicted_cancel_ratios: Cancel-to-fill ratio forecasts
# - confidence_scores: Prediction confidence for each horizon
```

### Configuration

```yaml
microstructure_prediction:
  sequence_length: 100          # Input sequence length
  prediction_horizons: [1, 2, 5, 10]  # Prediction horizons (seconds)
  feature_size: 20              # Feature vector size
  hidden_size: 128              # Neural network hidden size
  num_layers: 3                 # Number of layers
  dropout: 0.2                  # Dropout rate
  batch_size: 32                # Training batch size
  learning_rate: 0.001          # Learning rate
  min_training_samples: 1000    # Minimum samples for training
```

### Model Types

**LSTM Network**:
- Bidirectional LSTM with attention mechanism
- Captures temporal dependencies in order book evolution
- Excellent for sequence prediction

**Transformer Model**:
- Multi-head attention mechanism
- Parallel processing of sequences
- Superior for complex pattern recognition

### Use Cases

1. **Entry Timing**: Predict optimal entry points based on spread evolution
2. **Order Sizing**: Adjust order sizes based on predicted liquidity
3. **Market Making**: Optimize quote placement using spread predictions
4. **Risk Management**: Anticipate adverse price movements

## Liquidity Shock Detection

### What It Does

Detects unusual order book states and potential liquidity shocks:

- **Isolation Forest**: Detects order book anomalies
- **Variational Autoencoders**: Unsupervised anomaly detection
- **Whale Movement Detection**: Identifies large order patterns
- **Funding Rate Shocks**: Monitors perpetual-spot basis changes
- **Ensemble Methods**: Combines multiple detection approaches

### Key Features

```python
from src.ml.advanced_features import LiquidityShockDetection

# Initialize detector
detector = LiquidityShockDetection(config)

# Add order book data
detector.add_order_book_data(order_book_data)

# Add funding rate data
detector.add_funding_rate_data(funding_data)

# Detect whale movements
whale_movements = detector.detect_whale_movements(order_book_data, trade_data)

# Get detection summary
summary = detector.get_detection_summary()
```

### Detection Types

**Liquidity Shock Events**:
- Severity levels: low, medium, high, critical
- Types: whale_movement, funding_shock, liquidation_cascade, market_maker_withdrawal
- Impact metrics: volume_impact, price_impact, anomaly_score

**Whale Movements**:
- Direction and estimated size
- Impact and stealth scores
- Urgency indicators

**Funding Rate Shocks**:
- Rate change magnitude
- Expected flow direction
- Impact probability

### Configuration

```yaml
liquidity_shock_detection:
  anomaly_threshold: 0.1        # Anomaly detection threshold
  whale_threshold: 100000       # USD value for whale detection
  funding_rate_threshold: 0.01  # 1% funding rate change threshold
  isolation_forest_contamination: 0.1  # Contamination rate
  vae_latent_dim: 10            # VAE latent dimension
  ensemble_threshold: 0.6       # Ensemble decision threshold
  feature_dim: 50               # Feature vector dimension
```

### Machine Learning Models

**Isolation Forest**:
- Unsupervised anomaly detection
- Efficient for high-dimensional data
- Robust to outliers

**Variational Autoencoder (VAE)**:
- Deep learning anomaly detection
- Learns normal market patterns
- Detects deviations from normal behavior

**Ensemble Approach**:
- Combines multiple detection methods
- Reduces false positives
- Improves detection accuracy

### Use Cases

1. **Risk Management**: Avoid trading during liquidity shocks
2. **Opportunity Detection**: Capitalize on whale movements
3. **Market Making**: Adjust spreads during anomalous conditions
4. **Portfolio Protection**: Hedge against funding rate shocks

## Advanced ML Feature Engine

### What It Does

Orchestrates all five modules and generates interaction features:

- **Parallel Processing**: Runs all modules simultaneously
- **Feature Interaction**: Combines signals from all modules
- **Alpha Generation**: Creates high-confidence trading signals
- **Risk Assessment**: Evaluates signal reliability

### Key Features

```python
from src.ml.advanced_features import AdvancedMLFeatureEngine

# Initialize engine
feature_engine = AdvancedMLFeatureEngine(config)

# Generate comprehensive features
features = feature_engine.generate_features(
    order_book_data=order_book,
    trade_data=recent_trades,
    market_data=market_data
)

# Features include:
# - Individual module features
# - Interaction features
# - Confidence scores
# - Risk assessments
```

### Interaction Features

The engine creates sophisticated interaction features:

- **Imbalance-Regime Alignment**: How well imbalance matches regime
- **Regime-Liquidity Stress**: Regime consistency with liquidity conditions
- **Three-Way Alpha**: Combined signal from all three modules

### Configuration

```yaml
feature_engine:
  enable_parallel_processing: true    # Process modules in parallel
  feature_interaction_depth: 2        # Depth of feature interactions
  confidence_threshold: 0.6           # Minimum confidence for signals
  risk_adjustment_factor: 1.2         # Risk adjustment multiplier
```

## Integration with Strategies

### Basic Integration

```python
from src.strategy_framework.strategy_types.order_flow_strategy import OrderFlowStrategy

class AdvancedOrderFlowStrategy(OrderFlowStrategy):
    def __init__(self, config):
        super().__init__(config)
        # Advanced features are automatically enabled if configured
        
    async def analyze_market_conditions(self):
        # Enhanced analysis with advanced features
        analysis = self.order_flow_analyzer.get_advanced_analysis(
            order_book_data=self.order_book,
            trade_data=self.recent_trades,
            market_data=self.market_data
        )
        
        return analysis
    
    async def generate_signals(self):
        # Get high-alpha opportunities
        opportunities = self.order_flow_analyzer.get_alpha_opportunities(
            order_book_data=self.order_book,
            trade_data=self.recent_trades,
            market_data=self.market_data
        )
        
        # Convert to trading signals
        signals = []
        for opportunity in opportunities:
            if opportunity['confidence'] > self.config.confidence_threshold:
                signals.append(self.create_signal_from_opportunity(opportunity))
        
        return signals
```

### Strategy Configuration

```yaml
strategy:
  name: "Advanced Order Flow Strategy"
  type: "src.strategies.advanced_order_flow_strategy.AdvancedOrderFlowStrategy"
  
  # Enable advanced features
  enable_advanced_features: true
  advanced_features_config_path: "config/ml/advanced_features_config.yaml"
  
  # Trading parameters
  confidence_threshold: 0.7
  max_position_size: 0.1
  risk_per_trade: 0.02
  
  # Advanced feature weights
  imbalance_weight: 0.3
  regime_weight: 0.4
  liquidity_weight: 0.3
```

## Configuration

### Complete Configuration Example

```yaml
# config/ml/advanced_features_config.yaml

advanced_features:
  # Global settings
  enable_logging: true
  log_level: "INFO"
  update_frequency: 1.0  # seconds
  
  # Order Book Imbalance Momentum
  order_book_imbalance:
    max_depth_levels: 10
    distance_decay_rate: 0.1
    momentum_window: 20
    exponential_decay_factor: 0.95
    min_update_interval: 1.0
    
  # Microstructure Regime Classification
  regime_classification:
    hmm_components: 5
    feature_window: 100
    min_regime_duration: 30
    transition_smoothing: 0.1
    gb_n_estimators: 100
    gb_max_depth: 6
    
  # Liquidity Provision Patterns
  liquidity_patterns:
    spread_window: 50
    depth_levels: 5
    mm_confidence_threshold: 0.3
    liquidity_stress_threshold: 0.7
    update_frequency: 5
    
  # Advanced ML Feature Engine
  feature_engine:
    enable_parallel_processing: true
    feature_interaction_depth: 2
    confidence_threshold: 0.6
    risk_adjustment_factor: 1.2
    
  # Signal generation
  signal_generation:
    min_confidence: 0.65
    max_signals_per_minute: 10
    signal_decay_time: 300  # seconds
```

## Usage Examples

### Example 1: Market Regime Detection

```python
# Detect current market regime and adapt strategy
async def adapt_to_market_regime():
    analysis = analyzer.get_advanced_analysis(order_book, trades, market_data)
    regime = analysis['regime']['current_regime']
    
    if regime == 'accumulation':
        # In accumulation, follow smart money
        strategy_params = {
            'follow_large_orders': True,
            'anticipate_breakouts': True,
            'reduce_noise_trading': True
        }
    elif regime == 'distribution':
        # In distribution, be cautious
        strategy_params = {
            'reduce_position_sizes': True,
            'tighten_stops': True,
            'avoid_breakout_trades': True
        }
    elif regime == 'volatile':
        # In volatile conditions, trade ranges
        strategy_params = {
            'use_mean_reversion': True,
            'widen_stops': True,
            'reduce_frequency': True
        }
    
    return strategy_params
```

### Example 2: Liquidity-Aware Entry Timing

```python
# Wait for optimal liquidity conditions before entering
async def time_entry_with_liquidity():
    opportunities = analyzer.get_alpha_opportunities(order_book, trades, market_data)
    
    for opportunity in opportunities:
        liquidity_score = opportunity['liquidity_score']
        
        if liquidity_score > 0.7:  # High liquidity
            # Enter immediately at market
            return {
                'entry_type': 'market',
                'size': 'full_position',
                'urgency': 'high'
            }
        elif liquidity_score > 0.4:  # Medium liquidity
            # Enter with limit orders
            return {
                'entry_type': 'limit',
                'size': 'half_position',
                'urgency': 'medium'
            }
        else:  # Low liquidity
            # Wait for better conditions
            return {
                'entry_type': 'wait',
                'reason': 'low_liquidity'
            }
```

### Example 3: Multi-Timeframe Analysis

```python
# Combine advanced features across timeframes
async def multi_timeframe_analysis():
    timeframes = ['1m', '5m', '15m']
    combined_analysis = {}
    
    for timeframe in timeframes:
        # Get data for specific timeframe
        tf_data = get_timeframe_data(timeframe)
        
        # Run advanced analysis
        analysis = analyzer.get_advanced_analysis(
            order_book_data=tf_data['order_book'],
            trade_data=tf_data['trades'],
            market_data=tf_data['market_data']
        )
        
        combined_analysis[timeframe] = analysis
    
    # Find confluence across timeframes
    confluence_signals = find_confluence(combined_analysis)
    
    return confluence_signals
```

### Example 4: Risk-Adjusted Position Sizing

```python
# Adjust position size based on advanced features
async def calculate_position_size():
    analysis = analyzer.get_advanced_analysis(order_book, trades, market_data)
    
    # Base position size
    base_size = 0.1  # 10% of capital
    
    # Adjust based on regime
    regime = analysis['regime']['current_regime']
    if regime == 'accumulation':
        regime_multiplier = 1.2  # Increase size
    elif regime == 'distribution':
        regime_multiplier = 0.8  # Decrease size
    else:
        regime_multiplier = 1.0
    
    # Adjust based on liquidity
    liquidity_stress = analysis['liquidity']['liquidity_stress']
    if liquidity_stress > 0.7:
        liquidity_multiplier = 0.5  # Reduce size in stressed conditions
    else:
        liquidity_multiplier = 1.0
    
    # Adjust based on imbalance confidence
    imbalance_confidence = analysis['imbalance']['confidence']
    imbalance_multiplier = min(1.5, max(0.5, imbalance_confidence * 2))
    
    # Final position size
    final_size = base_size * regime_multiplier * liquidity_multiplier * imbalance_multiplier
    
    return final_size
```

## Best Practices

### 1. Feature Selection

**Use All Features Initially**:
```python
# Start with all features enabled
config = {
    'enable_imbalance_features': True,
    'enable_regime_features': True,
    'enable_liquidity_features': True,
    'enable_interaction_features': True
}
```

**Then Optimize Based on Performance**:
```python
# Track feature importance
feature_importance = analyzer.get_feature_importance()

# Disable low-importance features
if feature_importance['regime_features'] < 0.1:
    config['enable_regime_features'] = False
```

### 2. Confidence Thresholds

**Start Conservative**:
```python
# High confidence threshold for live trading
config['confidence_threshold'] = 0.75
```

**Adjust Based on Backtesting**:
```python
# Lower threshold if missing too many opportunities
# Higher threshold if too many false signals
config['confidence_threshold'] = optimal_threshold
```

### 3. Risk Management

**Always Use Risk Limits**:
```python
# Maximum position size per signal
config['max_position_per_signal'] = 0.05

# Maximum total exposure
config['max_total_exposure'] = 0.2

# Stop loss based on liquidity
config['dynamic_stop_loss'] = True
```

### 4. Performance Monitoring

**Track Key Metrics**:
```python
# Monitor signal accuracy
signal_accuracy = track_signal_accuracy()

# Monitor feature stability
feature_stability = track_feature_stability()

# Monitor regime detection accuracy
regime_accuracy = track_regime_accuracy()
```

### 5. Continuous Learning

**Regular Model Updates**:
```python
# Retrain models with new data
if new_data_samples > 1000:
    analyzer.retrain_models()
```

**Feature Engineering**:
```python
# Add new features based on market evolution
new_features = engineer_new_features(market_data)
analyzer.add_features(new_features)
```

### 6. Common Pitfalls to Avoid

1. **Over-fitting to Recent Data**: Ensure sufficient historical data
2. **Ignoring Regime Changes**: Always consider regime context
3. **Neglecting Liquidity**: Account for liquidity in all decisions
4. **Static Thresholds**: Adapt thresholds to market conditions
5. **Signal Overlap**: Avoid redundant signals from multiple features

### 7. Optimization Tips

**Performance Optimization**:
```python
# Use parallel processing
config['enable_parallel_processing'] = True

# Optimize update frequencies
config['update_frequency'] = 2.0  # Balance speed vs accuracy

# Cache expensive calculations
config['enable_feature_caching'] = True
```

**Memory Management**:
```python
# Limit data history
config['max_history_length'] = 1000

# Clean up old data
config['auto_cleanup_interval'] = 3600  # seconds
```

This advanced ML features library provides the tools to extract alpha from market microstructure data. By combining order book imbalance momentum, regime classification, and liquidity analysis, you can build sophisticated trading strategies that adapt to changing market conditions and capture opportunities that traditional methods miss. 