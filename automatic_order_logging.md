# Automatic Order Logging System

## Overview
The Automatic Order Logging system provides a centralized, consistent way to log all order-related events across different broker implementations. This system ensures that all trading activities are properly recorded for auditing, monitoring, and analysis purposes.

## Architecture

### OrderLogger Class
The `OrderLogger` class is the core component that handles all order-related logging. It is instantiated by the `BrokerFactory` and injected into each broker instance.

```python
class OrderLogger:
    def __init__(self):
        self.logger = get_logger("OrderLogger", "trading_orders")
        self.logger.setLevel(logging.INFO)
        
        # Create a separate file handler for orders
        order_handler = logging.FileHandler('logs/trading_orders.log')
        order_handler.setFormatter(logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        ))
        self.logger.addHandler(order_handler)
```

### Logged Events
The system logs the following types of events:

1. **Order Entry**
   ```python
   log_order_entry(broker: str, symbol: str, side: str, quantity: float, 
                  price: float, order_id: str)
   ```

2. **Order Exit**
   ```python
   log_order_exit(broker: str, symbol: str, side: str, quantity: float,
                 price: float, order_id: str, pnl: float = None)
   ```

3. **Stop Loss**
   ```python
   log_stop_loss(broker: str, symbol: str, side: str, quantity: float,
                price: float, order_id: str, pnl: float = None)
   ```

4. **Take Profit**
   ```python
   log_take_profit(broker: str, symbol: str, side: str, quantity: float,
                  price: float, order_id: str, pnl: float = None)
   ```

5. **Order Cancel**
   ```python
   log_order_cancel(broker: str, symbol: str, order_id: str)
   ```

6. **Order Modify**
   ```python
   log_order_modify(broker: str, symbol: str, order_id: str, 
                   new_price: float = None, new_quantity: float = None)
   ```

7. **Order Failed**
   ```python
   log_order_failed(broker: str, symbol: str, side: str, quantity: float,
                    price: float, reason: str)
   ```

## Implementation

### Broker Factory Integration
The `BrokerFactory` creates and manages the `OrderLogger` instance:

```python
class BrokerFactory:
    def __init__(self):
        self.order_logger = OrderLogger()
        self.brokers: Dict[str, BaseBroker] = {}
        self.logger = get_logger(__name__, "BrokerFactory")

    def create_broker(self, broker_type: str, config: Dict[str, Any]) -> BaseBroker:
        """Create a new broker instance
        
        Args:
            broker_type: Type of broker to create (e.g., 'kraken', 'binance')
            config: Configuration dictionary containing broker-specific settings
                   Required keys:
                   - api_key: API key for authentication
                   - api_secret: API secret for authentication
                   - base_url: Base URL for API endpoints
                   Optional keys:
                   - testnet: Whether to use testnet (default: True)
                   - timeout: Request timeout in seconds (default: 30)
                   - max_retries: Maximum number of retry attempts (default: 3)
        
        Returns:
            BaseBroker: Configured broker instance
            
        Raises:
            ValueError: If broker_type is unknown or config is invalid
            RuntimeError: If broker creation fails
        """
        try:
            if not config:
                raise ValueError("Config dictionary is required")
                
            if broker_type in self.brokers:
                self.logger.info(f"Returning existing broker instance for {broker_type}")
                return self.brokers[broker_type]

            self.logger.info(f"Creating new broker instance for {broker_type}")
            
            if broker_type == "kraken":
                broker = KrakenBroker(config)
            elif broker_type == "binance":
                broker = BinanceBroker(config)
            elif broker_type == "saxo":
                broker = SaxoBroker(config)
            elif broker_type == "mock":
                broker = MockBroker(config)
            else:
                raise ValueError(f"Unknown broker type: {broker_type}")

            # Inject the order logger
            broker.set_order_logger(self.order_logger)
            
            # Store the broker instance
            self.brokers[broker_type] = broker
            
            return broker
            
        except Exception as e:
            self.logger.error(f"Error creating broker {broker_type}: {e}")
            raise RuntimeError(f"Failed to create broker: {str(e)}")
```

### Base Broker Interface
The `BaseBroker` class provides the interface for order logging:

```python
class BaseBroker(ABC):
    def __init__(self, config: Dict[str, Any]):
        """Initialize base broker
        
        Args:
            config: Configuration dictionary containing broker-specific settings
        """
        self.config = config
        self.order_logger = None
        self._connected = False

    def set_order_logger(self, logger):
        """Set the order logger instance"""
        self.order_logger = logger
```

### Broker Implementations
Each broker implementation (Kraken, Binance, Saxo, Mock) uses the order logger in their order-related methods:

```python
async def place_order(self, symbol: str, side: str, quantity: Decimal,
                     price: Optional[Decimal] = None,
                     stop_loss: Optional[Decimal] = None,
                     take_profit: Optional[Decimal] = None) -> Dict[str, Any]:
    # ... order placement logic ...
    if self.order_logger:
        self.order_logger.log_order_entry(
            "BrokerName",
            symbol,
            side,
            float(quantity),
            float(price) if price else None,
            order_id
        )
```

## Usage

### Creating a Broker with Order Logging
```python
from src.broker_interface.broker_factory import BrokerFactory
from decimal import Decimal

# Create broker factory
factory = BrokerFactory()

# Create broker configuration
config = {
    'api_key': 'your_api_key',
    'api_secret': 'your_api_secret',
    'base_url': 'https://api.kraken.com',
    'testnet': True,
    'timeout': 30,
    'max_retries': 3
}

# Create a broker (order logger is automatically injected)
kraken_broker = factory.create_broker("kraken", config)

# Place an order (automatically logged)
await kraken_broker.place_order("BTC/USD", "buy", Decimal("0.1"), Decimal("50000"))
```

### Error Handling
The system includes comprehensive error handling:

1. **Configuration Validation**
   ```python
   if not config:
       raise ValueError("Config dictionary is required")
   ```

2. **Broker Type Validation**
   ```python
   if broker_type not in ["kraken", "binance", "saxo", "mock"]:
       raise ValueError(f"Unknown broker type: {broker_type}")
   ```

3. **Broker Creation Error Handling**
   ```python
   try:
       broker = factory.create_broker("kraken", config)
   except RuntimeError as e:
       logger.error(f"Failed to create broker: {e}")
   ```

### Viewing Logs
Order logs are written to `logs/trading_orders.log`. You can monitor them in real-time using:
```bash
tail -f logs/trading_orders.log
```

## Telegram Notifications
The system can be configured to send real-time order notifications to a Telegram chat. This allows for immediate updates on trading activity.

### Configuration
To enable Telegram notifications, you need to configure the settings in `config/main.yaml`:

```yaml
telegram:
  enabled: true
  bot_token: "${TELEGRAM_BOT_TOKEN}"
  chat_id: "${TELEGRAM_CHAT_ID}"
```

- `enabled`: Set to `true` to enable notifications, `false` to disable them.
- `bot_token`: Your Telegram bot token. It's recommended to load this from an environment variable.
- `chat_id`: The ID of the Telegram chat where notifications will be sent.

You must also set the following environment variables in your `.env` file:
```
TELEGRAM_BOT_TOKEN="your_telegram_bot_token"
TELEGRAM_CHAT_ID="your_telegram_chat_id"
```

The `OrderLogger` will automatically pick up this configuration and start sending alerts for confirmed and failed orders.

## Benefits
1. **Centralized Logging**: All order-related events are logged in one place
2. **Consistent Format**: Standardized log format across all brokers
3. **Audit Trail**: Complete record of all trading activities
4. **Debugging**: Easy to track and debug order-related issues
5. **Analysis**: Logs can be used for performance analysis and strategy optimization
6. **Order Modify**
   ```python
   log_order_modify(broker: str, symbol: str, order_id: str, 
                   new_price: float = None, new_quantity: float = None)
   ```
7. **Order Failed**
   ```python
   log_order_failed(broker: str, symbol: str, side: str, quantity: float,
                    price: float, reason: str)
   ```

## Best Practices
1. Always check if `order_logger` exists before logging
2. Include all relevant information in log entries
3. Use appropriate log levels (INFO for normal operations)
4. Monitor log file size and implement rotation if needed
5. Consider implementing log aggregation for distributed systems
6. Always provide a valid configuration dictionary when creating brokers
7. Handle broker creation errors appropriately in your application

## Future Enhancements
1. Add log rotation to manage file size
2. Implement log compression for historical data
3. Add log aggregation for distributed systems
4. Create log analysis tools for performance monitoring
5. Add support for structured logging formats (JSON)
6. Add configuration validation and schema checking
7. Implement broker health monitoring and automatic reconnection 