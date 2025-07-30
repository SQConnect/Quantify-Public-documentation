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

---

## 7. Microservice Architecture

The Quantify framework is built on a robust microservice architecture that provides modularity, scalability, and fault isolation. Each service operates independently and communicates through standardized interfaces.

### 7.1. Service Overview

The framework consists of several specialized microservices, each handling specific aspects of the trading system:

#### **Main Trading Execution Service**
- **Purpose**: Core trading logic, strategy execution, and order management
- **Port**: 5555 (default)
- **Key Components**:
  - Strategy Factory and Runner
  - Order Manager
  - Position Manager
  - Event Manager
  - Broker Interface
- **Responsibilities**:
  - Strategy lifecycle management
  - Order placement and tracking
  - Position monitoring
  - Market data distribution
  - Risk management integration

#### **Risk Management Service**
- **Purpose**: Real-time risk monitoring and control
- **Port**: 5557 (default)
- **Key Features**:
  - Margin call monitoring
  - Assignment risk assessment
  - Real-time assignment notifications
  - Portfolio-wide risk controls
  - Automatic position reduction on margin breaches
- **Integration**: Automatically connects to Position Manager and Order Manager
- **Documentation**: See `docs/risk_management.md` for detailed usage

#### **Data Service**
- **Purpose**: Historical data management and market data aggregation
- **Port**: 5556 (default)
- **Key Features**:
  - Historical data storage and retrieval
  - Real-time data streaming
  - Data normalization across brokers
  - Candle management and OHLC data
  - Market data caching and optimization

#### **News Service**
- **Purpose**: News aggregation and sentiment analysis
- **Port**: 5558 (default)
- **Key Features**:
  - Real-time news feeds
  - Sentiment analysis
  - News filtering by symbols
  - Impact assessment
  - Historical news storage

#### **Machine Learning Service**
- **Purpose**: AI/ML model management and predictions
- **Port**: 5559 (default)
- **Key Features**:
  - Model training and deployment
  - Real-time predictions
  - Feature engineering
  - Model performance monitoring
  - Automated model updates

### 7.2. Service Communication

All microservices communicate using a standardized protocol:

#### **Message Format**
```json
{
  "command": "command_name",
  "ticker": "optional_ticker_or_account",
  "payload": {
    "parameter1": "value1",
    "parameter2": "value2"
  }
}
```

#### **Response Format**
```json
{
  "status": "success|error",
  "message": "Human readable message",
  "data": {
    "result_data": "value"
  }
}
```

### 7.3. Service Management

#### **Starting Services**

All services can be started using the `quantify.sh` script:

```bash
# Start all services
./quantify.sh start

# Start specific services
./quantify.sh start --services main,risk,data

# Start in daemon mode
./quantify.sh start --daemon
```

#### **Service Configuration**

Each service has its own configuration section in `config/main.yaml`:

```yaml
# Main trading service configuration
main_service:
  server:
    host: "0.0.0.0"
    port: 5555
    use_ipc: false
    use_binary: false

# Risk management service configuration
risk_management:
  server:
    host: "0.0.0.0"
    port: 5557
    use_ipc: false
    use_binary: false
  risk_config:
    margin_threshold: 85.0
    assignment_monitoring: true
    auto_position_reduction: true

# Data service configuration
data_service:
  server:
    host: "0.0.0.0"
    port: 5556
    use_ipc: false
    use_binary: false

# News service configuration
news_service:
  server:
    host: "0.0.0.0"
    port: 5558
    use_ipc: false
    use_binary: false

# ML service configuration
ml_service:
  server:
    host: "0.0.0.0"
    port: 5559
    use_ipc: false
    use_binary: false
```

#### **Service Health Monitoring**

Each service provides health monitoring endpoints:

```bash
# Check main service health
curl http://localhost:5555/health

# Check risk service health
curl http://localhost:5557/health

# Check data service health
curl http://localhost:5556/health
```

### 7.4. Service Integration

#### **Automatic Integration**

The main trading service automatically integrates with other services:

- **Risk Management**: Position Manager automatically subscribes to balance and position updates
- **Data Service**: Candle Manager automatically connects for historical data
- **News Service**: Event Manager automatically receives news events
- **ML Service**: Strategies can request predictions through the ML client

#### **Manual Integration**

For custom integrations, use the service clients:

```python
from src.risk_management.risk_client import RiskManagementClient
from src.data.data_client import DataClient
from src.news.news_client import NewsClient
from src.ml.ml_client import MLClient

# Risk management integration
risk_client = RiskManagementClient("AAPL")
await risk_client.subscribe_to_margin_events()
await risk_client.get_risk_assessment()

# Data service integration
data_client = DataClient()
historical_data = await data_client.get_historical_data("AAPL", "1D", limit=100)

# News service integration
news_client = NewsClient()
news_events = await news_client.get_news_for_symbol("AAPL")

# ML service integration
ml_client = MLClient()
prediction = await ml_client.get_prediction("AAPL", "price_direction")
```

### 7.5. Service Development

#### **Creating New Services**

To create a new microservice:

1. **Create the service directory**:
   ```
   src/your_service/
   ├── __init__.py
   ├── main.py
   ├── handlers/
   │   └── your_handler.py
   ├── client.py
   └── dto.py
   ```

2. **Implement the handler**:
   ```python
   from src.service_framework.base_handler import BaseHandler
   
   class YourHandler(BaseHandler):
       def get_commands(self):
           return ["command1", "command2"]
       
       async def _handle_request_impl(self, command: str, ticker: str, payload: Dict[str, Any]):
           if command == "command1":
               return await self._handle_command1(ticker, payload)
           # ... other commands
   ```

3. **Create the main entry point**:
   ```python
   from src.service_framework.service_host import ServiceHost
   from src.your_service.handlers.your_handler import YourHandler
   
   async def main():
       service_host = ServiceHost(config)
       handler = YourHandler()
       service_host.register_handler(handler)
       await service_host.start()
   ```

4. **Add to quantify.sh**:
   ```bash
   # Add your service to the start_services function
   start_your_service() {
       python -m src.your_service.main &
       echo $! > /tmp/quantify_your_service.pid
   }
   ```

#### **Service Best Practices**

- **Fault Isolation**: Each service should handle its own errors and not crash other services
- **Stateless Design**: Services should be stateless when possible for easy scaling
- **Standardized Interfaces**: Use consistent message formats and error handling
- **Health Checks**: Implement health check endpoints for monitoring
- **Logging**: Use structured logging for better debugging
- **Configuration**: Externalize all configuration parameters

### 7.6. Service Scaling

#### **Horizontal Scaling**

Each service can be scaled independently:

```bash
# Start multiple instances of a service
./quantify.sh start --service data --instances 3
./quantify.sh start --service ml --instances 2
```

#### **Load Balancing**

For high-throughput scenarios, use load balancers:

```yaml
# Example with load balancer configuration
data_service:
  load_balancer:
    enabled: true
    instances: 3
    algorithm: "round_robin"
```

#### **Service Discovery**

Services automatically discover each other using the service registry:

```python
# Automatic service discovery
from src.service_framework.service_registry import ServiceRegistry

registry = ServiceRegistry()
data_service = await registry.get_service("data_service")
```

This microservice architecture provides the foundation for a scalable, maintainable, and robust trading system that can handle complex requirements while maintaining clear separation of concerns. 