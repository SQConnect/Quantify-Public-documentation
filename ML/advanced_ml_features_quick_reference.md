# Advanced ML Features Quick Reference

## 🚀 Quick Setup

```python
# Enable advanced features in OrderFlowAnalyzer
config = {
    'enable_advanced_features': True,
    'advanced_features_config_path': 'config/ml/advanced_features_config.yaml'
}
analyzer = OrderFlowAnalyzer(config)
```

## 📊 Core Analysis Methods

### Get Advanced Analysis
```python
# Comprehensive analysis of all three modules
analysis = analyzer.get_advanced_analysis(order_book, trades, market_data)

# Results structure:
{
    'imbalance': {
        'weighted_imbalance': 0.234,
        'momentum_score': 0.678,
        'confidence': 0.82
    },
    'regime': {
        'current_regime': 'accumulation',
        'regime_probability': 0.91,
        'regime_duration': 45  # minutes
    },
    'liquidity': {
        'spread_z_score': 2.1,
        'liquidity_stress': 0.34,
        'mm_confidence': 0.67
    }
}
```

### Get Alpha Opportunities
```python
# Get actionable trading opportunities
opportunities = analyzer.get_alpha_opportunities(order_book, trades, market_data)

# Each opportunity contains:
{
    'signal': 'buy',
    'confidence': 0.85,
    'expected_return': 0.0023,
    'risk_level': 'medium',
    'liquidity_score': 0.78,
    'regime_support': True
}
```

## 🎯 Signal Interpretation

### Imbalance Signals
```python
imbalance = analysis['imbalance']['weighted_imbalance']
momentum = analysis['imbalance']['momentum_score']

if imbalance > 0.1 and momentum > 0.5:
    # Strong buying pressure with acceleration
    signal = 'strong_buy'
elif imbalance < -0.1 and momentum > 0.5:
    # Strong selling pressure with acceleration
    signal = 'strong_sell'
```

### Regime Signals
```python
regime = analysis['regime']['current_regime']
probability = analysis['regime']['regime_probability']

if regime == 'accumulation' and probability > 0.8:
    # Follow smart money accumulation
    strategy = 'follow_large_orders'
elif regime == 'distribution' and probability > 0.8:
    # Be cautious during distribution
    strategy = 'reduce_risk'
```

### Liquidity Signals
```python
liquidity_stress = analysis['liquidity']['liquidity_stress']
mm_confidence = analysis['liquidity']['mm_confidence']

if liquidity_stress > 0.7 and mm_confidence < 0.3:
    # Market makers stepping away - potential big move
    signal = 'prepare_for_volatility'
```

## 📈 Strategy Integration Patterns

### Pattern 1: Regime-Aware Position Sizing
```python
def calculate_position_size(base_size, analysis):
    regime = analysis['regime']['current_regime']
    liquidity = analysis['liquidity']['liquidity_stress']
    
    # Adjust for regime
    if regime == 'accumulation':
        regime_mult = 1.2
    elif regime == 'distribution':
        regime_mult = 0.8
    else:
        regime_mult = 1.0
    
    # Adjust for liquidity
    liquidity_mult = max(0.5, 1.0 - liquidity)
    
    return base_size * regime_mult * liquidity_mult
```

### Pattern 2: Multi-Signal Confirmation
```python
def get_confirmed_signals(analysis):
    signals = []
    
    # Check all modules agree
    imbalance_signal = 'buy' if analysis['imbalance']['weighted_imbalance'] > 0.1 else 'sell'
    regime_signal = 'buy' if analysis['regime']['current_regime'] == 'accumulation' else 'sell'
    liquidity_ok = analysis['liquidity']['liquidity_stress'] < 0.5
    
    if imbalance_signal == regime_signal and liquidity_ok:
        signals.append({
            'signal': imbalance_signal,
            'confidence': min(analysis['imbalance']['confidence'], 
                             analysis['regime']['regime_probability']),
            'type': 'confirmed'
        })
    
    return signals
```

### Pattern 3: Dynamic Risk Management
```python
def adjust_stop_loss(base_stop, analysis):
    liquidity_stress = analysis['liquidity']['liquidity_stress']
    regime = analysis['regime']['current_regime']
    
    # Widen stops in stressed conditions
    if liquidity_stress > 0.7:
        stress_mult = 1.5
    else:
        stress_mult = 1.0
    
    # Tighten stops in stable regimes
    if regime in ['accumulation', 'distribution']:
        regime_mult = 0.8
    else:
        regime_mult = 1.0
    
    return base_stop * stress_mult * regime_mult
```

## 🔧 Common Configuration Patterns

### Conservative Setup
```yaml
advanced_features:
  order_book_imbalance:
    max_depth_levels: 5
    momentum_window: 30
  regime_classification:
    min_regime_duration: 60
    transition_smoothing: 0.2
  feature_engine:
    confidence_threshold: 0.8
```

### Aggressive Setup
```yaml
advanced_features:
  order_book_imbalance:
    max_depth_levels: 15
    momentum_window: 10
  regime_classification:
    min_regime_duration: 15
    transition_smoothing: 0.05
  feature_engine:
    confidence_threshold: 0.5
```

### High-Frequency Setup
```yaml
advanced_features:
  order_book_imbalance:
    min_update_interval: 0.5
  liquidity_patterns:
    update_frequency: 2
  feature_engine:
    enable_parallel_processing: true
```

## 📋 Feature Cheat Sheet

### Order Book Imbalance Features
| Feature | Range | Interpretation |
|---------|-------|----------------|
| `weighted_imbalance` | -1.0 to 1.0 | >0: Buy pressure, <0: Sell pressure |
| `momentum_score` | 0.0 to 1.0 | >0.5: Accelerating trend |
| `spread_impact` | 0.0 to 1.0 | >0.5: Imbalance affecting spread |

### Regime Classification
| Regime | Characteristics | Trading Approach |
|--------|----------------|------------------|
| `accumulation` | Smart money buying | Follow large orders |
| `distribution` | Smart money selling | Reduce risk |
| `trending` | Strong momentum | Trend following |
| `volatile` | High volatility | Range trading |
| `neutral` | Normal conditions | Standard strategies |

### Liquidity Patterns
| Metric | Threshold | Action |
|--------|-----------|--------|
| `liquidity_stress` | >0.7 | Reduce position sizes |
| `mm_confidence` | <0.3 | Expect volatility |
| `spread_z_score` | >2.0 | Avoid market orders |

## 🛠️ Debugging & Monitoring

### Check Feature Availability
```python
if analyzer.is_advanced_features_available():
    print("Advanced features enabled")
    
    # Get feature status
    status = analyzer.get_feature_status()
    print(f"Imbalance module: {status['imbalance_enabled']}")
    print(f"Regime module: {status['regime_enabled']}")
    print(f"Liquidity module: {status['liquidity_enabled']}")
```

### Monitor Performance
```python
# Track signal accuracy
metrics = analyzer.get_performance_metrics()
print(f"Signal accuracy: {metrics['signal_accuracy']:.2%}")
print(f"Feature stability: {metrics['feature_stability']:.2%}")
print(f"Regime accuracy: {metrics['regime_accuracy']:.2%}")
```

### Feature Importance
```python
# Get feature importance for model tuning
importance = analyzer.get_feature_importance()
print("Top features:")
for feature, score in importance.items():
    print(f"  {feature}: {score:.3f}")
```

## 🔄 Integration with Existing Strategies

### Enhance OrderFlowStrategy
```python
class EnhancedOrderFlowStrategy(OrderFlowStrategy):
    async def analyze_market_conditions(self):
        # Get standard analysis
        standard_analysis = await super().analyze_market_conditions()
        
        # Add advanced features if available
        if self.order_flow_analyzer.is_advanced_features_available():
            advanced_analysis = self.order_flow_analyzer.get_advanced_analysis(
                self.order_book, self.recent_trades, self.market_data
            )
            
            # Combine analyses
            return {**standard_analysis, 'advanced': advanced_analysis}
        
        return standard_analysis
```

### Add to Custom Strategy
```python
class MyCustomStrategy(BaseStrategy):
    def __init__(self, config):
        super().__init__(config)
        self.analyzer = OrderFlowAnalyzer({
            'enable_advanced_features': True,
            'advanced_features_config_path': 'config/ml/advanced_features_config.yaml'
        })
    
    async def generate_signals(self):
        opportunities = self.analyzer.get_alpha_opportunities(
            self.order_book, self.recent_trades, self.market_data
        )
        
        signals = []
        for opp in opportunities:
            if opp['confidence'] > self.config.min_confidence:
                signals.append(self.create_signal(opp))
        
        return signals
```

## 📊 Performance Optimization

### Parallel Processing
```python
# Enable parallel processing for better performance
config = {
    'feature_engine': {
        'enable_parallel_processing': True,
        'max_workers': 4
    }
}
```

### Memory Management
```python
# Limit data history to control memory usage
config = {
    'max_history_length': 1000,
    'auto_cleanup_interval': 3600
}
```

### Update Frequency Tuning
```python
# Balance accuracy vs performance
config = {
    'update_frequency': 2.0,  # 2 seconds
    'order_book_imbalance': {
        'min_update_interval': 1.0
    }
}
```

## 🎯 Success Metrics

### Key Performance Indicators
```python
# Track these metrics to measure success
metrics_to_track = [
    'signal_accuracy',      # % of profitable signals
    'regime_accuracy',      # % correct regime classifications
    'liquidity_prediction', # % accurate liquidity forecasts
    'feature_stability',    # Feature consistency over time
    'alpha_generation',     # Excess returns from features
    'risk_adjusted_return'  # Sharpe ratio improvement
]
```

### Benchmarking
```python
# Compare with and without advanced features
baseline_performance = run_strategy_without_advanced_features()
enhanced_performance = run_strategy_with_advanced_features()

improvement = (enhanced_performance - baseline_performance) / baseline_performance
print(f"Performance improvement: {improvement:.2%}")
```

## 🔍 Troubleshooting

### Common Issues
1. **Low Signal Accuracy**: Increase confidence threshold
2. **Too Few Signals**: Decrease confidence threshold
3. **High Memory Usage**: Reduce history length
4. **Slow Performance**: Enable parallel processing
5. **Inconsistent Features**: Check data quality

### Debug Mode
```python
# Enable debug logging
config = {
    'enable_logging': True,
    'log_level': 'DEBUG'
}
```

This quick reference provides the essential patterns and code snippets needed to effectively use the advanced ML features in your trading strategies. 