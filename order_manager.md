# OrderManager Documentation

## Overview

The `OrderManager` is the central component for order tracking, logging, alerting, risk management, and analytics in the Quantify trading system. It provides a unified, extensible API for all order types (options, futures, equities, etc.) and supports advanced features for robust, production-grade trading.

---

## Features

- **Centralized Order Registry:** Tracks all orders (open, closed, failed, etc.) with full state and history.
- **Order Logging & Alerting:** Integrates with `OrderLogger` for file logging and Telegram alerts on all key order events.
- **Callbacks/Event Hooks:** Register custom callbacks for order events (placed, filled, cancelled, modified, failed).
- **Risk/Compliance Checks:** Register risk/compliance functions to block or flag orders before placement.
- **Audit Trail:** Every order maintains a `history` of status changes, event types, timestamps, and reasons.
- **Advanced Filtering & Querying:** Query orders by strategy, status, symbol, time, or custom filter functions.
- **Persistence:** Save/load all orders (with history) to/from a JSON file for recovery after restart.
- **Batch Operations:** Place, update, or cancel multiple orders at once.

---

## API Reference

### Order Placement
```python
OrderManager.add_order(order)
OrderManager.add_orders([order1, order2, ...])
```

### Order Status Updates
```python
OrderManager.update_order(order_id, status, event_data, event_type)
OrderManager.update_orders([
    (order_id1, status1, event_data1, event_type1),
    (order_id2, status2, event_data2, event_type2),
])
```

### Callbacks/Event Hooks
```python
def my_callback(order, event_type, event_data):
    print(f"Order {order.order_id} event: {event_type}")

OrderManager.register_callback('filled', my_callback)
```
Supported event types: `'placed'`, `'filled'`, `'cancelled'`, `'modified'`, `'failed'`

### Risk/Compliance Checks
```python
def my_risk_check(order):
    if order.quantity > 100:
        return False, "Order size exceeds limit"
    return True, None

OrderManager.register_risk_check(my_risk_check)
```
All registered checks must pass for an order to be placed.

### Audit Trail
- Every order has a `.history` list of dicts:
  ```python
  for event in order.history:
      print(event['timestamp'], event['status'], event['event_type'], event['reason'])
  ```

### Filtering & Querying
```python
OrderManager.get_orders_by_strategy('MyStrategy')
OrderManager.get_orders_by_status(OrderStatus.PLACED)
OrderManager.get_orders_by_symbol('AAPL:xnas')
OrderManager.get_orders_by_time(start_time, end_time)
OrderManager.get_orders(lambda o: o.price > 100)
```

### Persistence
```python
OrderManager.save_orders_to_file('orders.json')
OrderManager.load_orders_from_file('orders.json')
```

### Batch Operations
```python
OrderManager.add_orders([order1, order2, ...])
OrderManager.update_orders([
    (order_id1, status1, event_data1, event_type1),
    (order_id2, status2, event_data2, event_type2),
])
```

---

## Extension Points
- **Custom Logging/Alerting:** Swap out or extend `OrderLogger` for new channels or formats.
- **Risk/Compliance:** Integrate with external microservices or add complex logic in risk checks.
- **Dashboard Integration:** Use callbacks or persistence to feed real-time dashboards or analytics.
- **Async/Event Bus:** Extend callback system for async/event-driven architectures.

---

## Best Practices
- Register all risk/compliance checks at startup.
- Use callbacks for real-time strategy/risk/UI reactions.
- Persist orders regularly or on shutdown for recovery.
- Use filtering/querying for analytics, reporting, and troubleshooting.

---

## Example: Full Order Lifecycle
```python
# Place an order
OrderManager.add_order(order)

# Order is placed, logged, and alert sent
# If filled, cancelled, or failed, logs and alerts are sent automatically

# Query all filled orders for a strategy
filled_orders = [o for o in OrderManager.get_orders_by_strategy('MyStrategy') if o.status == OrderStatus.FILL]

# Save all orders to disk
OrderManager.save_orders_to_file('orders.json')
``` 