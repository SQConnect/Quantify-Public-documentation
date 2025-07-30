# Risk Management Documentation

This document describes the comprehensive risk management system for margin calls and options/futures assignments in the Quantify framework.

## Overview

The risk management system provides real-time monitoring of:
- **Margin Calls**: Monitor margin utilization and receive alerts when thresholds are breached
- **Assignment Risks**: Track options and futures positions approaching expiry and assess assignment risks
- **Automatic Assignment Detection**: Real-time detection of actual assignments (when positions are assigned to your account)
- **Position Monitoring**: Real-time updates on account balances and positions
- **Risk Alerts**: Configurable alerts and automated responses to risk events

## Architecture

The risk management system is built as a **microservice** within the Quantify service framework:

### 1. Risk Management Service
- **ServiceHost**: Runs the risk management service on port 5557 (configurable)
- **RiskMonitoringHandler**: Implements BaseHandler for risk operations
- **RiskManagementClient**: Extends BaseClient for service communication
- **RiskManager**: Core risk assessment and state management
- **DTOs**: Data Transfer Objects for structured communication

### 2. Automatic Integration
- **PositionManager**: Automatically subscribes to balance and position updates
- **OrderManager**: Automatically initializes risk management integration
- **Broker Integration**: All brokers automatically start assignment detection when connected
- **Framework Integration**: Risk management starts automatically with the framework

### 3. Automatic Assignment Detection

**Key Feature**: The framework automatically detects when you are assigned positions without requiring manual subscription.

#### How It Works:
1. **Broker Connection**: When any broker connects, it automatically starts assignment detection
2. **Position Monitoring**: Brokers continuously monitor position changes every 5 minutes
3. **Assignment Detection**: When new positions appear or existing positions increase unexpectedly, assignments are detected
4. **Event Publishing**: Assignment events are automatically published to the event system
5. **Risk Service Integration**: The risk management service automatically receives and processes assignment events
6. **Strategy Notification**: Strategies automatically receive assignment notifications through the event system

#### Automatic Flow:
```
Broker Connects → Assignment Detection Starts → Position Changes Detected → 
Assignment Events Published → Risk Service Processes → Strategies Notified
```

### 4. Service Framework Integration
The service integrates seamlessly with the existing service framework:
- **ZMQ Communication**: Uses the same transport layer as other services
- **Handler Pattern**: Follows the established BaseHandler pattern
- **Client Pattern**: Uses the established BaseClient pattern
- **Configuration**: Uses the same configuration management approach

## Quick Start

### Automatic Setup (Recommended)

The risk management system is **fully automatic** and requires no manual intervention:

```bash
# Start all services including risk management
./quantify.sh start

# Or start just the risk management service
./quantify.sh start risk_management
```

The system automatically:
1. Starts the risk management service
2. PositionManager subscribes to broker data streams
3. OrderManager connects to risk management service
4. **All brokers automatically start assignment detection when they connect**
5. All risk monitoring happens in the background

### Framework Integration

Within the framework, assignment detection is completely automatic:

```python
# In your strategy - no manual assignment monitoring needed!
class MyStrategy(BaseStrategy):
    async def on_assignment_event(self, event):
        """Automatically called when assignments are detected"""
        symbol = event.symbol
        quantity = event.quantity
        asset_type = event.asset_type
        
        self.logger.critical(f"ASSIGNMENT DETECTED: {symbol} - {quantity} {asset_type}")
        
        # Handle the assignment (close position, hedge, etc.)
        await self.handle_assignment(symbol, quantity, asset_type)
    
    async def handle_assignment(self, symbol, quantity, asset_type):
        """Handle the assignment - implement your logic here"""
        if asset_type == 'option':
            # Handle option assignment
            await self.close_option_position(symbol, quantity)
        elif asset_type == 'futures':
            # Handle futures delivery
            await self.close_futures_position(symbol, quantity)
```

## Service Commands

The RiskMonitoringHandler supports the following commands:

### Core Monitoring Commands
- `start_monitoring`: Start risk monitoring for a ticker/account
- `stop_monitoring`: Stop risk monitoring for a ticker/account
- `get_risk_summary`: Get comprehensive risk summary

### Data Update Commands
- `update_margin_state`: Update margin state with broker data
- `update_assignment_risks`: Update assignment risks with position data
- `get_assignment_risks`: Get current assignment risks
- `get_margin_summary`: Get current margin summary

### Alert Configuration Commands
- `update_risk_alert`: Update risk alert configuration
- `enable_risk_alert`: Enable a risk alert
- `disable_risk_alert`: Disable a risk alert

## Configuration

### Service Configuration

Add the following to your configuration file:

```yaml
risk_management:
  server:
    host: "0.0.0.0"
    port: 5557
    use_ipc: false
    use_binary: false
  client:
    host: "127.0.0.1"
    port: 5557
    timeout: 5000
    use_ipc: false
    use_binary: false
  # Risk limits
  max_margin_utilization: 0.75
  margin_call_threshold: 0.85
  assignment_risk_days: 7
  critical_assignment_days: 1
  max_option_delta_exposure: 0.3
  max_futures_roll_days: 3
  # Alert thresholds
  margin_warning_threshold: 80.0
  margin_critical_threshold: 90.0
  assignment_high_threshold: 3
  assignment_critical_threshold: 1
```

### Risk Limits

```python
from src.risk_management.risk_manager import RiskLimits

risk_limits = RiskLimits(
    max_margin_utilization=0.75,      # Warning at 75%
    margin_call_threshold=0.85,       # Critical at 85%
    assignment_risk_days=7,           # Start monitoring 7 days before expiry
    critical_assignment_days=1,       # Critical alerts 1 day before expiry
    max_option_delta_exposure=0.3,    # Max option delta exposure
    max_futures_roll_days=3           # Max days before futures roll
)
```

## Margin Call Monitoring

### Real-time Monitoring

The service monitors margin utilization in real-time:

```python
# Update margin state from broker
margin_data = {
    'account_id': 'AAPL_ACCOUNT',
    'MarginUtilizationPct': 85.5,
    'MarginAvailable': 25000.0,
    'MarginUsed': 175000.0,
    'InitialMargin': 100000.0,
    'MarginUsedByCurrentPositions': 165000.0
}

result = await client.update_margin_state(margin_data)

# Check for alerts
if result.get('alerts'):
    for alert in result['alerts']:
        print(f"Alert: {alert['message']}")
```

### Alert Levels

- **Warning**: Margin utilization >= 80% (configurable)
- **Critical**: Margin utilization >= 90% (configurable)

## Assignment Risk Monitoring

### Automatic Assignment Detection

**No manual setup required!** Assignment detection is automatic:

1. **Broker Connection**: When brokers connect, they automatically start monitoring positions
2. **Position Changes**: Every 5 minutes, brokers check for new positions or position increases
3. **Assignment Detection**: Unexpected position changes trigger assignment events
4. **Event Processing**: The risk management service automatically processes assignment events
5. **Strategy Notification**: Strategies receive assignment notifications through the event system

### Assignment Event Types

- **`OPTION_ASSIGNMENT`**: When options are assigned
- **`FUTURES_ASSIGNMENT`**: When futures are delivered

### Assignment Event Data

```python
{
    'symbol': 'AAPL240315C150',
    'quantity': 100,
    'asset_type': 'option',
    'assignment_type': 'new_position',  # or 'quantity_increase'
    'timestamp': '2024-03-15T09:30:00',
    'position_data': {...},
    'broker': 'saxo_broker'
}
```

### Strategy Integration

Strategies automatically receive assignment events:

```python
class MyStrategy(BaseStrategy):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        # Subscribe to assignment events (automatic in framework)
        self.event_manager.subscribe(EventType.OPTION_ASSIGNMENT, self.on_assignment)
        self.event_manager.subscribe(EventType.FUTURES_ASSIGNMENT, self.on_assignment)
    
    async def on_assignment(self, event):
        """Called automatically when options or futures are assigned"""
        symbol = event.symbol
        quantity = event.quantity
        asset_type = event.asset_type  # 'option' or 'futures'
        
        if asset_type == 'option':
            self.logger.critical(f"OPTION ASSIGNED: {symbol} - {quantity} shares")
        else:
            self.logger.critical(f"FUTURES DELIVERED: {symbol} - {quantity} contracts")
        
        # Immediate response required
        await self.handle_assignment(symbol, quantity, asset_type)
```

## Response Strategies

### Immediate Actions for Assignments

When assignments are detected, immediate action is typically required:

```python
async def handle_assignment(self, symbol, quantity, asset_type):
    """Handle assignment for both options and futures - immediate action required"""
    
    # 1. Close the assigned position immediately
    await self.broker.place_order(
        symbol=symbol,
        side='sell',
        quantity=quantity,
        order_type='market'
    )
    
    if asset_type == 'option':
        # 2. Hedge the position if needed (for options)
        underlying = self.get_underlying_from_option(symbol)
        if self.should_hedge_assignment(symbol, quantity):
            await self.hedge_assignment(underlying, quantity)
        
        # 3. Log the assignment
        self.logger.critical(f"Handled option assignment: {symbol} - {quantity}")
    else:
        # 2. Roll to next contract if needed (for futures)
        next_contract = self.get_next_futures_contract(symbol)
        if next_contract:
            await self.roll_futures_position(symbol, next_contract, quantity)
        
        # 3. Log the delivery
        self.logger.critical(f"Handled futures delivery: {symbol} - {quantity}")
    
    # 4. Update risk management
    await self.update_risk_state()
```

### Risk Management Integration

The risk management service automatically tracks assignments:

```python
# Get current assignment summary
summary = await client.get_risk_summary()
assignments = summary.get('recent_assignments', [])

for assignment in assignments:
    print(f"Recent assignment: {assignment['symbol']} - {assignment['quantity']}")
```

## Troubleshooting

### Assignment Detection Issues

If assignments are not being detected:

1. **Check Broker Connection**: Ensure brokers are connected and healthy
2. **Check Event System**: Verify assignment events are being published
3. **Check Risk Service**: Ensure risk management service is running
4. **Check Logs**: Look for assignment detection logs in broker logs

### Common Issues

- **No Assignment Events**: Check if brokers are properly connected
- **Missing Position Data**: Verify broker position APIs are working
- **Event Not Received**: Check event manager subscriptions
- **Service Not Responding**: Restart risk management service

## Best Practices

### Assignment Management

1. **Immediate Response**: Always respond immediately to assignment events
2. **Position Closure**: Close assigned positions as soon as possible
3. **Hedging**: Consider hedging strategies for large assignments
4. **Monitoring**: Use the risk management service to track assignment patterns
5. **Documentation**: Log all assignment handling for audit purposes

### Risk Management

1. **Regular Monitoring**: Check risk summaries regularly
2. **Alert Configuration**: Configure appropriate alert thresholds
3. **Response Plans**: Have predefined response plans for different risk scenarios
4. **Testing**: Test assignment handling in paper trading environments
5. **Backup Plans**: Have backup strategies for assignment handling

## Integration with Framework

The assignment detection is fully integrated into the Quantify framework:

- **Automatic Startup**: Starts automatically with the framework
- **Broker Integration**: Works with all supported brokers (Saxo, Binance, Kraken)
- **Event System**: Uses the framework's event system for notifications
- **Risk Service**: Integrated with the risk management service
- **Strategy Framework**: Works seamlessly with all strategy types 