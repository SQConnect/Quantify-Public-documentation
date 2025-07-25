# OCO & Trailing Stop Orders Manual

## Overview

This manual describes how to use OCO (One-Cancels-the-Other) and trailing stop orders in the Quantify trading framework, covering all supported asset classes (Equities, Futures, Options). It explains the API, order lifecycle, and integration with the OrderManager and portfolio/PnL tracking.

---

## What Are OCO and Trailing Stop Orders?

- **OCO (One-Cancels-the-Other) Order:**
  - An order type where two linked orders are placed simultaneously. If one order is executed (filled), the other is automatically cancelled.
  - Commonly used for take-profit and stop-loss scenarios.

- **Trailing Stop Order:**
  - A stop order that "trails" the market price by a fixed amount or percentage. If the price moves in your favor, the stop price moves with it. If the price reverses, the stop stays put, and a market/limit order is triggered if the stop is hit.
  - Can be used as a leg in an OCO order.

---

## Supported Asset Classes

- Equities
- Futures
- Options

---

## Supported Brokers

- **SaxoBroker**: Native OCO support with full trailing stop functionality
- **BinanceBroker**: OCO emulation for futures with auto-cancel logic
- **KrakenCleanBroker**: OCO emulation for futures with auto-cancel logic
- **IBBroker**: Native OCO support (if available)
- **MockBroker**: OCO emulation for testing

---

## Placing OCO and Trailing Stop Orders

### SaxoBroker (Native OCO Support)

Use the `place_oco_order` method on your SaxoBroker instance:

```python
oco_result = await broker.place_oco_order(
    symbol="AAPL",
    side="Buy",
    quantity=1,
    take_profit={
        "order_type": "LIMIT",
        "price": 200.0
    },
    stop_loss={
        "order_type": "TRAILING_STOP",
        "trailing_amount": 5.0  # or trailing_percent: 2.5
    },
    time_in_force="GTC",  # or "DAY", "IOC", "FOK"
    strategy_name="MyStrategy"
)
```

### Binance & Kraken (OCO Emulation)

For brokers without native OCO support, the framework emulates OCO behavior:

```python
oco_result = await broker.place_oco_order(
    symbol="BTCUSDT",
    side="Buy",
    quantity=Decimal("0.1"),
    price=Decimal("50000"),  # Take profit price
    stop_price=Decimal("48000")  # Stop loss price
)
```

### Parameters
- `symbol`: Instrument symbol/UIC
- `side`: "Buy" or "Sell"
- `quantity`: Order quantity
- `take_profit`: Dict with order_type, price, trailing params, etc. (Saxo)
- `stop_loss`: Dict with order_type, price, trailing params, etc. (Saxo)
- `price`: Take profit price (Binance/Kraken)
- `stop_price`: Stop loss price (Binance/Kraken)
- `time_in_force`: "GTC", "DAY", "IOC", "FOK" (Saxo)
- `strategy_name`: (Optional) For PnL tracking

---

## Modifying OCO Orders

### SaxoBroker (Native OCO Support)

You can modify the take-profit and/or stop-loss legs of an existing OCO order using the `modify_oco_order` method:

```python
# Assume oco_result is the dict returned by place_oco_order
modified_oco = await broker.modify_oco_order(
    oco_result,
    new_take_profit={"order_type": "LIMIT", "price": 210.0},  # Move TP to 210
    new_stop_loss=None  # Leave SL unchanged
)
```

You can modify either or both legs:

```python
modified_oco = await broker.modify_oco_order(
    oco_result,
    new_take_profit={"order_type": "LIMIT", "price": 210.0},
    new_stop_loss={"order_type": "TRAILING_STOP", "trailing_amount": 7.5}
)
```

### Parameters
- `oco_order`: The OCO order dict as returned by `place_oco_order` (must include `oco_group_id` and `orders`).
- `new_take_profit`: (Optional) Dict with new parameters for the take-profit leg (same format as in `place_oco_order`).
- `new_stop_loss`: (Optional) Dict with new parameters for the stop-loss leg (same format as in `place_oco_order`).

### Return Value
Returns a dict with the same structure as `place_oco_order`:
```python
{
    "oco_group_id": ...,  # The OCO group ID
    "orders": [take_profit_order, stop_loss_order]  # Updated order objects
}
```

### Important Notes
- **Cancel & Replace:** Saxo does not support in-place modification of orders. The modify function cancels the old leg and places a new one. There is a brief window where neither order is active.
- **Atomicity:** If a leg is filled or cancelled before modification, it will not be replaced. Always check the returned order statuses.
- **OrderManager Integration:** The OCO group in OrderManager is updated automatically to track the new order IDs.
- **Error Handling:** If placing the new order fails after cancelling the old one, you may be left with only one active leg. Always check the result and consider adding your own error handling.

---

## Auto-Cancel Logic

### How It Works

The framework implements robust auto-cancel logic for OCO orders:

1. **Event-Driven:** Order status updates are published to the EventManager
2. **OCO Detection:** The `_on_order_update` handler checks if a filled/canceled order is part of an OCO group
3. **Auto-Cancellation:** If one leg is executed, the other leg(s) are automatically canceled
4. **Logging:** All auto-cancel actions are logged for audit purposes

### Implementation Details

```python
async def _on_order_update(self, order_event):
    """Handle order status updates for OCO auto-cancel logic."""
    order_id = getattr(order_event, 'order_id', None) or order_event.get('order_id')
    status = getattr(order_event, 'status', None) or order_event.get('status')
    
    # Only act if order is filled or canceled
    if status.lower() not in ('filled', 'canceled', 'cancelled'):
        return
        
    # Check if this order is part of an OCO group
    oco_group_id = OrderManager.get_oco_group_id(order_id)
    if not oco_group_id:
        return
        
    # Get all order IDs in the group and cancel other legs
    oco_orders = OrderManager.get_oco_group_orders(oco_group_id)
    for other_order_id in oco_orders:
        if other_order_id != order_id:
            await self.cancel_order(other_order_id, order_event.get('symbol', ''))
            self.logger.info(f"OCO auto-cancel: canceled order {other_order_id} after {order_id} was {status}")
```

### Broker-Specific Behavior

- **SaxoBroker:** Uses native OCO functionality, auto-cancel handled by broker
- **BinanceBroker:** Emulated OCO with auto-cancel via `cancel_futures_order`
- **KrakenCleanBroker:** Emulated OCO with auto-cancel via `cancel_order`
- **IBBroker:** Uses native OCO functionality if available
- **MockBroker:** Emulated OCO for testing purposes

---

## OrderManager: OCO & Trailing Stop Logic

- **OCO Grouping:**
  - Each OCO order is assigned a unique `oco_group_id`.
  - Both legs (take-profit and stop-loss) are tracked as children of the OCO group.
- **Parent-Child Relationships:**
  - Each child order has `parent_order_id` and `is_oco_child` fields.
- **Order State Machine:**
  - Orders transition through: pending → partial fill → filled/cancelled.
  - When one OCO leg is filled, the sibling is automatically cancelled.
- **Trailing Stop Handling:**
  - Trailing stop parameters (`trailing_amount`, `trailing_percent`) are validated and tracked.
- **Automatic Cancellation:**
  - If one leg of the OCO group is executed, the other is cancelled by OrderManager.
- **OCO Group Management:**
  - `add_oco_group()`: Adds orders to OCO groups
  - `get_oco_group_id()`: Retrieves OCO group ID for an order
  - `get_oco_group_orders()`: Gets all orders in an OCO group
  - `update_oco_leg()`: Updates OCO leg after modification

---

## Example: OCO with Trailing Stop (Saxo)

```python
oco_result = await broker.place_oco_order(
    symbol="FDXMZ5",
    side="Sell",
    quantity=2,
    take_profit={"order_type": "LIMIT", "price": 5000.0},
    stop_loss={"order_type": "TRAILING_STOP", "trailing_amount": 50.0},
    time_in_force="DAY",
    strategy_name="SpreadStrategy"
)
```

## Example: OCO for Futures (Binance/Kraken)

```python
oco_result = await broker.place_oco_order(
    symbol="BTCUSDT",
    side="Buy",
    quantity=Decimal("0.1"),
    price=Decimal("52000"),  # Take profit at 52,000
    stop_price=Decimal("48000")  # Stop loss at 48,000
)
```

---

## Order State Transitions & Auto-Cancellation

- When an OCO order is placed, both legs are tracked in OrderManager.
- If one leg is filled (fully or partially), the sibling is automatically cancelled.
- Partial fills are handled for both legs; remaining quantity is managed according to the state machine.
- Order status is updated in real time and reflected in portfolio and PnL tracking.
- Auto-cancel events are logged for audit and debugging purposes.

---

## Integration with PnL Tracking & Portfolio

- All OCO and trailing stop orders are fully integrated with the PnLTracker and PositionManager.
- Strategy name (if provided) is used for per-strategy PnL and trade history.
- Portfolio always reflects the latest state of all OCO/trailing stop orders.
- Auto-cancel actions are tracked and reflected in position calculations.

---

## Error Handling & Best Practices

### Error Scenarios
- **Partial Fills:** OCO logic handles partial fills correctly
- **Network Issues:** Auto-cancel retries on failure
- **Order Rejection:** Failed orders don't trigger auto-cancel
- **Race Conditions:** Event ordering is handled properly

### Best Practices
- Always check order status after placement
- Monitor logs for auto-cancel events
- Use appropriate error handling for order modifications
- Test OCO logic in simulation before live trading

---

## Limitations & Future Extensions

- **Current Limitations:**
  - OCO emulation for Binance/Kraken is futures-only
  - Trailing stops are Saxo-only (native support)
  - Multi-leg OCO (beyond 2 legs) not yet supported

- **Future Extensions:**
  - Multi-leg OCO orders
  - Cross-symbol OCO orders
  - Advanced trailing stop algorithms
  - OCO with conditional orders
  - Simulation/backtesting of OCO logic

---

## See Also
- [OrderManager documentation](./position_manager.md)
- [PnLTracker documentation](./pnl_tracker.md)
- [Broker-specific documentation](./broker_guides.md)

For questions or feature requests, contact the Quantify development team. 