# Advanced ML Strategy Example

This document provides a complete example of how to build a sophisticated trading strategy using the advanced ML features. The example demonstrates real-world usage patterns and best practices.

## Example Strategy: Smart Money Follower

This strategy uses advanced ML features to identify and follow smart money movements while avoiding liquidity traps.

### Strategy Logic

1. **Regime Detection**: Identify accumulation/distribution regimes
2. **Imbalance Analysis**: Detect order book imbalances 
3. **Liquidity Assessment**: Ensure sufficient liquidity for execution
4. **Signal Combination**: Generate high-confidence trading signals
5. **Dynamic Risk Management**: Adjust position sizes based on market conditions

### Complete Strategy Implementation

```python
import asyncio
import logging
from typing import Dict, List, Optional
from datetime import datetime, timedelta

from src.strategy_framework.strategy_types.order_flow_strategy import OrderFlowStrategy
from src.ml.order_flow_analyzer import OrderFlowAnalyzer
from src.signal_generation.signal import Signal, SignalType
from src.data_manager.market_data import MarketData

class SmartMoneyFollowerStrategy(OrderFlowStrategy):
    """
    Advanced ML strategy that follows smart money movements using
    order book imbalance, regime classification, and liquidity analysis.
    """
    
    def __init__(self, config: Dict):
        super().__init__(config)
        
        # Initialize advanced ML analyzer
        self.analyzer = OrderFlowAnalyzer({
            'model_type': 'xgboost',
            'enable_advanced_features': True,
            'advanced_features_config_path': 'config/ml/advanced_features_config.yaml',
            'large_order_threshold': config.get('large_order_threshold', 1000)
        })
        
        # Strategy parameters
        self.confidence_threshold = config.get('confidence_threshold', 0.75)
        self.max_position_size = config.get('max_position_size', 0.1)
        self.min_liquidity_score = config.get('min_liquidity_score', 0.6)
        self.regime_weight = config.get('regime_weight', 0.4)
        self.imbalance_weight = config.get('imbalance_weight', 0.3)
        self.liquidity_weight = config.get('liquidity_weight', 0.3)
        
        # Risk management
        self.max_drawdown = config.get('max_drawdown', 0.05)
        self.stop_loss_base = config.get('stop_loss_base', 0.02)
        self.take_profit_base = config.get('take_profit_base', 0.04)
        
        # Signal history for analysis
        self.signal_history = []
        self.regime_history = []
        self.performance_metrics = {
            'signals_generated': 0,
            'signals_executed': 0,
            'profitable_signals': 0,
            'average_confidence': 0.0
        }
        
        self.logger = logging.getLogger(__name__)
    
    async def initialize(self):
        """Initialize strategy components."""
        await super().initialize()
        
        # Load or initialize ML models
        try:
            await self.analyzer.load_model()
            self.logger.info("Advanced ML models loaded successfully")
        except Exception as e:
            self.logger.warning(f"Could not load ML models: {e}")
            self.logger.info("Models will be trained with incoming data")
    
    async def analyze_market_conditions(self) -> Dict:
        """Analyze current market conditions using advanced ML features."""
        try:
            # Get advanced analysis
            analysis = self.analyzer.get_advanced_analysis(
                order_book_data=self.order_book,
                trade_data=self.recent_trades,
                market_data=self.market_data
            )
            
            # Store regime history
            self.regime_history.append({
                'timestamp': datetime.now(),
                'regime': analysis['regime']['current_regime'],
                'probability': analysis['regime']['regime_probability']
            })
            
            # Calculate overall market score
            market_score = self._calculate_market_score(analysis)
            
            return {
                'analysis': analysis,
                'market_score': market_score,
                'regime_stable': analysis['regime']['regime_probability'] > 0.8,
                'liquidity_healthy': analysis['liquidity']['liquidity_stress'] < 0.5,
                'imbalance_significant': abs(analysis['imbalance']['weighted_imbalance']) > 0.1
            }
            
        except Exception as e:
            self.logger.error(f"Error in market analysis: {e}")
            return {'error': str(e)}
    
    def _calculate_market_score(self, analysis: Dict) -> float:
        """Calculate overall market opportunity score."""
        # Regime score (0-1)
        regime_score = self._get_regime_score(analysis['regime'])
        
        # Imbalance score (0-1)
        imbalance_score = min(1.0, abs(analysis['imbalance']['weighted_imbalance']) * 2)
        
        # Liquidity score (0-1)
        liquidity_score = max(0.0, 1.0 - analysis['liquidity']['liquidity_stress'])
        
        # Weighted combination
        market_score = (
            regime_score * self.regime_weight +
            imbalance_score * self.imbalance_weight +
            liquidity_score * self.liquidity_weight
        )
        
        return market_score
    
    def _get_regime_score(self, regime_data: Dict) -> float:
        """Convert regime information to opportunity score."""
        regime = regime_data['current_regime']
        probability = regime_data['regime_probability']
        
        # Score based on regime type
        regime_scores = {
            'accumulation': 0.9,  # High opportunity
            'distribution': 0.7,  # Medium opportunity (short)
            'trending': 0.8,      # Good opportunity
            'volatile': 0.3,      # Low opportunity
            'neutral': 0.5        # Neutral
        }
        
        base_score = regime_scores.get(regime, 0.5)
        
        # Adjust by probability
        return base_score * probability
    
    async def generate_signals(self) -> List[Signal]:
        """Generate trading signals using advanced ML features."""
        signals = []
        
        try:
            # Get market conditions
            market_conditions = await self.analyze_market_conditions()
            
            if 'error' in market_conditions:
                return signals
            
            # Get alpha opportunities
            opportunities = self.analyzer.get_alpha_opportunities(
                order_book_data=self.order_book,
                trade_data=self.recent_trades,
                market_data=self.market_data
            )
            
            # Process each opportunity
            for opportunity in opportunities:
                signal = self._process_opportunity(opportunity, market_conditions)
                if signal:
                    signals.append(signal)
            
            # Update performance metrics
            self.performance_metrics['signals_generated'] += len(signals)
            if signals:
                avg_confidence = sum(s.confidence for s in signals) / len(signals)
                self.performance_metrics['average_confidence'] = avg_confidence
            
            return signals
            
        except Exception as e:
            self.logger.error(f"Error generating signals: {e}")
            return signals
    
    def _process_opportunity(self, opportunity: Dict, market_conditions: Dict) -> Optional[Signal]:
        """Process a single trading opportunity."""
        # Check confidence threshold
        if opportunity['confidence'] < self.confidence_threshold:
            return None
        
        # Check liquidity requirements
        if opportunity.get('liquidity_score', 0) < self.min_liquidity_score:
            return None
        
        # Check market conditions
        if not market_conditions['regime_stable']:
            # Lower confidence during regime transitions
            opportunity['confidence'] *= 0.8
        
        # Calculate position size
        position_size = self._calculate_position_size(opportunity, market_conditions)
        
        # Calculate risk parameters
        stop_loss, take_profit = self._calculate_risk_parameters(
            opportunity, market_conditions
        )
        
        # Create signal
        signal = Signal(
            signal_type=SignalType.BUY if opportunity['signal'] == 'buy' else SignalType.SELL,
            symbol=self.symbol,
            price=self.current_price,
            quantity=position_size,
            confidence=opportunity['confidence'],
            stop_loss=stop_loss,
            take_profit=take_profit,
            metadata={
                'strategy': 'smart_money_follower',
                'regime': market_conditions['analysis']['regime']['current_regime'],
                'imbalance': market_conditions['analysis']['imbalance']['weighted_imbalance'],
                'liquidity_stress': market_conditions['analysis']['liquidity']['liquidity_stress'],
                'market_score': market_conditions['market_score'],
                'expected_return': opportunity.get('expected_return', 0),
                'risk_level': opportunity.get('risk_level', 'medium')
            }
        )
        
        # Store signal for analysis
        self.signal_history.append({
            'timestamp': datetime.now(),
            'signal': signal,
            'opportunity': opportunity,
            'market_conditions': market_conditions
        })
        
        return signal
    
    def _calculate_position_size(self, opportunity: Dict, market_conditions: Dict) -> float:
        """Calculate position size based on opportunity and market conditions."""
        # Base position size
        base_size = self.max_position_size
        
        # Adjust for confidence
        confidence_mult = opportunity['confidence']
        
        # Adjust for regime
        regime = market_conditions['analysis']['regime']['current_regime']
        if regime == 'accumulation':
            regime_mult = 1.2  # Increase size during accumulation
        elif regime == 'distribution':
            regime_mult = 0.8  # Reduce size during distribution
        elif regime == 'volatile':
            regime_mult = 0.6  # Reduce size during volatility
        else:
            regime_mult = 1.0
        
        # Adjust for liquidity
        liquidity_stress = market_conditions['analysis']['liquidity']['liquidity_stress']
        liquidity_mult = max(0.5, 1.0 - liquidity_stress)
        
        # Adjust for market score
        market_score_mult = market_conditions['market_score']
        
        # Calculate final size
        final_size = base_size * confidence_mult * regime_mult * liquidity_mult * market_score_mult
        
        # Ensure minimum and maximum bounds
        final_size = max(0.01, min(final_size, self.max_position_size))
        
        return final_size
    
    def _calculate_risk_parameters(self, opportunity: Dict, market_conditions: Dict) -> tuple:
        """Calculate stop loss and take profit levels."""
        current_price = self.current_price
        
        # Base risk parameters
        stop_loss_pct = self.stop_loss_base
        take_profit_pct = self.take_profit_base
        
        # Adjust for liquidity stress
        liquidity_stress = market_conditions['analysis']['liquidity']['liquidity_stress']
        if liquidity_stress > 0.7:
            # Widen stops in illiquid conditions
            stop_loss_pct *= 1.5
            take_profit_pct *= 1.3
        
        # Adjust for regime
        regime = market_conditions['analysis']['regime']['current_regime']
        if regime == 'volatile':
            # Widen stops in volatile conditions
            stop_loss_pct *= 1.3
            take_profit_pct *= 1.2
        elif regime in ['accumulation', 'distribution']:
            # Tighten stops in stable regimes
            stop_loss_pct *= 0.8
            take_profit_pct *= 0.9
        
        # Calculate levels
        if opportunity['signal'] == 'buy':
            stop_loss = current_price * (1 - stop_loss_pct)
            take_profit = current_price * (1 + take_profit_pct)
        else:
            stop_loss = current_price * (1 + stop_loss_pct)
            take_profit = current_price * (1 - take_profit_pct)
        
        return stop_loss, take_profit
    
    async def on_signal_executed(self, signal: Signal, execution_result: Dict):
        """Handle signal execution."""
        self.performance_metrics['signals_executed'] += 1
        
        # Log execution
        self.logger.info(f"Signal executed: {signal.signal_type.value} "
                        f"at {execution_result.get('price', 'N/A')} "
                        f"with confidence {signal.confidence:.3f}")
    
    async def on_trade_closed(self, trade_result: Dict):
        """Handle trade closure and update performance."""
        if trade_result.get('profit', 0) > 0:
            self.performance_metrics['profitable_signals'] += 1
        
        # Update ML models with trade result
        await self.analyzer.update_performance(trade_result)
    
    def get_performance_summary(self) -> Dict:
        """Get strategy performance summary."""
        executed = self.performance_metrics['signals_executed']
        profitable = self.performance_metrics['profitable_signals']
        
        return {
            'signals_generated': self.performance_metrics['signals_generated'],
            'signals_executed': executed,
            'profitable_signals': profitable,
            'win_rate': profitable / executed if executed > 0 else 0,
            'average_confidence': self.performance_metrics['average_confidence'],
            'regime_distribution': self._get_regime_distribution(),
            'recent_signals': len(self.signal_history[-100:])  # Last 100 signals
        }
    
    def _get_regime_distribution(self) -> Dict:
        """Get distribution of regimes in recent history."""
        recent_regimes = [r['regime'] for r in self.regime_history[-100:]]
        
        if not recent_regimes:
            return {}
        
        regime_counts = {}
        for regime in recent_regimes:
            regime_counts[regime] = regime_counts.get(regime, 0) + 1
        
        total = len(recent_regimes)
        return {regime: count/total for regime, count in regime_counts.items()}

# Example usage and configuration
def create_smart_money_strategy():
    """Create and configure the Smart Money Follower strategy."""
    config = {
        # Basic parameters
        'symbol': 'BTC/USD',
        'timeframe': '1m',
        'confidence_threshold': 0.75,
        'max_position_size': 0.1,
        'min_liquidity_score': 0.6,
        
        # Feature weights
        'regime_weight': 0.4,
        'imbalance_weight': 0.3,
        'liquidity_weight': 0.3,
        
        # Risk management
        'max_drawdown': 0.05,
        'stop_loss_base': 0.02,
        'take_profit_base': 0.04,
        
        # ML parameters
        'large_order_threshold': 1000,
        'advanced_features_config_path': 'config/ml/advanced_features_config.yaml'
    }
    
    return SmartMoneyFollowerStrategy(config)

# Example advanced features configuration
ADVANCED_FEATURES_CONFIG = """
advanced_features:
  # Global settings
  enable_logging: true
  log_level: "INFO"
  update_frequency: 1.0

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
    signal_decay_time: 300
"""

# Example monitoring and analysis
async def monitor_strategy_performance():
    """Monitor strategy performance and advanced features."""
    strategy = create_smart_money_strategy()
    
    while True:
        try:
            # Get performance summary
            performance = strategy.get_performance_summary()
            
            # Get advanced features status
            if strategy.analyzer.is_advanced_features_available():
                feature_status = strategy.analyzer.get_feature_status()
                feature_importance = strategy.analyzer.get_feature_importance()
                
                print(f"Strategy Performance:")
                print(f"  Win Rate: {performance['win_rate']:.2%}")
                print(f"  Average Confidence: {performance['average_confidence']:.3f}")
                print(f"  Signals Generated: {performance['signals_generated']}")
                
                print(f"\nAdvanced Features Status:")
                print(f"  Imbalance Module: {feature_status['imbalance_enabled']}")
                print(f"  Regime Module: {feature_status['regime_enabled']}")
                print(f"  Liquidity Module: {feature_status['liquidity_enabled']}")
                
                print(f"\nTop Features:")
                for feature, importance in list(feature_importance.items())[:5]:
                    print(f"  {feature}: {importance:.3f}")
                
                print(f"\nRegime Distribution:")
                for regime, pct in performance['regime_distribution'].items():
                    print(f"  {regime}: {pct:.1%}")
            
            await asyncio.sleep(300)  # Update every 5 minutes
            
        except Exception as e:
            print(f"Error in monitoring: {e}")
            await asyncio.sleep(60)

if __name__ == "__main__":
    # Run the monitoring
    asyncio.run(monitor_strategy_performance())
```

## Key Features of This Strategy

### 1. Multi-Modal Analysis
- **Regime Detection**: Identifies market phases (accumulation, distribution, etc.)
- **Imbalance Analysis**: Tracks order book imbalances and momentum
- **Liquidity Assessment**: Monitors market maker behavior and liquidity stress

### 2. Dynamic Risk Management
- **Regime-Aware Position Sizing**: Adjusts position sizes based on market regime
- **Liquidity-Adjusted Stops**: Widens stops during illiquid conditions
- **Confidence-Based Sizing**: Scales position size with signal confidence

### 3. Advanced Signal Processing
- **Multi-Factor Scoring**: Combines signals from all advanced features
- **Confidence Filtering**: Only trades high-confidence opportunities
- **Market Condition Awareness**: Adapts to current market conditions

### 4. Performance Monitoring
- **Real-time Metrics**: Tracks strategy performance continuously
- **Feature Importance**: Monitors which features drive performance
- **Regime Analysis**: Analyzes performance across different market regimes

## Configuration Examples

### Conservative Configuration
```yaml
strategy:
  confidence_threshold: 0.8
  max_position_size: 0.05
  min_liquidity_score: 0.7
  regime_weight: 0.5
  stop_loss_base: 0.015
  take_profit_base: 0.03
```

### Aggressive Configuration  
```yaml
strategy:
  confidence_threshold: 0.6
  max_position_size: 0.15
  min_liquidity_score: 0.4
  imbalance_weight: 0.4
  stop_loss_base: 0.025
  take_profit_base: 0.05
```

### High-Frequency Configuration
```yaml
strategy:
  confidence_threshold: 0.7
  max_position_size: 0.08
  min_liquidity_score: 0.6
  update_frequency: 0.5
  signal_decay_time: 60
```

## Performance Optimization Tips

1. **Feature Selection**: Monitor feature importance and disable low-impact features
2. **Parameter Tuning**: Adjust confidence thresholds based on backtesting results
3. **Regime Specialization**: Consider different parameters for different regimes
4. **Liquidity Filtering**: Ensure sufficient liquidity for all trades
5. **Continuous Learning**: Regularly retrain models with new data

This example demonstrates how to build a sophisticated trading strategy that leverages all the advanced ML features while maintaining proper risk management and performance monitoring. 