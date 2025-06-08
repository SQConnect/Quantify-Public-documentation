# Trading Framework Scheduler Manual

## Overview

The Trading Framework Scheduler is a robust scheduling system designed specifically for trading algorithms. It provides market-aware scheduling capabilities, multiple timezone support, conditional execution, event prioritization, and thread-safe execution.

## Key Features

- Market hours aware scheduling
- Multiple timezone support
- Conditional execution
- Event prioritization
- Thread-safe execution
- Integration with trading calendars
- Event history tracking
- Flexible scheduling patterns

## Installation and Setup

The scheduler is automatically initialized and managed by the `BaseStrategy` class. You don't need to manually create, start, or stop the scheduler in your strategies. The scheduler is available as `self.scheduler` in any strategy that inherits from `BaseStrategy`.

```python
from src.strategy_framework.base_strategy import BaseStrategy

class MyStrategy(BaseStrategy):
    def __init__(self, name: str, config: Dict[str, Any]):
        super().__init__(name, config)
        # self.scheduler is now available and ready to use
```

## Basic Usage

### Accessing the Scheduler

In your strategy class, you can access the scheduler directly:

```python
class MyStrategy(BaseStrategy):
    async def initialize(self) -> bool:
        # Schedule tasks using self.scheduler
        self.scheduler.schedule().at("09:30").do(self.my_task)
        return True
```

### Scheduling Tasks

```python
# Schedule at specific time
self.scheduler.schedule().at("09:30").do(self.task)

# Schedule with specific timezone
self.scheduler.schedule().at("09:30", timezone="Europe/London").do(self.task)

# Schedule with custom time object
from datetime import time
self.scheduler.schedule().at(time(9, 30)).do(self.task)
```

### Starting and Stopping

```python
# Start the scheduler
self.scheduler.start()

# Stop the scheduler
self.scheduler.stop()
```

## Scheduling Patterns

### Time-Based Scheduling

```python
# Schedule at specific time
self.scheduler.schedule().at("09:30").do(self.task)

# Schedule with specific timezone
self.scheduler.schedule().at("09:30", timezone="Europe/London").do(self.task)

# Schedule with custom time object
from datetime import time
self.scheduler.schedule().at(time(9, 30)).do(self.task)
```

### Recurring Events

```python
from datetime import timedelta

# Every 5 minutes
self.scheduler.schedule().every(timedelta(minutes=5)).do(self.task)

# Every hour
self.scheduler.schedule().every(timedelta(hours=1)).do(self.task)

# Every day at specific time
self.scheduler.schedule().every(timedelta(days=1)).at("09:30").do(self.task)
```

### Market-Based Scheduling

```python
# At market open
self.scheduler.schedule().at_market_open("US").do(self.task)

# Before market open
self.scheduler.schedule().before_market_open("US", timedelta(minutes=5)).do(self.task)

# At market close
self.scheduler.schedule().at_market_close("US").do(self.task)

# After market close
self.scheduler.schedule().after_market_close("US", timedelta(minutes=15)).do(self.task)
```

### Conditional Execution

```python
# Only execute during market hours
self.scheduler.schedule().every(timedelta(minutes=5))\
    .when(lambda: self.calendar.is_market_open())\
    .do(self.task)

# Only execute on specific days
self.scheduler.schedule().every(timedelta(days=1))\
    .when(lambda: datetime.now().weekday() < 5)\
    .do(self.task)
```

### One-Time Events

```python
from datetime import datetime

# Schedule one-time event
self.scheduler.schedule().once_at(datetime(2024, 1, 1, 9, 30)).do(self.task)
```

## Advanced Features

### Event Priority

```python
from src.data.scheduler import Priority

# High priority event
self.scheduler.schedule().at("09:30")\
    .with_priority(Priority.HIGH)\
    .do(self.critical_task)

# Normal priority event
self.scheduler.schedule().at("09:30")\
    .with_priority(Priority.NORMAL)\
    .do(self.normal_task)
```

### Maximum Executions

```python
# Execute only once
self.scheduler.schedule().every(timedelta(minutes=5))\
    .max_executions(1)\
    .do(self.task)

# Execute 10 times
self.scheduler.schedule().every(timedelta(minutes=5))\
    .max_executions(10)\
    .do(self.task)
```

### Event Metadata

```python
# Add metadata to event
self.scheduler.schedule().at("09:30")\
    .with_metadata(
        description="Daily market analysis",
        category="analysis",
        importance="high"
    )\
    .do(self.task)
```

## Real-World Examples

### Daily Pre-Market Preparation

```python
# Daily pre-market task at 9:25 AM on weekdays
self.scheduler.schedule()\
    .at("09:25")\
    .on_weekdays(0,1,2,3,4)\
    .do(self.premarket_prep)
```

### Position Management

```python
# Check positions 15 minutes before market close
self.scheduler.schedule()\
    .before_market_close("US", timedelta(minutes=15))\
    .do(self.position_check)

# Monitor positions every 5 minutes during market hours
self.scheduler.schedule()\
    .every(timedelta(minutes=5))\
    .when(lambda: self.calendar.is_market_open())\
    .do(self.monitor_positions)
```

### End of Month Reporting

```python
# Generate monthly report at 4:30 PM, execute only once
self.scheduler.schedule()\
    .at("16:30")\
    .do(self.monthly_report)\
    .max_executions(1)
```

### Market Open/Close Tasks

```python
# Prepare for market open
self.scheduler.schedule()\
    .before_market_open("US", timedelta(minutes=5))\
    .do(self.prepare_for_market_open)

# Clean up at market close
self.scheduler.schedule()\
    .at_market_close("US")\
    .do(self.market_close_cleanup)
```

## Best Practices

1. **Scheduler Management**
   - Don't manually start/stop the scheduler - it's managed by BaseStrategy
   - Don't create new scheduler instances - use self.scheduler
   - Schedule tasks in the initialize() method of your strategy

2. **Error Handling**
   - Always wrap scheduled tasks in try-except blocks
   - Log errors appropriately
   - Consider implementing retry logic for critical tasks

3. **Resource Management**
   - Stop the scheduler when shutting down the application
   - Clean up resources in task handlers
   - Monitor memory usage for long-running tasks

4. **Performance**
   - Use appropriate priority levels for tasks
   - Avoid long-running tasks in the main thread
   - Consider using async/await for I/O-bound tasks

5. **Testing**
   - Test scheduled tasks in isolation
   - Use mock calendars for testing
   - Verify timezone handling

## API Reference

### ScheduleBuilder Methods

- `at(time: Union[str, time])` - Schedule at specific time
- `every(interval: timedelta)` - Set recurring interval
- `on_weekdays(*weekdays: int)` - Set days of week (0=Monday, 6=Sunday)
- `before_market_open(market_code: str, offset: timedelta)` - Schedule before market open
- `after_market_close(market_code: str, offset: timedelta)` - Schedule after market close
- `at_market_open(market_code: str)` - Schedule at market open
- `at_market_close(market_code: str)` - Schedule at market close
- `once_at(when: datetime)` - Schedule one-time event
- `with_priority(priority: Priority)` - Set event priority
- `max_executions(count: int)` - Set maximum number of executions
- `when(condition: Callable[[], bool])` - Set execution condition
- `with_metadata(**metadata)` - Add metadata to event
- `do(callback: Callable, name: str)` - Schedule the event

### TradingScheduler Methods

- `start()` - Start the scheduler
- `stop()` - Stop the scheduler
- `add_event(event: ScheduledEvent)` - Add event to scheduler
- `remove_event(event_id: str)` - Remove event from scheduler
- `enable_event(event_id: str)` - Enable event
- `disable_event(event_id: str)` - Disable event
- `get_events(enabled_only: bool)` - Get all events
- `get_event(event_id: str)` - Get event by ID
- `get_next_executions(count: int)` - Get next scheduled executions
- `get_statistics()` - Get scheduler statistics
- `get_execution_history(count: int)` - Get execution history

## Troubleshooting

### Common Issues

1. **Events Not Executing**
   - Check if scheduler is running
   - Verify timezone settings
   - Check market calendar status
   - Verify event conditions

2. **Timezone Issues**
   - Ensure correct timezone is set
   - Check daylight savings time handling
   - Verify market hours in correct timezone

3. **Performance Issues**
   - Monitor event queue size
   - Check task execution times
   - Review event priorities
   - Monitor system resources

### Debugging

```python
# Get scheduler statistics
stats = self.scheduler.get_statistics()
print(f"Total events: {stats['total_events']}")
print(f"Enabled events: {stats['enabled_events']}")
print(f"Next execution: {stats['next_execution']}")

# Get execution history
history = self.scheduler.get_execution_history(count=10)
for entry in history:
    print(f"Event: {entry['name']}, Duration: {entry['duration']}s")
```

## Contributing

The scheduler is part of the trading framework. To contribute:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests
5. Submit a pull request

## License

This scheduler is part of the trading framework and is subject to its license terms.

## Schedule Types

- **ONCE**: Schedule a one-time event.
- **RECURRING**: Schedule a recurring event at specified intervals.
- **MARKET_OPEN**: Schedule an event at market open.
- **MARKET_CLOSE**: Schedule an event at market close.
- **BEFORE_MARKET_OPEN**: Schedule an event before market open.
- **AFTER_MARKET_CLOSE**: Schedule an event after market close.
- **BEFORE_MARKET_CLOSE**: Schedule an event before market close.
- **INTRADAY**: Schedule an event during market hours.
- **END_OF_DAY**: Schedule an event at the end of the trading day.
- **END_OF_WEEK**: Schedule an event at the end of the trading week.
- **END_OF_MONTH**: Schedule an event at the end of the trading month. 