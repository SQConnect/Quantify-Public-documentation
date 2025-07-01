# PnL Tracker & Order Management Guide

The PnL Tracker and Order Logger are automatically integrated into all strategies inheriting from `BaseStrategy`. This provides comprehensive trade tracking, performance metrics calculation, and centralized order logging without requiring any code changes in individual strategies.

## Key Benefits

- **🔄 Automatic PnL Tracking**: Every trade is automatically tracked with entry/exit prices, P&L calculation
- **📊 Comprehensive Metrics**: Win rate, Sharpe ratio, drawdown, profit factor, and more
- **📝 Centralized Order Logging**: All order activity logged consistently across brokers
- **🚨 Telegram Alerts**: Optional notifications for trades and errors
- **🎯 Zero Broker Code**: Order logging removed from individual brokers for cleaner architecture

## Automatic Integration

The PnL tracker is automatically initialized when a strategy is created:

```python
class MyStrategy(BaseStrategy):
    def __init__(self, strategy_id: str, config: Dict[str, Any], ...):
        super().__init__(strategy_id, config, ...)
        # PnL tracker and order logger are automatically available:
        # self.pnl_tracker - for performance metrics
        # self.order_logger - for order logging (used internally)
```

## Automatic Trade Tracking

### Using place_tracked_order()

The `place_tracked_order()` method provides automatic PnL tracking AND order logging. It replaces direct broker calls and centralizes all order-related logic:

```python
class MyStrategy(BaseStrategy):
    async def execute_trade(self, signal):
        # This will automatically:
        # 1. Place the order through the broker
        # 2. Log the order (success/failure) with order logger
        # 3. Track the trade in PnL tracker
        # 4. Send Telegram alerts if configured
        result = await self.place_tracked_order(
            symbol=signal.symbol,
            order_type='market',
            side=signal.action,  # 'buy' or 'sell'
            quantity=signal.quantity,
            price=signal.price  # Optional for market orders
        )
        
        if result.get('error'):
            self.logger.error(f"Order failed: {result['error']}")
        else:
            self.logger.info(f"Order placed successfully: {result}")
```

### Manual Trade Tracking

If you need to track trades manually (e.g., for complex order types or external executions):

```python
class MyStrategy(BaseStrategy):
    async def manual_trade_tracking(self):
        # Record trade entry
        trade_id = self._track_trade_entry(
            symbol="ETH/USD",
            quantity=0.1,
            price=2500.0,
            side="buy",
            commission=5.0,
            metadata={"strategy": "my_strategy", "signal_type": "breakout"}
        )
        
        # Later, when the position is closed
        completed_trade = self._track_trade_exit(
            symbol="ETH/USD",
            exit_price=2600.0,
            commission=5.0
        )
        
        if completed_trade:
            self.logger.info(f"Trade closed with PnL: ${completed_trade.realized_pnl:.2f}")
```

### Order Cancellation with Logging

```python
class MyStrategy(BaseStrategy):
    async def cancel_order_example(self, order_id: str, symbol: str):
        # Cancel order with automatic logging
        success = await self.cancel_tracked_order(order_id, symbol)
        
        if success:
            self.logger.info(f"Order {order_id} cancelled successfully")
        else:
            self.logger.error(f"Failed to cancel order {order_id}")
```

### Manual Trade Exit Logging

```python
class MyStrategy(BaseStrategy):
    async def close_position_manually(self, symbol: str, exit_price: float):
        # For complex exit scenarios, manually log the trade exit
        self.log_trade_exit(
            symbol=symbol,
            exit_price=exit_price,
            quantity=0.1,
            order_id="manual_exit_123"
        )
```

## Accessing Performance Metrics

### Get Current Performance Metrics

```python
class MyStrategy(BaseStrategy):
    async def log_performance(self):
        # Get comprehensive performance metrics
        metrics = self.get_pnl_metrics()
        
        self.logger.info(f"Total Trades: {metrics['total_trades']}")
        self.logger.info(f"Win Rate: {metrics['win_rate_pct']:.1f}%")
        self.logger.info(f"Total PnL: ${metrics['total_pnl']:.2f}")
        self.logger.info(f"Sharpe Ratio: {metrics['sharpe_ratio']:.2f}")
        self.logger.info(f"Max Drawdown: {metrics['max_drawdown_pct']:.1f}%")
        self.logger.info(f"Profit Factor: {metrics['profit_factor']:.2f}")
```

### Available Metrics

The PnL tracker provides the following metrics:

- **Trade Statistics**: `total_trades`, `winning_trades`, `losing_trades`, `win_rate`, `win_rate_pct`
- **PnL Metrics**: `total_pnl`, `gross_profit`, `gross_loss`, `profit_factor`
- **Risk Metrics**: `max_drawdown`, `max_drawdown_pct`, `sharpe_ratio`, `sortino_ratio`
- **Capital Metrics**: `initial_capital`, `current_capital`, `total_return_pct`
- **Position Info**: `open_positions_count`

### Get Trade History

```python
class MyStrategy(BaseStrategy):
    def analyze_recent_trades(self):
        # Get last 10 trades
        recent_trades = self.get_trade_history(limit=10)
        
        for trade in recent_trades:
            self.logger.info(f"Trade {trade.trade_id}: "
                           f"{trade.side} {trade.quantity} {trade.symbol} @ {trade.entry_price} "
                           f"-> {trade.exit_price} = ${trade.realized_pnl:.2f}")
```

### Get Open Positions

```python
class MyStrategy(BaseStrategy):
    def check_open_positions(self):
        open_positions = self.get_open_positions_pnl()
        
        for symbol, trade in open_positions.items():
            self.logger.info(f"Open position: {trade.side} {trade.quantity} {symbol} "
                           f"@ {trade.entry_price} (Duration: {trade.duration_hours:.1f}h)")
```

## Performance Reporting

### Export Comprehensive Report

```python
class MyStrategy(BaseStrategy):
    async def generate_performance_report(self):
        # Export detailed performance report
        report = self.export_pnl_report()
        
        # Save to file
        import json
        with open(f"performance_report_{self.strategy_name}.json", "w") as f:
            json.dump(report, f, indent=2, default=str)
        
        self.logger.info(f"Performance report saved for {self.strategy_name}")
```

### Integration with get_performance_snapshot()

The base strategy's `get_performance_snapshot()` method automatically includes PnL tracker metrics:

```python
class MyStrategy(BaseStrategy):
    async def log_status(self):
        # This now includes both traditional metrics and PnL tracker metrics
        snapshot = self.get_performance_snapshot()
        
        self.logger.info(f"Strategy Status: {snapshot}")
```

## Best Practices

### 1. Use place_tracked_order() for Automatic Tracking

```python
# Good - Automatic tracking
result = await self.place_tracked_order(
    symbol="BTC/USD",
    order_type="market",
    side="buy",
    quantity=0.01
)

# Avoid - Manual broker calls (unless you handle tracking separately)
result = await self.broker.place_order(...)  # Won't be automatically tracked
```

### 2. Update Unrealized PnL Regularly

```python
class MyStrategy(BaseStrategy):
    async def on_live_candle(self, event):
        # Update unrealized PnL with current market prices
        for symbol in self.symbols:
            current_price = event.candle.close
            self._update_unrealized_pnl(symbol, current_price)
```

### 3. Monitor Performance Metrics

```python
class MyStrategy(BaseStrategy):
    async def check_performance_limits(self):
        metrics = self.get_pnl_metrics()
        
        # Stop strategy if drawdown is too high
        if metrics['max_drawdown_pct'] > 15.0:
            self.logger.warning(f"Max drawdown exceeded: {metrics['max_drawdown_pct']:.1f}%")
            await self.stop()
        
        # Log performance every 10 trades
        if metrics['total_trades'] % 10 == 0:
            self.logger.info(f"Performance update: Win rate {metrics['win_rate_pct']:.1f}%, "
                           f"PnL ${metrics['total_pnl']:.2f}")
```

## Configuration

The PnL tracker uses the strategy's `initial_capital` from the configuration:

```yaml
# config/strategies/my_strategy.yaml
name: "MyStrategy"
initial_capital: 50000  # PnL tracker will use this value
symbols: ["ETH/USD"]
# ... other config
```

## Advanced Usage

### Custom Metadata

```python
# Track additional information with trades
trade_id = self._track_trade_entry(
    symbol="ETH/USD",
    quantity=0.1,
    price=2500.0,
    side="buy",
    metadata={
        "signal_type": "breakout",
        "confidence": 0.85,
        "stop_loss": 2400.0,
        "take_profit": 2700.0
    }
)
```

### Backtesting Integration

```python
class BacktestStrategy(BaseStrategy):
    def reset_for_backtest(self):
        # Reset PnL tracker for new backtest run
        self.pnl_tracker.reset_tracker(new_initial_capital=100000)
```

The PnL tracker provides a comprehensive, automatic solution for tracking trading performance without requiring any manual intervention from strategy developers. Simply inherit from `BaseStrategy` and use `place_tracked_order()` for automatic trade tracking and performance monitoring.

## Updated Strategy Examples

The following strategies have been updated to use the centralized order management system:

### 1. Sandwich Strategy (`src/strategy_framework/strategy_types/sandwich_strategy.py`)
- Updated to use `self.place_tracked_order()` for automatic PnL tracking and order logging
- Updated to use `self.cancel_tracked_order()` for order cancellations
- Provides automatic performance monitoring for regime-based trading

### 2. Elliott Wave Strategy (`src/strategies/elliott_wave_strategy.py`)
- Contains example code showing how to use `self.place_tracked_order()` 
- Demonstrates integration with technical analysis patterns
- Currently analysis-only but shows proper order placement patterns

### 3. Short Put Spread Strategy (`src/options/strategies/short_put_spread.py`)
- Uses legacy signal-based approach with ExecutionManager
- Contains commented example of centralized approach using `self.place_tracked_multi_leg_order()`
- Demonstrates multi-leg options strategy integration

### 4. All Brokers Updated
- **Binance Broker**: Removed order logging (now centralized)
- **Kraken Broker**: Removed order logging (now centralized)  
- **Saxo Broker**: Removed order logging (now centralized)
- **Options Helper**: Removed duplicate order logging

### Migration Guide

For existing strategies, update order placement calls:

```python
# Old approach
result = await self.broker.place_order(symbol, order_type, side, quantity, price)

# New centralized approach
result = await self.place_tracked_order(symbol, order_type, side, quantity, price)
```

For options strategies with multi-leg orders:

```python
# Old approach (still works via ExecutionManager)
signal = Signal(action='place_strategy', metadata={'legs': legs})
await self.event_manager.publish(CoreEventType.SIGNAL.value, SignalEvent(signal))

# New centralized approach (recommended)
result = await self.place_tracked_multi_leg_order(legs, 'Limit', price)
```

This centralized architecture provides:
- **Consistent logging** across all brokers and strategies
- **Automatic PnL tracking** for all trades
- **Centralized order management** in BaseStrategy
- **Clean broker separation** - brokers focus only on API communication
- **Zero code changes** required in most strategies - just change the method call 