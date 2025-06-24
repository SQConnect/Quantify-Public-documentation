# Quantify Trading Framework Manual

This document provides a comprehensive guide to configuring, running, and managing the Quantify trading framework.

## 1. Introduction

The Quantify framework is a powerful, event-driven platform for developing, testing, and deploying algorithmic trading strategies. It is designed for modularity and scalability, allowing multiple strategies to run concurrently while being managed from a central command-line interface or via direct database commands.

Key features include:
- **Centralized Strategy Management**: Start, stop, pause, and resume strategies via CLI or database.
- **Real-time Monitoring**: Live health, performance, and resource monitoring.
- **Daemonization**: Run the framework as a background process.
- **Event-Driven Architecture**: A core event bus for handling market data, signals, and system events.
- **Database Control**: Interact with and control strategies by inserting commands into a database.
- **Modular Strategy Design**: A clear `BaseStrategy` interface for creating new strategies.

---

## 2. Running the Framework

The framework is operated via the command-line interface (CLI) in `main.py`.

### 2.1. Command-Line Interface (CLI)

The primary entry point is `main.py`, which provides a rich set of commands to manage the lifecycle of your strategies.

**General Usage:**
```bash
python main.py [COMMAND] [OPTIONS]
```

---

#### **Core Commands**

**`start`**
Initializes the framework and runs configured strategies.

```bash
# Start all strategies defined in config/main.yaml
python main.py start

# Start a specific list of strategies by their names
python main.py start MyAwesomeStrategy_1 ElliottWaveStrategy

# Start the framework as a background process (daemon)
python main.py start --daemon

# Specify a custom PID file and log file for daemon mode
python main.py start --daemon --pidfile /var/run/quantify.pid --logfile /var/log/quantify.log
```
- `strategies` (optional): A space-separated list of strategy names to run. If omitted, all strategies in `config/main.yaml` are started.
- `--daemon` / `-d`: Runs the framework in the background.
- `--pidfile`: Specifies a PID file location for the daemon.
- `--logfile`: Specifies a log file location for the daemon.

**`stop-strategy`**
Stops a specific running strategy.

```bash
python main.py stop-strategy --strategy <strategy_id>
```
- `--strategy`: The unique ID of the strategy instance to stop.

**`pause`**
Pauses a running strategy. It will stop processing new events but maintain its state.

```bash
python main.py pause --strategy <strategy_id>
```
- `--strategy`: The unique ID of the strategy to pause.

**`resume`**
Resumes a previously paused strategy.

```bash
python main.py resume --strategy <strategy_id>
```
- `--strategy`: The unique ID of the strategy to resume.

---

#### **Monitoring and Status Commands**

**`status`**
Retrieves the current status of all running strategies from the database.

```bash
python main.py status
```

**`monitor`**
Displays the real-time resource (CPU, memory) usage of strategies.

```bash
# Monitor all strategies
python main.py monitor

# Monitor a specific strategy
python main.py monitor --strategy <strategy_id>
```
- `--strategy`: The unique ID of the strategy to monitor. Defaults to `all`.

**`logs`**
Tails the logs for a specific strategy.

```bash
python main.py logs --strategy <strategy_id> --tail 200
```
- `--strategy`: The unique ID of the strategy to view logs for.
- `--tail`: The number of recent log lines to display. Defaults to 100.

---

#### **Runtime Management Commands**

**`add-strategy`**
Deploys a new strategy at runtime without restarting the framework.

```bash
python main.py add-strategy --name MyNewStrategy --config /path/to/new_strategy_config.yaml
```
- `--name`: A unique name for the new strategy instance.
- `--config`: The path to the strategy's configuration YAML file.

**`interactive`**
Starts the framework in an interactive mode, allowing you to issue commands directly.

```bash
python main.py interactive
```
Once inside, you can type commands like `status`, `stop_strategy --name <id>`, etc.

---

### 2.2. Using a Shell Script (`quantify.sh`)

To simplify execution, you can use a wrapper script.

A simple `quantify.sh` script might look like this:

```sh
#!/bin/bash

# Get the directory of the script
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )"

# Activate your virtual environment if you have one
# source /path/to/your/venv/bin/activate

# Run the python script with all arguments passed to the shell script
python "$DIR/main.py" "$@"
```

Make it executable (`chmod +x quantify.sh`), and you can run the framework like so:
```bash
# Start in daemon mode
./quantify.sh start --daemon

# Check status
./quantify.sh status

# Stop a strategy
./quantify.sh stop-strategy --strategy my_awesome_strategy_instance_001
```

---

## 3. Framework Configuration (`config/main.yaml`)

The main configuration file, `config/main.yaml`, controls all aspects of the framework.

```yaml
# List of strategy configuration files to load on startup
strategies:
  - "config/strategies/elliott_wave.yaml"
  - "config/strategies/short_put_spread.yaml"
  - "config/strategies/futures_calendar_spread.yaml"

# Database configuration
database:
  enabled: true
  path: "data/trading_framework.db"

# Data source configuration (e.g., live feeds)
data_sources:
  - type: "CandleManager" # For live data from brokers

# Broker and execution settings
execution:
  enabled: true
  broker:
    name: "Saxo" # Or "Binance", "IB", "Kraken", "Mock"
    # ... other broker-specific credentials and settings

# Health and performance monitoring settings
monitoring:
  health_check_interval_seconds: 30
  performance_snapshot_interval_seconds: 300 # 5 minutes
  health_check_config:
    heartbeat_threshold_minutes: 5
    max_errors_per_minute: 10
  recovery_policies:
    default:
      max_recovery_attempts: 3
      recovery_delay_seconds: 30
      escalation_policy: 'restart' # or 'pause', 'stop'

# Global resource limits for all strategies
resource_limits:
  max_memory_mb: 1024
  max_cpu_percent: 50
```

---

## 4. Controlling Strategies via the Database

For advanced control or integration with other systems, you can issue commands directly to the framework's database (`data/trading_framework.db` by default). This allows external programs or scripts to manage strategies without using the CLI.

Commands are inserted into the `strategy_commands` table. The framework periodically polls this table for new commands with a `status` of `'pending'`.

**Table Schema:** `strategy_commands`
- `id` (INTEGER, PK): Unique ID for the command.
- `strategy_id` (TEXT): The ID of the strategy to command.
- `command` (TEXT): The command to execute (`pause`, `resume`, `stop`).
- `parameters_json` (TEXT): JSON string for command parameters (e.g., used by `add_strategy`).
- `status` (TEXT): The state of the command (`pending`, `completed`, `failed`).
- `created_at` (TIMESTAMP): When the command was created.
- `processed_at` (TIMESTAMP): When the framework processed the command.
- `result_json` (TEXT): A JSON string with the result of the command execution.

### Example: Pausing and Resuming with SQL

1.  **Find the `strategy_id`**:
    First, find the ID of the strategy you want to control. You can get this from the `status` command or by querying the database:
    ```sql
    SELECT id, name FROM strategy_instances;
    ```
    Assume the ID for your target strategy is `elliott_wave_12345`.

2.  **Insert the `pause` command**:
    ```sql
    INSERT INTO strategy_commands (strategy_id, command, status)
    VALUES ('elliott_wave_12345', 'pause', 'pending');
    ```

3.  **Insert the `resume` command**:
    After the strategy is paused, you can resume it with another command.
    ```sql
    INSERT INTO strategy_commands (strategy_id, command, status)
    VALUES ('elliott_wave_12345', 'resume', 'pending');
    ```
The framework will pick up these commands within its next polling cycle (typically a few seconds) and execute them.

---

## 5. Monitoring and Health Checks

The framework includes a robust system for monitoring the health and resource consumption of all running strategies. This is crucial for ensuring the stability and performance of your trading system.

### 5.1. Health Status

The framework continuously monitors each strategy for:
- **Heartbeats**: Each strategy regularly reports a "heartbeat" to the system. If a heartbeat is not received within the `heartbeat_threshold_minutes` defined in `config/main.yaml`, the strategy is considered unhealthy.
- **Errors**: The system tracks the number of unhandled exceptions a strategy produces. If it exceeds `max_errors_per_minute`, it may be flagged.

To check the health and status of all strategies, use the `status` command:
```bash
python main.py status
```
The output is a YAML-formatted list of your strategies and their current state, including the `status` (`running`, `paused`, `stopped`, `error`), `last_heartbeat`, and `error_count`.

### 5.2. Resource Monitoring

To prevent memory leaks or runaway CPU usage, the framework monitors the resources consumed by each strategy.

You can view the latest resource snapshot using the `monitor` command:
```bash
python main.py monitor
```
This command provides the `cpu_percent` and `memory_mb` for each active strategy. This is useful for identifying strategies that may be impacting system performance.

### 5.3. Log Inspection

Detailed operational logs are the primary tool for debugging or tracking a strategy's behavior. Use the `logs` command to quickly inspect them:
```bash
python main.py logs --strategy <strategy_id>
```

---

## 6. Strategy Development

### 6.1. The `BaseStrategy` Class

All strategies must inherit from `src.strategy_framework.base_strategy.BaseStrategy` (or one of its children like `BaseOptionsStrategy` or `BaseFuturesStrategy`).

You must implement the following abstract methods:

- `async def initialize(self) -> bool`:
  Called once when the strategy starts. Use this to set up subscriptions to market data, initialize indicators, and prepare any other resources. Return `True` for success.

- `async def run(self) -> None`:
  This is the main "tick" loop of your strategy. However, for an event-driven strategy, this method will often do very little. The main logic will reside in event handlers (e.g., `on_candle_closed`). You can use this method for periodic checks if needed.

- `async def cleanup(self) -> None`:
  Called once when the strategy is stopping. Use this to clean up resources, cancel subscriptions, or persist state.

### 6.2. Event-Driven Logic

The framework is event-driven. Instead of polling for data, your strategy should subscribe to events and react to them.

**Subscribing to Candle Events:**
In your `initialize` method, you can subscribe to candle data:
```python
# In your strategy's initialize() method
await self.event_manager.subscribe(
    topic=CandleEventType.CANDLE_CLOSED.to_topic_string(symbol="EURUSD", timeframe="1H"),
    handler=self.on_eurusd_1h_candle
)

# ... elsewhere in your class
async def on_eurusd_1h_candle(self, event: CandleEvent):
    self.logger.info(f"New 1H candle for EURUSD: {event.candle}")
    # ... your logic here ...
```

### 6.3. Placing Orders

Strategies do not interact with the broker directly. They generate `SignalEvent`s and publish them to the event bus. The `ExecutionManager` listens for these events and places the orders.

```python
from src.core.events.signal_event import Signal, SignalEvent
from src.core.events.event_types import CoreEventType

# ... inside your strategy logic ...
trade_signal = Signal(
    symbol="AAPL",
    action="buy", # or "sell"
    quantity=100,
    order_type="market",
    strategy_name=self.strategy_name
)

await self.event_manager.publish(
    topic=CoreEventType.SIGNAL.value,
    event=SignalEvent(signal=trade_signal)
)
```

### 6.4. Strategy Configuration

Each strategy should have its own YAML configuration file in `config/strategies/`.

**Example:** `config/strategies/my_awesome_strategy.yaml`
```yaml
# Unique name for this strategy configuration
name: MyAwesomeStrategy_1

# The Python class name to instantiate
type: MyAwesomeStrategy

# A unique ID for this instance. If not provided, one will be generated.
id: my_awesome_strategy_instance_001

# --- Strategy-specific parameters ---
symbols:
  - "AAPL"
  - "GOOGL"

timeframe: "15m"

stop_loss_pct: 0.05
take_profit_pct: 0.10

# ... any other parameters your strategy needs
```
You must then add the path to this file to the `strategies` list in `config/main.yaml` for it to be loaded at startup, or deploy it at runtime using the `add-strategy` command. 