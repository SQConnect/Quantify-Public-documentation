# Assignment Notifications Guide

This guide explains how to implement and use the assignment notification system across all supported brokers (Saxo, Binance, Kraken) to monitor for **actual assignments** when positions are being assigned to your account.

## Overview

The assignment notification system is designed to detect when you are **actually being assigned** positions, not just monitor the risk of assignment. This is critical for:

- **Options**: When your short options are being exercised and you're assigned the underlying shares
- **Futures**: When your futures contracts are being delivered and you're assigned the underlying commodity/asset

## Key Concepts

### Assignment Risk vs. Assignment Notification

- **Assignment Risk**: Monitoring positions that might be assigned (e.g., in-the-money options near expiry)
- **Assignment Notification**: Monitoring when positions are actually being assigned to your account

### What Triggers Assignment Notifications

1. **New Position Creation**: When a new position appears in your account that you didn't explicitly trade
2. **Position Size Increase**: When an existing position size increases without you placing a new order
3. **WebSocket Events**: Real-time notifications from broker APIs about assignment events

## Implementation Across Brokers

### Saxo Bank

Saxo provides assignment notifications through:

```python
# Subscribe to assignment notifications
await broker.subscribe_to_assignment_notifications(callback)

# Start monitoring for assignments
await broker.monitor_assignment_notifications(callback)
```

**How it works:**
- Subscribes to position updates via `/port/v1/positions/subscriptions`
- Monitors position changes every 5 minutes
- Detects new positions or position size increases
- Publishes assignment events when detected

### Binance

Binance provides assignment notifications through:

```python
# Subscribe to assignment notifications
await broker.subscribe_to_assignment_notifications(callback)

# Start monitoring for assignments
await broker.monitor_assignment_notifications(callback)
```

**How it works:**
- Uses WebSocket user data stream for real-time updates
- Monitors position changes every 5 minutes
- Detects futures delivery assignments
- Publishes assignment events when detected

### Kraken

Kraken provides assignment notifications through:

```python
# Subscribe to assignment notifications
await broker.subscribe_to_assignment_notifications(callback)

# Start monitoring for assignments
await broker.monitor_assignment_notifications(callback)
```

**How it works:**
- Uses private WebSocket feed for real-time updates
- Monitors position changes every 5 minutes
- Detects futures delivery assignments
- Publishes assignment events when detected

## Usage Example

```python
from src.broker_interface.saxo_broker import SaxoBroker
from src.broker_interface.binance_broker import BinanceBroker
from src.broker_interface.kraken_clean_broker import KrakenCleanBroker

class AssignmentMonitor:
    def __init__(self):
        self.brokers = {}
        
    async def setup_brokers(self):
        # Setup Saxo
        saxo_config = {
            'api_key': 'your_saxo_api_key',
            'api_secret': 'your_saxo_api_secret',
            'base_url': 'https://gateway.saxobank.com/sim/openapi'
        }
        self.brokers['saxo'] = SaxoBroker(event_manager, candle_manager, order_logger, saxo_config)
        await self.brokers['saxo'].connect()
        
        # Setup Binance
        binance_config = {
            'api_key': 'your_binance_api_key',
            'api_secret': 'your_binance_api_secret',
            'testnet': True
        }
        self.brokers['binance'] = BinanceBroker(event_manager, candle_manager, order_logger, binance_config)
        await self.brokers['binance'].connect()
        
        # Setup Kraken
        kraken_config = {
            'api_key': 'your_kraken_api_key',
            'api_secret': 'your_kraken_api_secret',
            'ws_url': 'wss://ws.kraken.com'
        }
        self.brokers['kraken'] = KrakenCleanBroker(event_manager, candle_manager, order_logger, kraken_config)
        await self.brokers['kraken'].connect()
        
    async def start_monitoring(self):
        """Start monitoring for actual assignments across all brokers."""
        for broker_name, broker in self.brokers.items():
            # Subscribe to assignment notifications
            await broker.subscribe_to_assignment_notifications(self.handle_assignment)
            
            # Start monitoring
            await broker.monitor_assignment_notifications(self.handle_assignment_monitoring)
            
            print(f"Started assignment monitoring for {broker_name}")
            
    def handle_assignment(self, event_type: str, data: dict):
        """Handle actual assignment notifications."""
        symbol = data.get('symbol', 'Unknown')
        quantity = data.get('quantity', 0)
        asset_type = data.get('asset_type', 'Unknown')
        assignment_type = data.get('assignment_type', 'Unknown')
        
        print(f"🚨 ASSIGNMENT NOTIFICATION!")
        print(f"Symbol: {symbol}")
        print(f"Quantity: {quantity}")
        print(f"Asset Type: {asset_type}")
        print(f"Assignment Type: {assignment_type}")
        
        # Implement immediate response logic
        self.handle_actual_assignment(data)
        
    def handle_assignment_monitoring(self, event_type: str, data: dict):
        """Handle assignment monitoring events."""
        print(f"Assignment monitoring: {event_type} - {data}")
        
    def handle_actual_assignment(self, assignment_data: dict):
        """Handle actual assignment - when you're being assigned a position."""
        symbol = assignment_data.get('symbol', '')
        quantity = assignment_data.get('quantity', 0)
        asset_type = assignment_data.get('asset_type', '')
        
        if asset_type == 'option':
            print(f"OPTION ASSIGNMENT: You are being assigned {quantity} shares of {symbol}")
            # Immediate actions:
            # 1. Hedge the position if you don't want to hold the underlying
            # 2. Close the position if you don't want to hold the underlying
            # 3. Roll to a different expiry
            
        elif asset_type == 'futures':
            print(f"FUTURES ASSIGNMENT: You are being delivered {quantity} contracts of {symbol}")
            # Immediate actions:
            # 1. Close the position if you don't want delivery
            # 2. Roll to the next contract
            # 3. Take delivery if that's your intention

# Usage
async def main():
    monitor = AssignmentMonitor()
    await monitor.setup_brokers()
    await monitor.start_monitoring()
    
    # Keep running to receive notifications
    await asyncio.sleep(3600)  # Run for 1 hour

if __name__ == "__main__":
    asyncio.run(main())
```

## Assignment Detection Logic

The system detects assignments by comparing position snapshots:

```python
def _detect_new_assignments(self, previous_positions: List[Dict], current_positions: List[Dict]) -> List[Dict]:
    """Detect new assignments by comparing previous and current positions."""
    new_assignments = []
    
    # Create lookup dictionaries for easier comparison
    prev_positions = {pos.get('symbol', ''): pos for pos in previous_positions}
    curr_positions = {pos.get('symbol', ''): pos for pos in current_positions}
    
    for symbol, curr_pos in curr_positions.items():
        prev_pos = prev_positions.get(symbol)
        
        if not prev_pos:
            # New position - could be an assignment
            if curr_pos.get('quantity', 0) != 0:
                assignment_data = {
                    'symbol': symbol,
                    'quantity': curr_pos.get('quantity', 0),
                    'asset_type': curr_pos.get('asset_type', ''),
                    'assignment_type': 'new_position',
                    'timestamp': datetime.now().isoformat(),
                    'position_data': curr_pos
                }
                new_assignments.append(assignment_data)
        else:
            # Existing position - check for quantity changes
            prev_qty = prev_pos.get('quantity', 0)
            curr_qty = curr_pos.get('quantity', 0)
            
            if curr_qty > prev_qty and curr_qty > 0:
                # Position size increased - could be assignment
                assignment_data = {
                    'symbol': symbol,
                    'quantity': curr_qty - prev_qty,  # The assigned quantity
                    'asset_type': curr_pos.get('asset_type', ''),
                    'assignment_type': 'quantity_increase',
                    'timestamp': datetime.now().isoformat(),
                    'position_data': curr_pos,
                    'previous_quantity': prev_qty
                }
                new_assignments.append(assignment_data)
    
    return new_assignments
```

## Event Types

The system publishes the following events:

- `EventType.OPTION_ASSIGNMENT`: When options are being assigned
- `EventType.FUTURES_ASSIGNMENT`: When futures are being delivered

## Response Strategies

### For Option Assignments

1. **Immediate Hedge**: If you don't want to hold the underlying shares
2. **Close Position**: Sell the assigned shares immediately
3. **Roll Position**: Roll to a different expiry or strike
4. **Hold Position**: If you want to hold the underlying shares

### For Futures Assignments

1. **Close Position**: If you don't want delivery
2. **Roll Position**: Roll to the next contract month
3. **Take Delivery**: If you want the underlying commodity/asset

## Best Practices

1. **Monitor Continuously**: Always have assignment monitoring active
2. **Set Up Alerts**: Configure immediate notifications for assignments
3. **Have Response Plans**: Pre-plan your response to different assignment scenarios
4. **Test Regularly**: Test the system with small positions first
5. **Document Procedures**: Document your assignment response procedures

## Troubleshooting

### Common Issues

1. **False Positives**: Some position changes might not be assignments
   - Solution: Filter by assignment type and verify with broker
   
2. **Missed Assignments**: Assignments might be missed if monitoring is down
   - Solution: Use broker's official assignment notifications as backup
   
3. **Delayed Notifications**: Network issues might delay notifications
   - Solution: Implement retry logic and fallback monitoring

### Debugging

```python
# Enable debug logging
import logging
logging.basicConfig(level=logging.DEBUG)

# Check assignment monitoring status
for broker_name, broker in self.brokers.items():
    print(f"{broker_name} assignment monitoring active: {hasattr(broker, '_assignment_monitoring_task')}")
```

## Integration with Risk Management

The assignment notification system integrates with the broader risk management framework:

```python
from src.risk_management.risk_service import RiskService

# Register assignment callbacks with risk service
risk_service.register_assignment_callback(handle_assignment)

# Risk service will automatically:
# 1. Update position tracking
# 2. Recalculate risk metrics
# 3. Trigger risk alerts
# 4. Update margin requirements
```

This ensures that assignments are properly tracked and managed within your overall risk management system. 