# Historical Data API

This document describes the standardized historical data functionality available across all brokers in the Quantify framework.

## Overview

The `get_historical_data` method provides a unified interface for retrieving historical OHLC (Open, High, Low, Close) data from any supported broker. This method is implemented in the `BaseBroker` abstract class and available in all broker implementations.

## Method Signature

```python
async def get_historical_data(
    self, 
    symbol: str, 
    timeframe: str, 
    start: Optional[datetime] = None, 
    end: Optional[datetime] = None,
    limit: Optional[int] = None
) -> List[Dict[str, Any]]
```

## Parameters

- **symbol** (str): Trading symbol (e.g., "AAPL", "BTCUSDT", "EURUSD")
- **timeframe** (str): Timeframe for the data (e.g., "1m", "5m", "15m", "30m", "1h", "4h", "1d")
- **start** (Optional[datetime]): Start datetime for the data range. If not provided, defaults to `end - limit periods`
- **end** (Optional[datetime]): End datetime for the data range. If not provided, defaults to current time
- **limit** (Optional[int]): Maximum number of candles to return. If not provided, uses broker-specific default

## Return Value

Returns a list of dictionaries, where each dictionary represents a single OHLC candle with the following structure:

```python
{
    'timestamp': datetime,      # Candle timestamp
    'open': Decimal,           # Opening price
    'high': Decimal,           # Highest price
    'low': Decimal,            # Lowest price
    'close': Decimal,          # Closing price
    'volume': Decimal          # Trading volume
}
```

## Supported Timeframes

All brokers support the following standard timeframes:

- `1m` - 1 minute
- `5m` - 5 minutes
- `15m` - 15 minutes
- `30m` - 30 minutes
- `1h` - 1 hour
- `4h` - 4 hours
- `1d` - 1 day

## Broker-Specific Defaults

Each broker has its own default limit when none is specified:

- **Saxo**: 1200 candles
- **Interactive Brokers**: 1000 candles
- **Binance**: 1000 candles
- **Kraken**: 1000 candles
- **Mock Broker**: 100 candles
- **Backtest Broker**: Uses available data in cache

## Usage Examples

### Basic Usage

```python
# Get the last 100 candles for AAPL on 1-hour timeframe
historical_data = await broker.get_historical_data(
    symbol="AAPL",
    timeframe="1h",
    limit=100
)
```

### With Date Range

```python
from datetime import datetime, timedelta

# Get data for the last 7 days
end = datetime.now()
start = end - timedelta(days=7)

historical_data = await broker.get_historical_data(
    symbol="BTCUSDT",
    timeframe="4h",
    start=start,
    end=end
)
```

### Strategy Integration

```python
class MyStrategy(BaseStrategy):
    async def on_initialize(self):
        # Get historical data for warm-up
        historical_data = await self.broker.get_historical_data(
            symbol=self.symbols[0],
            timeframe=self.timeframes[0],
            limit=1000
        )
        
        # Process historical data for indicators
        for candle in historical_data:
            # Update your indicators here
            pass
```

## Error Handling

The method handles various error conditions gracefully:

- **Connection errors**: Returns empty list if broker is not connected
- **Invalid symbols**: Returns empty list for unsupported symbols
- **API errors**: Logs error and returns empty list
- **Invalid timeframes**: Raises ValueError for unsupported timeframes

## Performance Considerations

- **Rate limiting**: Each broker implements its own rate limiting
- **Data size**: Large requests may be split into smaller chunks
- **Caching**: Some brokers may cache recent data for faster retrieval
- **Network**: Consider network latency for real-time applications

## Broker-Specific Notes

### Saxo Broker
- Requires instrument details to be preloaded
- Supports both "From" and "UpTo" modes for date ranges
- Maximum limit is 1200 candles per request

### Interactive Brokers
- Converts timeframes to IB-specific format
- Uses regular trading hours by default
- Supports "TRADES" data type only

### Binance Broker
- Converts symbols to Binance format (e.g., "BTCUSDT")
- Supports both spot and futures data
- Uses millisecond timestamps

### Kraken Broker
- Converts symbols using internal mapping
- Supports "since" parameter for start time
- Returns data in Kraken's specific format

### Mock Broker
- Generates realistic mock data for testing
- Useful for strategy development and testing
- Configurable data patterns

## Testing

You can test the historical data functionality using the provided example script:

```bash
python examples/historical_data_example.py
```

This script demonstrates usage with different brokers and shows how to process the returned data.

## Integration with Event System

Historical data is separate from the real-time event system. To integrate historical data with live trading:

1. Use historical data for strategy initialization and indicator warm-up
2. Subscribe to real-time market data for live trading
3. Combine both data sources as needed for your strategy logic

## Best Practices

1. **Always specify limits**: Avoid requesting unlimited data
2. **Use appropriate timeframes**: Match your strategy's requirements
3. **Handle empty results**: Always check if data is returned
4. **Error handling**: Implement proper error handling for production use
5. **Caching**: Consider caching frequently requested data
6. **Rate limiting**: Respect broker-specific rate limits 