# Logging System Documentation

## Overview
Our logging system provides a structured way to track and debug application behavior, especially for trading strategies. It uses a custom logger factory that creates consistent, formatted logs across the application.

## Basic Usage

### Getting a Logger
```python
from src.logging_stats.logger_factory import get_logger

# Get a logger for your module
logger = get_logger(__name__, "YourComponentName")
```

### Log Levels
```python
# Different log levels
logger.debug("Detailed information for debugging")
logger.info("General information about program execution")
logger.warning("Warning messages for potentially problematic situations")
logger.error("Error messages for serious problems")
logger.critical("Critical messages for fatal errors")
```

## Log Format
Our logs follow this format:
```
YYYY-MM-DD HH:MM:SS | LEVEL | module_name_ComponentName | Thread-ID | Message
```

Example:
```
2025-06-02 22:07:43 | INFO | strategy_framework.strategy_types.paper_trading_strategy_PaperTradingStrategy | Thread-8533597952 | Received market data - Symbol: XBT/USD, Price: 104041.00
```

## Best Practices

### 1. Component Identification
Always include a component name when getting a logger:
```python
# Good
logger = get_logger(__name__, "PaperTradingStrategy")

# Avoid
logger = get_logger(__name__)  # Missing component name
```

### 2. Log Levels Usage
- `DEBUG`: Detailed information for debugging
  ```python
  logger.debug(f"Processing tick data: {tick_data}")
  ```

- `INFO`: General operational information
  ```python
  logger.info(f"Order placed successfully - ID: {order_id}")
  ```

- `WARNING`: Unexpected but handled situations
  ```python
  logger.warning(f"Order not filled. Status: {order_status}")
  ```

- `ERROR`: Serious problems that need attention
  ```python
  logger.error(f"Failed to connect to broker: {error}", exc_info=True)
  ```

- `CRITICAL`: Fatal errors that may crash the application
  ```python
  logger.critical("Database connection lost", exc_info=True)
  ```

### 3. Exception Logging
Always include `exc_info=True` when logging exceptions:
```python
try:
    # Your code
except Exception as e:
    logger.error(f"Error occurred: {e}", exc_info=True)
```

### 4. Structured Logging
Use f-strings for complex log messages:
```python
logger.info(
    f"Trading Conditions:\n"
    f"- Current Price: {price:.2f}\n"
    f"- PMA: {pma:.2f}\n"
    f"- Position: {'In' if in_position else 'Out'}"
)
```

### 5. Performance Metrics
Log performance-related information:
```python
logger.info(f"Order execution time: {execution_time:.2f}ms")
```

## Common Patterns

### 1. Market Data Logging
```python
logger.info(f"Received market data - Symbol: {symbol}, Price: {price:.2f}, Volume: {volume:.2f}")
```

### 2. Order Logging
```python
logger.info(f"Order placed - ID: {order_id}, Status: {status}, Price: {price:.2f}")
```

### 3. Strategy State Logging
```python
logger.info(
    f"Strategy State:\n"
    f"- Position: {'In' if in_position else 'Out'}\n"
    f"- Entry Price: {entry_price:.2f}\n"
    f"- P&L: {pnl:.2f}%"
)
```

### 4. Error Handling
```python
try:
    await broker.place_order(order)
except Exception as e:
    logger.error(f"Failed to place order: {e}", exc_info=True)
    logger.error(f"Order details: {json.dumps(order, indent=2)}")
```

## Log File Management

### Location
Logs are stored in the `logs` directory with the following structure:
```
logs/
  ├── strategy/
  │   ├── paper_trading/
  │   └── live_trading/
  ├── broker/
  └── system/
```

### Rotation
Log files are automatically rotated when they reach 10MB, keeping up to 5 backup files.

## Debugging Tips

1. **Enable Debug Logging**
   ```python
   # In your configuration
   logging_level = "DEBUG"
   ```

2. **Filter Logs**
   ```bash
   # Filter logs by component
   grep "PaperTradingStrategy" logs/strategy/paper_trading.log
   
   # Filter logs by level
   grep "ERROR" logs/strategy/paper_trading.log
   ```

3. **Monitor Live Logs**
   ```bash
   tail -f logs/strategy/paper_trading.log
   ```

## Best Practices for Strategy Logging

1. **Entry/Exit Points**
   ```python
   # Entry
   logger.info(f"Entry conditions met - Price: {price:.2f}, PMA: {pma:.2f}")
   
   # Exit
   logger.info(f"Exit conditions met - P&L: {pnl:.2f}%")
   ```

2. **Indicator Updates**
   ```python
   logger.info(f"Indicator Update - EMA: {ema:.2f}, PMA: {pma:.2f}")
   ```

3. **Position Management**
   ```python
   logger.info(f"Position Update - Size: {size}, Entry: {entry:.2f}, Current: {current:.2f}")
   ```

4. **Performance Metrics**
   ```python
   logger.info(f"Performance - Win Rate: {win_rate:.2%}, P&L: {pnl:.2f}")
   ```

## Troubleshooting

1. **No Logs Appearing**
   - Check log level configuration
   - Verify log directory permissions
   - Ensure logger is properly initialized

2. **Missing Information**
   - Use appropriate log level
   - Include all relevant context
   - Add exception information when needed

3. **Performance Issues**
   - Use appropriate log levels
   - Avoid excessive debug logging
   - Use structured logging for complex data 