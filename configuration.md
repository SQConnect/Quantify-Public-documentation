# Configuration Management

## Overview
The ConfigManager is a centralized configuration management system for the Quantify trading platform. It handles all configuration settings, including broker credentials, strategy parameters, and system settings.

## Basic Usage

### Initialization
```python
from src.core.config import ConfigManager

# Create a new ConfigManager instance
config_manager = ConfigManager()
```

### Accessing Configuration
```python
# Get a specific configuration value using dot notation
api_key = config_manager.get('broker.kraken.api_key')

# Get broker configuration
kraken_config = config_manager.get_broker_config('kraken')

# Get strategy configuration
strategy_config = config_manager.get_strategy_config('pma_candle')

# Get the entire configuration
full_config = config_manager.config
```

## Environment Variables

The ConfigManager automatically loads API credentials from environment variables. Create a `.env` file in your project root with the following structure:

```env
# Kraken API
KRAKEN_API_KEY=your_kraken_api_key
KRAKEN_API_SECRET=your_kraken_api_secret

# Saxo API
SAXO_CLIENT_ID=your_saxo_client_id
SAXO_CLIENT_SECRET=your_saxo_client_secret

# Binance API
BINANCE_API_KEY=your_binance_api_key
BINANCE_API_SECRET=your_binance_api_secret
```

## Configuration Structure

The configuration is organized into several sections:

### Broker Configuration
```python
broker_config = {
    'kraken': {
        'api_key': 'your_api_key',
        'api_secret': 'your_api_secret',
        'testnet': True
    },
    'saxo': {
        'client_id': 'your_client_id',
        'client_secret': 'your_client_secret'
    },
    'binance': {
        'api_key': 'your_api_key',
        'api_secret': 'your_api_secret'
    }
}
```

### Strategy Configuration
```python
strategy_config = {
    'pma_candle': {
        'parameters': {
            'position_size': 0.1,
            'required_rising_candles': 4,
            'hold_candles': 3
        }
    }
}
```

### System Configuration
```python
system_config = {
    'logging': {
        'level': 'INFO',
        'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    },
    'database': {
        'type': 'sqlite',
        'path': 'trading.db'
    }
}
```

## Available Methods

### Getting Configuration
- `get(key: str, default: Any = None) -> Any`: Get a specific configuration value
- `get_broker_config(broker: str) -> Dict[str, Any]`: Get broker configuration
- `get_strategy_config(strategy: str) -> Dict[str, Any]`: Get strategy configuration
- `get_backtest_config() -> Dict[str, Any]`: Get backtesting configuration
- `get_logging_config() -> Dict[str, Any]`: Get logging configuration
- `get_database_config() -> Dict[str, Any]`: Get database configuration
- `get_cache_config() -> Dict[str, Any]`: Get cache configuration
- `get_risk_config() -> Dict[str, Any]`: Get risk management configuration
- `get_websocket_config() -> Dict[str, Any]`: Get WebSocket configuration
- `get_rate_limit_config() -> Dict[str, Any]`: Get rate limiting configuration

### Modifying Configuration
- `set(key: str, value: Any) -> None`: Set a configuration value
- `update(config: Dict[str, Any]) -> None`: Update multiple configuration values
- `reset() -> None`: Reset configuration to default values

## Example Usage in Strategy

```python
from src.core.config import ConfigManager

class MyStrategy:
    def __init__(self):
        self.config_manager = ConfigManager()
        
        # Get broker configuration
        self.broker_config = self.config_manager.get_broker_config('kraken')
        
        # Get strategy parameters
        self.strategy_config = self.config_manager.get_strategy_config('my_strategy')
        
        # Get specific parameter
        self.position_size = self.config_manager.get('strategy.my_strategy.parameters.position_size')
```

## Best Practices

1. **Environment Variables**: Always use environment variables for sensitive information like API keys
2. **Configuration Updates**: Use the `update` method for bulk configuration changes
3. **Default Values**: Provide default values when using `get` to handle missing configuration
4. **Error Handling**: Use try-except blocks when accessing configuration that might not exist
5. **Configuration Validation**: Validate configuration values before using them

## Error Handling

The ConfigManager raises `ConfigurationError` in the following cases:
- Unknown broker type
- Unknown strategy type
- Invalid configuration key
- Failed to set configuration value

Example error handling:
```python
from src.core.exceptions import ConfigurationError

try:
    broker_config = config_manager.get_broker_config('unknown_broker')
except ConfigurationError as e:
    logger.error(f"Configuration error: {e}")
    # Handle error appropriately
```

## Configuration File Structure

The configuration is defined in YAML files, primarily `config/main.yaml`. The configuration hierarchy is:

1. **Base configuration**: Loaded from `config/main.yaml`
2. **Environment variables**: Override YAML settings using `${ENV_VAR}` syntax
3. **Runtime overrides**: Using the `set` or `update` methods

### YAML Configuration Format

The main configuration file uses YAML format with sections for different components:

```yaml
# config/main.yaml
broker:
  Saxo:
    client_id: "${SAXO_API_KEY}"
    client_secret: "${SAXO_API_SECRET}"
    base_url: "https://gateway.saxobank.com/sim/openapi"

trading:
  default_timeframe: "1h"
  default_commission: 0.001
  max_position_size: 0.1

strategy:
  ma_crossover:
    fast_period: 10
    slow_period: 20
    risk_per_trade: 0.02
```

### Environment Variable Resolution

Environment variables are automatically resolved using the `${ENV_VAR}` syntax in YAML files.

## Security Considerations

1. Never commit API keys or secrets to version control
2. Use environment variables for sensitive information
3. Validate configuration values before use
4. Use testnet for development and testing
5. Implement proper access control for configuration changes

## Troubleshooting

Common issues and solutions:

1. **Missing API Keys**
   - Check if environment variables are set
   - Verify .env file exists and is properly formatted
   - Ensure environment variables are loaded

2. **Invalid Configuration**
   - Check configuration structure
   - Verify all required fields are present
   - Use default values for optional fields

3. **Configuration Not Updated**
   - Call `reset()` to reload configuration
   - Check if environment variables are loaded
   - Verify configuration updates are applied 