# Futures Price Prediction Service

## Overview

The Futures Price Prediction Service is a new ML service handler that uses LSTM (Long Short-Term Memory) neural networks to predict futures prices. It provides long/short position flags, predicted price values, and confidence scores for trading decisions.

## Features

- **LSTM-based Price Prediction**: Uses deep learning to predict future price movements
- **Automatic Training**: Models train automatically when sufficient data is available
- **Position Flags**: Returns "long", "short", or "neutral" position recommendations
- **Confidence Scores**: Provides confidence levels for predictions (0.0 to 1.0)
- **Model Persistence**: Saves trained models to disk for reuse
- **Memory Management**: Includes memory monitoring and optimization
- **Configurable Parameters**: Customizable sequence length, prediction horizon, and training settings

## Architecture

### Components

1. **FuturesPricePredictionHandler**: Main service handler that manages requests
2. **FuturesPricePredictor**: LSTM model implementation with training capabilities
3. **FuturesPricePredictionClient**: Client library for easy integration
4. **FuturesPricePrediction**: DTO for prediction results

### Model Architecture

The LSTM model uses the following architecture:
- Input: Sequence of market data features (60 time steps by default)
- LSTM layers: 2 layers with 50 units each and dropout (0.2)
- Dense layers: 25 units followed by 3 output units
- Output: [predicted_price, direction_probability, confidence]

## Configuration

Add the following configuration to your `config/main.yaml`:

```yaml
ml_service:
  # ... existing config ...
  
  futures_price_prediction:
    sequence_length: 60  # Number of time steps for LSTM input
    prediction_horizon: 5  # Periods ahead to predict
    min_training_samples: 1000  # Minimum samples for training
    retrain_interval_hours: 24  # Retrain frequency
    model_save_path: "models/futures_price_prediction"
    lstm_config:
      hidden_units: 50
      dropout_rate: 0.2
      learning_rate: 0.001
      batch_size: 32
      epochs: 50
    feature_config:
      use_technical_indicators: true
      use_order_book_features: true
      use_volume_features: true
      use_funding_rate: true
      use_basis_features: true
```

## Usage

### Using the Client

```python
import asyncio
from ml_service.futures_price_prediction_client import FuturesPricePredictionClient
from ml_service.dto import FuturesMarketData

async def example():
    # Initialize client
    client = FuturesPricePredictionClient(config)
    
    # Prepare market data
    market_data = [
        FuturesMarketData(
            symbol="BTC-USD",
            contract_month="2024-12",
            current_price=50000.0,
            bid_price=49999.0,
            ask_price=50001.0,
            bid_size=100.0,
            ask_size=100.0,
            volume=1000.0,
            open_interest=10000.0,
            funding_rate=0.0001,
            basis=0.0,
            next_contract_price=50000.0,
            days_to_expiry=30,
            timestamp=datetime.now()
        )
        # ... more data points
    ]
    
    # Make prediction
    prediction = await client.predict_price("BTC-USD", market_data)
    
    if prediction:
        print(f"Position: {prediction.position_flag}")
        print(f"Predicted Price: ${prediction.predicted_price:,.2f}")
        print(f"Confidence: {prediction.confidence_score:.3f}")
        print(f"Direction Probability: {prediction.direction_probability:.3f}")
    
    # Get detailed prediction with recommendation
    result = await client.get_prediction_with_confidence("BTC-USD", market_data)
    if result.get('success'):
        recommendation = result['recommendation']
        print(f"Action: {recommendation['action']}")
        print(f"Reason: {recommendation['reason']}")
    
    await client.close()

# Run the example
asyncio.run(example())
```

### Direct Service Calls

You can also make direct calls to the service:

```python
# Predict price
request = {
    'ticker': 'BTC-USD',
    'payload': {
        'market_data': [market_data_dicts]
    }
}
response = await service_client.send_request(
    'futures_price_prediction_handler', 'predict_price', request
)

# Train model
request = {
    'ticker': 'BTC-USD',
    'payload': {
        'market_data': [training_data_dicts]
    }
}
response = await service_client.send_request(
    'futures_price_prediction_handler', 'train_model', request
)

# Get model status
request = {'ticker': 'BTC-USD'}
response = await service_client.send_request(
    'futures_price_prediction_handler', 'get_model_status', request
)
```

## API Reference

### Commands

1. **predict_price**: Get price prediction for a futures contract
2. **train_model**: Train or retrain the LSTM model
3. **add_training_data**: Add new market data for training
4. **get_model_status**: Get current model status and statistics

### Response Format

#### Prediction Response
```json
{
    "success": true,
    "prediction": {
        "ticker": "BTC-USD",
        "position_flag": "long",
        "predicted_price": 51000.0,
        "confidence_score": 0.85,
        "direction_probability": 0.72,
        "prediction_horizon": 5,
        "model_last_trained": "2024-01-15T10:30:00Z",
        "features_used": 13
    },
    "ticker": "BTC-USD"
}
```

#### Training Response
```json
{
    "success": true,
    "message": "Model for BTC-USD trained successfully",
    "ticker": "BTC-USD",
    "training_samples": 1500,
    "last_training_time": "2024-01-15T10:30:00Z"
}
```

#### Model Status Response
```json
{
    "success": true,
    "ticker": "BTC-USD",
    "model_trained": true,
    "training_samples": 1500,
    "last_training_time": "2024-01-15T10:30:00Z",
    "should_retrain": false,
    "sequence_length": 60,
    "prediction_horizon": 5
}
```

## Features Used

The model uses the following features from market data:

1. **Price Features**: current_price, bid_price, ask_price
2. **Volume Features**: bid_size, ask_size, volume, open_interest
3. **Futures-Specific**: funding_rate, basis, days_to_expiry
4. **Derived Features**: 
   - Spread ratio: (ask_price - bid_price) / current_price
   - Bid ratio: bid_size / (bid_size + ask_size)
   - Volume/OI ratio: volume / open_interest

## Model Training

### Automatic Training
- Models train automatically when sufficient data is available (default: 1000 samples)
- Retraining occurs every 24 hours by default
- Training data is automatically cleaned (keeps last 30 days)

### Manual Training
- Call `train_model` command to force training
- Can include additional data during training
- Training progress is logged

### Model Persistence
- Trained models are saved to `models/futures_price_prediction/`
- Models are automatically loaded on service startup
- Each ticker has its own model file

## Performance Considerations

### Memory Management
- Automatic memory monitoring every 5 minutes
- Memory optimization when usage exceeds 800MB
- Old predictors are cleaned up to free memory

### Training Performance
- Uses TensorFlow for LSTM implementation
- Supports GPU acceleration if available
- Batch processing for efficient training

### Prediction Performance
- Fast inference once model is trained
- Supports real-time predictions
- Minimal latency for production use

## Error Handling

The service includes comprehensive error handling:

- **Insufficient Data**: Returns error when not enough data for prediction
- **Model Not Trained**: Returns error when model hasn't been trained
- **Training Failures**: Logs errors and returns failure status
- **Memory Issues**: Automatic cleanup and optimization

## Monitoring

### Logs
- Training progress and model performance
- Memory usage and optimization events
- Prediction requests and results
- Error conditions and recovery

### Metrics
- Model training frequency and success rate
- Prediction accuracy and confidence distribution
- Memory usage patterns
- Request latency and throughput

## Example Implementation

See `examples/futures_price_prediction_example.py` for a complete working example that demonstrates:

- Generating sample market data
- Adding training data
- Training the model
- Making predictions
- Getting recommendations

## Dependencies

Required packages (already included in `requirements_ml.txt`):
- tensorflow>=2.13.0
- scikit-learn>=1.3.0
- pandas>=2.0.0
- numpy>=1.24.0

## Future Enhancements

Potential improvements for future versions:

1. **Multi-timeframe Support**: Predictions for different time horizons
2. **Ensemble Models**: Combine multiple LSTM models for better accuracy
3. **Feature Engineering**: More sophisticated technical indicators
4. **Real-time Updates**: Streaming model updates with new data
5. **Model Versioning**: Track model performance and rollback capabilities
6. **Cross-validation**: More robust model evaluation
7. **Hyperparameter Tuning**: Automatic optimization of model parameters 