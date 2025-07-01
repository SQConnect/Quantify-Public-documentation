# Quantify CLI Manual

This manual provides comprehensive documentation for the command-line interface in the Quantify Trading Framework. The **Framework CLI** is used for managing the core system and strategies.

## Table of Contents

1. [Framework CLI](#framework-cli)
   - [Starting the Framework](#starting-the-framework)
   - [Managing Strategies](#managing-strategies)
   - [Interactive Mode](#interactive-mode)
   - [Daemon Mode](#daemon-mode)
2. [Examples](#examples)
3. [Common Workflows](#common-workflows)

---

## Framework CLI

The main framework CLI is accessed through `main.py` and provides commands for managing the core trading system.

### Basic Usage
```bash
python main.py <command> [options]
```

### Available Commands

#### Starting the Framework

**Start all configured strategies:**
```bash
python main.py start
```

**Start specific strategies:**
```bash
python main.py start strategy1 strategy2
```

**Start with custom configuration:**
```bash
python main.py start --config config/custom.yaml --loglevel DEBUG
```

**Start in daemon mode (background):**
```bash
python main.py start --daemon --pidfile /tmp/quantify.pid --logfile logs/daemon.log
```

#### Managing Strategies

**Check framework status:**
```bash
python main.py status
```

**Stop a specific strategy:**
```bash
python main.py stop-strategy --strategy "MyStrategy_Instance"
```

**Restart a strategy:**
```bash
python main.py restart-strategy --strategy "MyStrategy_Instance"
```

**Deploy a new strategy:**
```bash
python main.py deploy-strategy --config config/strategies/new_strategy.yaml
```

#### Interactive Mode

**Enter interactive management mode:**
```bash
python main.py interactive
```

Interactive mode provides:
- Real-time strategy status monitoring
- Performance metrics viewing
- Interactive strategy management
- Live log viewing
- Command execution interface

#### Daemon Mode

**Start framework as a daemon:**
```bash
python main.py start --daemon --pidfile /var/run/quantify.pid --logfile /var/log/quantify.log
```

**Stop daemon:**
```bash
python main.py stop --pidfile /var/run/quantify.pid
```

**Check daemon status:**
```bash
python main.py status
```

### Framework CLI Options

| Option | Description | Default |
|--------|-------------|---------|
| `--config` | Path to main configuration file | `config/main.yaml` |
| `--loglevel` | Logging level (DEBUG, INFO, WARNING, ERROR, CRITICAL) | `INFO` |
| `--daemon` | Run as background daemon | `False` |
| `--pidfile` | PID file location for daemon mode | `/tmp/quantify_daemon.pid` |
| `--logfile` | Log file location for daemon mode | `logs/trading.log` |

---

## Examples

### Strategy Development Workflow

1. **Create strategy configuration:**
```bash
# Create your strategy configuration file manually
cp config/templates/strategy_template.yaml config/my_strategy.yaml
```

2. **Deploy strategy to framework:**
```bash
python main.py deploy-strategy --config config/my_strategy.yaml
```

3. **Start the framework:**
```bash
python main.py start
```

4. **Monitor live performance:**
```bash
python main.py interactive
```

### Strategy Management

**Check running strategies:**
```bash
python main.py status
```

**Stop a specific strategy:**
```bash
python main.py stop-strategy --strategy "MyStrategy_Instance"
```

**Restart a strategy with new config:**
```bash
python main.py restart-strategy --strategy "MyStrategy_Instance"
```

---

## Common Workflows

### Development & Testing
```bash
# 1. Create strategy configuration
cp config/templates/strategy_template.yaml config/my_strategy.yaml

# 2. Deploy to framework
python main.py deploy-strategy --config config/my_strategy.yaml

# 3. Start framework
python main.py start --loglevel DEBUG

# 4. Monitor in interactive mode
python main.py interactive
```

### Production Deployment
```bash
# 1. Start framework in daemon mode
python main.py start --daemon --pidfile /var/run/quantify.pid --logfile /var/log/quantify.log

# 2. Check status
python main.py status

# 3. Monitor logs (external tools)
tail -f logs/trading.log

# 4. Interactive monitoring when needed
python main.py interactive
```

### Strategy Updates
```bash
# 1. Stop specific strategy
python main.py stop-strategy --strategy "MyStrategy_Instance"

# 2. Update configuration
# Edit config/my_strategy.yaml

# 3. Restart strategy
python main.py restart-strategy --strategy "MyStrategy_Instance"

# 4. Verify status
python main.py status
```

---

## Tips & Best Practices

### Configuration Management
- Keep separate configs for development, testing, and production
- Use descriptive names for strategy instances
- Always validate configuration files before deployment

### Production Operations
- Use daemon mode for production deployments
- Monitor log files regularly
- Have backup configurations ready
- Test configuration changes in development first

### Monitoring & Debugging
- Use interactive mode for real-time monitoring
- Set appropriate log levels for your environment
- Use DEBUG level for development, INFO for production
- Monitor strategy performance metrics regularly

This CLI manual provides comprehensive coverage of all command-line interfaces in the Quantify framework. Use it as a reference for both development and production workflows. 