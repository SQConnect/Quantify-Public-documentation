# Technical Analysis Visualization System

This document provides comprehensive documentation for our unified technical analysis visualization system, which integrates multiple visualization components and uses Plotly to create interactive and static charts.

## Table of Contents
1. [Overview](#overview)
2. [Installation](#installation)
3. [Basic Usage](#basic-usage)
4. [Chart Types](#chart-types)
5. [Export Options](#export-options)
6. [Dashboard Features](#dashboard-features)
7. [Customization](#customization)
8. [Best Practices](#best-practices)
9. [Examples](#examples)

## Overview

The unified visualization system provides:
- Interactive and static charts
- Multiple export formats (HTML, PNG)
- Real-time dashboard
- Comprehensive technical analysis tools
- Strategy performance tracking
- News and event visualization

### Key Components
- Elliott Wave patterns
- Fibonacci levels
- Technical indicators
- Strategy performance
- Volume analysis
- Price predictions
- News events
- Trade history

## Installation

The visualization system requires the following dependencies:
```python
pip install plotly>=5.18.0
pip install dash>=2.14.0
pip install dash-core-components>=2.0.0
pip install dash-html-components>=2.0.0
pip install kaleido>=0.2.1  # For PNG export
pip install psutil>=5.9.0   # Required by kaleido
```

## Basic Usage

```python
from src.visualization.unified_visualizer import UnifiedVisualizer
from src.data.candle_data_manager import CandleData

# Initialize visualizer
visualizer = UnifiedVisualizer(output_dir='output/visualization')

# Add price data
visualizer.add_price_data('BTC/USD', candle_data)

# Add technical indicators
visualizer.add_indicator_data('BTC/USD', 'RSI', rsi_data)
visualizer.add_indicator_data('BTC/USD', 'MACD', macd_data)

# Create and save charts
fig = visualizer.plot_elliott_wave(
    symbol='BTC/USD',
    pattern=wave_pattern,
    show_predictions=True
)

# Save as HTML
visualizer.save_figure(fig, 'analysis.html', format='html')

# Save as PNG
visualizer.save_figure(fig, 'analysis.png', format='png')

# Display in browser
visualizer.show_figure(fig)

# Run interactive dashboard
visualizer.run_server()
```

## Chart Types

### 1. Elliott Wave Chart
```python
fig = visualizer.plot_elliott_wave(
    symbol='BTC/USD',
    pattern=wave_pattern,
    show_predictions=True
)
```

Features:
- Candlestick price chart
- Wave points with labels
- Wave connections
- Volume subplot
- Wave predictions
- Interactive legend

### 2. Fibonacci Levels Chart
```python
fig = visualizer.plot_fibonacci_levels(
    symbol='BTC/USD',
    levels=fibonacci_levels,
    show_volume=True
)
```

Features:
- Candlestick price chart
- Fibonacci level lines
- Level touch points
- Volume subplot
- Interactive legend

### 3. Technical Indicators Chart
```python
fig = visualizer.plot_technical_indicators(
    symbol='BTC/USD',
    indicators=['RSI', 'MACD', 'Bollinger Bands'],
    start_date='2024-01-01',
    end_date='2024-01-31'
)
```

Features:
- Multiple indicator overlay
- Custom date ranges
- Volume analysis
- Interactive legend

### 4. Strategy Performance Chart
```python
fig = visualizer.plot_strategy_performance(
    symbol='BTC/USD',
    start_date='2024-01-01',
    end_date='2024-01-31'
)
```

Features:
- Price and volume
- Trade markers
- Performance metrics
- News events

## Export Options

### HTML Export
```python
visualizer.save_figure(fig, 'analysis.html', format='html')
```
- Interactive charts
- Full Plotly functionality
- Hover information
- Zoom and pan capabilities

### PNG Export
```python
visualizer.save_figure(fig, 'analysis.png', format='png')
```
- High-quality static images
- Custom resolution
- Suitable for reports and documentation
- Fast rendering

## Dashboard Features

### Interactive Dashboard
```python
visualizer.run_server(debug=True, port=8050)
```

Features:
- Date range selector
- Main price chart
- Performance metrics
- News feed
- Trade history
- Real-time updates

### Data Management
```python
# Add price data
visualizer.add_price_data('BTC/USD', candle_data)

# Add indicators
visualizer.add_indicator_data('BTC/USD', 'RSI', rsi_data)

# Add trades
visualizer.add_trade({
    'symbol': 'BTC/USD',
    'side': 'long',
    'entry_price': 50000,
    'exit_price': 55000,
    'entry_time': '2024-01-15 10:00:00',
    'exit_time': '2024-01-15 14:00:00',
    'pnl': 5000
})

# Add news events
visualizer.add_news_event({
    'timestamp': '2024-01-15 12:00:00',
    'headline': 'Bitcoin Breaks $50k',
    'content': 'Bitcoin reaches new all-time high',
    'sentiment_score': 0.8,
    'impact_level': 'high'
})
```

## Customization

### Colors
```python
visualizer.technical_plotter.colors = {
    'bullish': '#26a69a',
    'bearish': '#ef5350',
    'neutral': '#bdbdbd',
    'fibonacci': '#ff9800',
    'wave': '#2196f3',
    'prediction': '#9c27b0'
}
```

### Layout
```python
fig.update_layout(
    title='Custom Title',
    yaxis_title='Custom Y-Axis',
    yaxis2_title='Custom Volume',
    height=1000,
    width=1200,
    showlegend=True,
    legend=dict(
        orientation='h',
        yanchor='bottom',
        y=1.02,
        xanchor='right',
        x=1
    )
)
```

## Best Practices

1. **Chart Organization**:
   - Use clear titles and labels
   - Organize legend logically
   - Maintain consistent color scheme
   - Include volume analysis

2. **Performance**:
   - Limit data points for large timeframes
   - Use appropriate update intervals
   - Optimize for browser rendering
   - Consider mobile viewing

3. **User Experience**:
   - Enable interactive features
   - Provide clear hover information
   - Use intuitive color coding
   - Include zoom controls

4. **Data Presentation**:
   - Show relevant timeframes
   - Highlight key levels
   - Include volume confirmation
   - Display predictions clearly

5. **Export Considerations**:
   - Use HTML for interactive analysis
   - Use PNG for reports and documentation
   - Consider file size for large datasets
   - Maintain consistent styling

## Examples

### 1. Complete Analysis Workflow
```python
# Initialize visualizer
visualizer = UnifiedVisualizer()

# Add data
visualizer.add_price_data('BTC/USD', candle_data)
visualizer.add_indicator_data('BTC/USD', 'RSI', rsi_data)
visualizer.add_indicator_data('BTC/USD', 'MACD', macd_data)

# Create analysis charts
ew_fig = visualizer.plot_elliott_wave('BTC/USD', wave_pattern)
fib_fig = visualizer.plot_fibonacci_levels('BTC/USD', fib_levels)
ind_fig = visualizer.plot_technical_indicators('BTC/USD', ['RSI', 'MACD'])

# Save charts
visualizer.save_figure(ew_fig, 'elliott_wave.png', format='png')
visualizer.save_figure(fib_fig, 'fibonacci_levels.png', format='png')
visualizer.save_figure(ind_fig, 'indicators.png', format='png')

# Run dashboard
visualizer.run_server()
```

### 2. Strategy Performance Analysis
```python
# Add strategy data
visualizer.add_trade({
    'symbol': 'BTC/USD',
    'side': 'long',
    'entry_price': 50000,
    'exit_price': 55000,
    'entry_time': '2024-01-15 10:00:00',
    'exit_time': '2024-01-15 14:00:00',
    'pnl': 5000
})

# Update metrics
visualizer.update_performance_metrics({
    'total_trades': 10,
    'win_rate': 0.6,
    'avg_profit': 2.5,
    'max_drawdown': -5.0,
    'sharpe_ratio': 1.5
})

# Create performance chart
perf_fig = visualizer.plot_strategy_performance('BTC/USD')
visualizer.save_figure(perf_fig, 'performance.png', format='png')
```

### 3. Custom Styling
```python
# Create plotter with custom colors
visualizer.technical_plotter.colors = {
    'bullish': '#00c853',
    'bearish': '#d50000',
    'neutral': '#757575',
    'fibonacci': '#ff6d00',
    'wave': '#2962ff',
    'prediction': '#aa00ff'
}

# Create chart with custom layout
fig = visualizer.plot_elliott_wave(
    symbol='BTC/USD',
    pattern=wave_pattern,
    show_predictions=True
)

# Update layout
fig.update_layout(
    title=dict(
        text='Custom Elliott Wave Analysis',
        x=0.5,
        y=0.95,
        xanchor='center',
        yanchor='top'
    ),
    template='plotly_dark',
    height=900,
    width=1200,
    showlegend=True,
    legend=dict(
        orientation='h',
        yanchor='bottom',
        y=1.02,
        xanchor='right',
        x=1
    )
)

# Save with custom styling
visualizer.save_figure(fig, 'custom_analysis.png', format='png')
``` 