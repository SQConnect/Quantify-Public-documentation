# Strategy Factory Documentation

## Overview
The Strategy Factory is a core component of our trading system that provides a centralized way to create and manage different types of trading strategies. It follows the Factory design pattern to encapsulate strategy creation logic and ensure consistent strategy initialization.

## Basic Usage

### Creating a Strategy
```python
from src.strategy_framework.strategy_factory import StrategyFactory

# Create a paper trading strategy
strategy = StrategyFactory.create_strategy(
    strategy_type="paper_trading",
    name="MyPaperStrategy",
    config={
        "parameters": {
            "position_size": 0.1,
            "ma_period": 20,
            "required_rising_candles": 3,
            "hold_candles": 2
        },
        "broker": {
            "api_key": "your_api_key",
            "api_secret": "your_api_secret"
        }
    }
)
```

## Available Strategy Types

### 1. Paper Trading Strategy
```python
config = {
    "parameters": {
        "position_size": 0.1,  # BTC position size
        "ma_period": 20,       # Moving average period
        "required_rising_candles": 3,  # Required candles for entry
        "hold_candles": 2      # Minimum hold period
    },
    "broker": {
        "api_key": "your_api_key",
        "api_secret": "your_api_secret"
    }
}

strategy = StrategyFactory.create_strategy(
    strategy_type="paper_trading",
    name="PaperStrategy",
    config=config
)
```

### 2. Live Trading Strategy
```python
config = {
    "parameters": {
        "position_size": 0.1,
        "risk_per_trade": 0.02,  # 2% risk per trade
        "max_positions": 3
    },
    "broker": {
        "api_key": "your_api_key",
        "api_secret": "your_api_secret"
    }
}

strategy = StrategyFactory.create_strategy(
    strategy_type="live_trading",
    name="LiveStrategy",
    config=config
)
```

## Strategy Configuration

### Common Parameters
```python
config = {
    "parameters": {
        # Position Sizing
        "position_size": 0.1,        # Fixed position size
        "risk_per_trade": 0.02,      # Risk per trade (2%)
        "max_positions": 3,          # Maximum concurrent positions
        
        # Technical Indicators
        "ma_period": 20,             # Moving average period
        "ema_fast": 7,               # Fast EMA period
        "ema_slow": 21,              # Slow EMA period
        
        # Entry/Exit Rules
        "required_rising_candles": 3, # Required candles for entry
        "hold_candles": 2,           # Minimum hold period
        "stop_loss_pct": 0.02,       # Stop loss percentage
        "take_profit_pct": 0.04      # Take profit percentage
    },
    
    # Broker Configuration
    "broker": {
        "api_key": "your_api_key",
        "api_secret": "your_api_secret",
        "testnet": True,             # Use testnet
        "paper_trading": True        # Enable paper trading
    },
    
    # Risk Management
    "risk_management": {
        "max_daily_loss": 0.05,      # Maximum daily loss (5%)
        "max_drawdown": 0.15,        # Maximum drawdown (15%)
        "position_sizing": "fixed"    # Position sizing method
    }
}
```

## Strategy Lifecycle

### 1. Initialization
```python
# Create and initialize strategy
strategy = StrategyFactory.create_strategy(
    strategy_type="paper_trading",
    name="MyStrategy",
    config=config
)

# Initialize strategy
await strategy.initialize()
```

### 2. Running the Strategy
```python
# Start the strategy
await strategy.start()

# Strategy will run until stopped
```

### 3. Stopping the Strategy
```python
# Stop the strategy
await strategy.stop()

# Cleanup resources
await strategy.cleanup()
```

## Error Handling

### Strategy Creation Errors
```python
try:
    strategy = StrategyFactory.create_strategy(
        strategy_type="invalid_type",
        name="MyStrategy",
        config=config
    )
except ValueError as e:
    logger.error(f"Failed to create strategy: {e}")
```

### Strategy Initialization Errors
```python
try:
    await strategy.initialize()
except Exception as e:
    logger.error(f"Failed to initialize strategy: {e}", exc_info=True)
```

## Best Practices

### 1. Strategy Configuration
- Use meaningful strategy names
- Validate configuration parameters
- Include all required parameters
- Document custom parameters

### 2. Error Handling
- Handle strategy creation errors
- Handle initialization errors
- Handle runtime errors
- Log all errors with context

### 3. Resource Management
- Properly initialize resources
- Clean up resources on stop
- Handle connection errors
- Monitor strategy health

### 4. Logging
```python
# Log strategy creation
logger.info(f"Creating strategy: {name} of type {strategy_type}")

# Log strategy initialization
logger.info(f"Initializing strategy: {name}")

# Log strategy start
logger.info(f"Starting strategy: {name}")

# Log strategy stop
logger.info(f"Stopping strategy: {name}")
```

## Monitoring and Debugging

### 1. Strategy Status
```python
# Check strategy status
status = await strategy.get_status()
logger.info(f"Strategy status: {status}")
```

### 2. Performance Metrics
```python
# Get performance metrics
metrics = await strategy.get_performance_metrics()
logger.info(f"Strategy performance: {metrics}")
```

### 3. Position Information
```python
# Get current positions
positions = await strategy.get_positions()
logger.info(f"Current positions: {positions}")
```

## Extending the Strategy Factory

### 1. Adding a New Strategy Type
```python
from src.strategy_framework.base_strategy import BaseStrategy

class MyCustomStrategy(BaseStrategy):
    def __init__(self, name: str, config: Dict[str, Any]):
        super().__init__(name, config)
        # Custom initialization

    async def initialize(self) -> bool:
        # Custom initialization logic
        return True

    async def on_market_data(self, event: MarketDataEvent):
        # Custom market data handling
        pass
```

### 2. Registering the Strategy
```python
# In strategy_factory.py
STRATEGY_TYPES = {
    "paper_trading": PaperTradingStrategy,
    "live_trading": LiveTradingStrategy,
    "custom": MyCustomStrategy  # Add new strategy type
}
```

## Troubleshooting

### 1. Strategy Creation Issues
- Verify strategy type exists
- Check configuration format
- Validate required parameters
- Check broker credentials

### 2. Initialization Issues
- Check broker connection
- Verify market data subscription
- Check resource availability
- Validate configuration values

### 3. Runtime Issues
- Monitor error logs
- Check strategy status
- Verify market data flow
- Check position updates 