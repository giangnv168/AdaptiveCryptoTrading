# Cryptofeed + InfluxDB Architecture (UPDATED 2025-11-24)

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DATA FLOW ARCHITECTURE                           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│    Binance      │        │   Host 73       │        │   InfluxDB      │
│   WebSocket     │───────▶│   Cryptofeed    │───────▶│  192.168.3.6    │
│   (164 pairs)   │        │  103.155.254.55 │        │  market_data    │
└─────────────────┘        └─────────────────┘        └────────┬────────┘
                                                               │
        ┌──────────────────────────────────────────────────────┤
        │                                                      │
        ▼                                                      ▼
┌───────────────────┐  ┌───────────────────┐  ┌───────────────────────────┐
│  Host 75          │  │  Host 120-124     │  │  Host 6 (Controllers)     │
│  3 feeders        │  │  9 feeders        │  │  - BOT3 Strategy          │
│  (LONG-PRODUCER)  │  │  (SHORT variants) │  │  - Optuna Optimizer       │
└───────────────────┘  └───────────────────┘  └───────────────────────────┘
```

## Data Available in InfluxDB

| Measurement | Description | Update Frequency |
|-------------|-------------|------------------|
| `ohlcv_3m` | 3-minute OHLCV candles | Every 3 minutes |
| `funding_rate` | Funding rate + predicted_rate | Real-time updates |
| `mark_price` | Mark price ticker | Real-time updates |

## Configuration Summary

### Cryptofeed Service (Host 73)
- **Service**: `cryptofeed-influxdb.service`
- **Outbound IP**: 103.155.254.55 (clean, not Binance-banned)
- **Pairs**: 164 pairs streaming
- **Status**: Active/Running

### Feeder Configuration Changes
All 12 feeders configured with:
```json
{
  "exchange": {
    "funding_fee_times": []  // Disabled - use InfluxDB instead
  }
  // external_message_consumer: REMOVED (old 12 sub-producers disabled)
}
```

## Feeder Distribution

| Host | Feeders | Type | Status |
|------|---------|------|--------|
| 75 | 75a, 75b, 75c | LONG-PRODUCER | ✅ Running |
| 120 | 120a, 120b, 120c | SHORT-PRODUCER | ✅ Running |
| 121 | 121a, 121b, 121c | SHORT-MEANREV | ✅ Running |
| 124 | 124a, 124b, 124c | SHORT-MTF | ✅ Running |

## Reading from InfluxDB

### Module Locations
```
/home/freqai/freqtrade/user_data/services/influxdb_ohlcv_reader.py      # OHLCV, mark_price
/home/freqai/freqtrade/user_data/services/influxdb_dataprovider.py     # Custom Funding Rate Provider
```

### Usage Example - OHLCV Reader
```python
from user_data.services.influxdb_ohlcv_reader import (
    get_ohlcv, 
    get_funding_rate, 
    get_mark_price
)

# Get OHLCV data
df = get_ohlcv("BTC/USDT:USDT", "3m", limit=200)

# Get funding rate (simple)
funding = get_funding_rate("BTC/USDT:USDT")  # Returns: 4.3e-06

# Get mark price
mark = get_mark_price("BTC/USDT:USDT")
```

### Usage Example - Custom Funding Rate DataProvider
```python
from user_data.services.influxdb_dataprovider import (
    get_funding_rate,
    get_funding_rate_history,
    get_all_funding_rates
)

# Get latest funding rate (with caching)
rate = get_funding_rate("BTC/USDT:USDT")  # Returns: 2.83e-06

# Get funding rate history
df = get_funding_rate_history("BTC/USDT:USDT", hours=24)

# Get all funding rates for all 163 pairs
rates = get_all_funding_rates()
```

## Disabled/Removed Components

| Component | Status | Reason |
|-----------|--------|--------|
| 12 sub-producer services | ❌ Disabled | Replaced by Cryptofeed |
| `external_message_consumer` | ❌ Removed from configs | No longer needed |
| `funding_fee_times` | ⚪ Set to `[]` | Use InfluxDB instead |

## Benefits of New Architecture

1. **Single Data Source**: One Cryptofeed instance feeds all systems
2. **No IP Bans**: Uses clean IP (103.155.254.55) for Binance WebSocket
3. **Real-time Data**: WebSocket updates instead of REST API polling
4. **Reduced Complexity**: 1 service instead of 14 producer services
5. **funding_rate Available**: From InfluxDB via `get_funding_rate()`
