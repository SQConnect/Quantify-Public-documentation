# Ticker-Specific Model System

The ticker-specific model system allows you to train, save, and use separate ML models for different tickers (symbols), enabling ticker-specific optimizations and knowledge retention.

## Overview

Each ticker has unique characteristics:
- **Volatility patterns** - Different assets have different volatility profiles
- **Volume profiles** - Trading volume patterns vary by asset
- **Market microstructure** - Order book dynamics differ between assets
- **Price movements** - Each asset has unique price behavior patterns

The ticker model system captures these differences by training separate models for each ticker.

## Key Features

### 🎯 Ticker-Specific Training
- Separate models for each ticker (ETHUSDT, BTCUSDT, etc.)
- Models learn ticker-specific patterns and behaviors
- Automatic model selection based on the current ticker

### 💾 Knowledge Retention
- Models are saved automatically after training
- Persistent storage of model knowledge
- Metadata tracking for each model (training sessions, performance, etc.)

### 📊 Performance Tracking
- Individual performance metrics per ticker
- Training progress monitoring
- Best performing model identification

### 🔄 Model Management
- Export/import model knowledge for backup or sharing
- Automatic cleanup of old models
- Preloading of models for better performance

## Usage

### 1. Basic Strategy Usage

The `OrderFlowTradingStrategy` automatically uses ticker-specific models:

```yaml
# config/strategies/order_flow_trading_strategy.yaml
symbol: "ETHUSDT"  # This will use the ETHUSDT-specific model

# The strategy will automatically:
# 1. Load existing ETHUSDT model if available
# 2. Create new ETHUSDT model if none exists
# 3. Train the model with ETHUSDT-specific data
# 4. Save the trained model for future use
```

### 2. Model Management Script

Use the management script to view and manage your ticker models:

```bash
# View status of all ticker models
python utils/manage_ticker_models.py --action status

# Export model knowledge for backup
python utils/manage_ticker_models.py --action export --ticker ETHUSDT --file ethusdt_model_backup.pkl

# Import model knowledge
python utils/manage_ticker_models.py --action import --file ethusdt_model_backup.pkl

# Preload models for better performance
python utils/manage_ticker_models.py --action preload --tickers ETHUSDT BTCUSDT ADAUSDT

# Clean up old models (older than 30 days)
python utils/manage_ticker_models.py --action cleanup --days 30
```

### 3. Programmatic Usage

You can also use the ticker model manager directly in your code:

```python
from ml.ticker_model_manager import TickerModelManager

# Initialize manager
base_config = {
    'model_type': 'xgboost',
    'min_training_samples': 1000,
    'retrain_interval': 500
}
manager = TickerModelManager(base_config)

# Get model for specific ticker
ethusdt_model = await manager.get_or_create_model('ETHUSDT')

# Make ticker-specific prediction
prediction = await manager.predict_for_ticker('ETHUSDT', order_events, market_data)

# Add training data for specific ticker
await manager.add_training_sample('ETHUSDT', features, flow_type, impact, time_to_execution)
```

## File Structure

```
data/models/tickers/
├── model_metadata.json          # Metadata for all ticker models
├── ethusdt_order_flow_model.pkl # ETHUSDT-specific model
├── btcusdt_order_flow_model.pkl # BTCUSDT-specific model
└── adausdt_order_flow_model.pkl # ADAUSDT-specific model
```

## Model Metadata

Each model tracks the following metadata:

```json
{
  "ETHUSDT": {
    "created_at": "2024-01-15T10:30:00",
    "training_sessions": 3,
    "total_samples": 2500,
    "last_trained": "2024-01-15T15:45:00",
    "performance_metrics": {
      "flow_type_accuracy": 0.85,
      "impact_mse": 0.0012,
      "time_to_execution_mse": 0.45
    },
    "model_version": "1.0"
  }
}
```

## Training Process

### 1. Data Collection
- Each ticker collects its own training data
- Features are generated specific to that ticker's market behavior
- Training samples accumulate over time

### 2. Model Training
- Models train when they reach the minimum sample threshold
- Training uses ticker-specific data only
- Models retrain periodically with new data

### 3. Model Persistence
- Trained models are automatically saved
- Model knowledge is preserved between strategy restarts
- Metadata is updated with training statistics

## Best Practices

### 🎯 Ticker Selection
- Train models for your most actively traded tickers first
- Focus on tickers with sufficient volume and activity
- Consider market hours and liquidity when selecting tickers

### 📈 Training Data Quality
- Ensure sufficient training data (minimum 1000 samples recommended)
- Include diverse market conditions in training data
- Regularly retrain models with fresh data

### 💾 Model Management
- Export important models for backup
- Clean up old, unused models regularly
- Monitor model performance and retrain when necessary

### 🔄 Performance Monitoring
- Track model accuracy and prediction quality
- Compare performance across different tickers
- Use best-performing models as templates for new tickers

## Example Workflow

### 1. Initial Setup
```bash
# Start with ETHUSDT model
# Configure strategy for ETHUSDT
# Let it collect training data and train the model
```

### 2. Expand to Multiple Tickers
```bash
# Add BTCUSDT configuration
# Preload models for better performance
python utils/manage_ticker_models.py --action preload --tickers ETHUSDT BTCUSDT

# Check model status
python utils/manage_ticker_models.py --action status
```

### 3. Model Backup and Sharing
```bash
# Export well-trained models
python utils/manage_ticker_models.py --action export --ticker ETHUSDT --file models/backups/ethusdt_v1.pkl

# Share models between environments
python utils/manage_ticker_models.py --action import --file models/backups/ethusdt_v1.pkl
```

### 4. Performance Optimization
```bash
# Monitor performance and identify best models
python utils/manage_ticker_models.py --action status

# Clean up old models
python utils/manage_ticker_models.py --action cleanup --days 30
```

## Advanced Features

### Model Versioning
- Each model tracks its version and training history
- Easy rollback to previous model versions
- Performance comparison across model versions

### Knowledge Transfer
- Export model knowledge for sharing between systems
- Import pre-trained models to bootstrap new environments
- Backup and restore model knowledge

### Performance Analytics
- Track model performance over time
- Identify best-performing tickers and models
- Optimize training parameters based on performance data

## Troubleshooting

### Model Not Training
- Check if minimum training samples threshold is met
- Verify data quality and feature generation
- Check logs for training errors

### Poor Model Performance
- Increase training data quantity
- Verify feature quality and relevance
- Consider adjusting model hyperparameters

### Model Loading Issues
- Check file permissions and paths
- Verify model file integrity
- Check for version compatibility issues

## Configuration

### Base Model Configuration
```python
base_config = {
    'model_type': 'xgboost',           # Model type (xgboost, lightgbm, random_forest)
    'min_training_samples': 1000,      # Minimum samples before training
    'retrain_interval': 500,           # Retrain every N new samples
    'prediction_horizon': 5,           # Prediction horizon in minutes
    'sequence_length': 20,             # Sequence length for time series
    'large_order_threshold': 1000      # Threshold for large order detection
}
```

### Directory Configuration
```python
# Custom model directory
manager = TickerModelManager(base_config, models_dir="custom/models/path")
```

This ticker-specific model system provides a robust foundation for training and managing ML models tailored to individual ticker characteristics, enabling better prediction accuracy and knowledge retention across different trading instruments. 