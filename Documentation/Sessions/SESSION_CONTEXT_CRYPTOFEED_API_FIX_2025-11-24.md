# Session Context: CryptoFeed API Performance Fix

**Date**: 2025-11-24 17:16 (Vietnam Time)
**Status**: ✅ COMPLETED

## Problem

Feeder 75a was generating continuous "Cryptofeed API failed" warnings, causing API timeout errors for all pairs:
```
WARNING - Cryptofeed API failed for ADA/USDT:USDT 1m. Check if cryptofeed-api service is...
```

## Root Causes Identified

1. **Wrong Query Format** (Fixed earlier): API server was querying `ohlcv_{tf}` measurement instead of `ohlcv` with `timeframe` tag
2. **Huge Time Range** (-4320h = 180 days): Caused 2+ minute query times
3. **Native Data Query First** (-720h): Slow with concurrent requests from feeders

## Solution Implemented

**Optimized API Server** (`user_data/services/cryptofeed_api_server.py`):

1. **Prioritize Historical Data First** (faster with optimized time range)
2. **Dynamic Time Range Calculation**: Based on timeframe and limit
   - 1m/1500 candles = 28 hours instead of 4320 hours
   - Formula: `hours_needed = (timeframe_minutes * limit * 1.1) / 60 + 1`
3. **Native Data as Fallback**: Only used when historical data not available

### Key Code Change
```python
# Calculate optimal time range based on timeframe and limit
tf_minutes = {'1m': 1, '3m': 3, '5m': 5, '15m': 15, '30m': 30, '1h': 60, ...}
minutes_needed = tf_minutes.get(timeframe, 60) * limit * 1.1
hours_needed = int(minutes_needed / 60) + 1

# Query historical data FIRST (faster)
query = f'''
    from(bucket: "{INFLUXDB_BUCKET}")
        |> range(start: -{hours_needed}h)
        |> filter(fn: (r) => r._measurement == "ohlcv")
        |> filter(fn: (r) => r.pair == "{pair}")
        |> filter(fn: (r) => r.timeframe == "{timeframe}")
        ...
'''
```

## Performance Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| ADA 1m 1500 candles | 2m 10s | 22.4s | **6x faster** |
| TRX 1m 1500 candles | timeout | 0.58s | **∞** |
| SOL 1m 1500 candles | timeout | 1.4s | **∞** |
| API Failed warnings | continuous | 0 | **100% fixed** |

## Files Modified

1. `user_data/services/cryptofeed_api_server.py` - Optimized query logic

## Deployment

API server deployed to Host 73:
```bash
scp user_data/services/cryptofeed_api_server.py freqai@192.168.3.73:/home/freqai/freqtrade/user_data/services/
sudo systemctl restart cryptofeed-api
```

## Verification

- ✅ API response time < 30 seconds for all pairs
- ✅ 0 "Cryptofeed API failed" warnings in feeder logs
- ✅ Historical data returns 1200-1500 candles per pair

## Architecture Notes

### Data Sources in InfluxDB
1. **Historical Data** (measurement: `ohlcv`, tags: `pair`, `timeframe`)
   - 6 months of historical data downloaded
   - 110 pairs with multiple timeframes
   
2. **Native Cryptofeed** (measurement: `candles-BINANCE_FUTURES`, tags: `symbol`, `interval`)
   - Real-time data from native writer
   - Limited history (~2 hours since service started)

### Query Priority
1. Historical data first (complete, faster with small time range)
2. Native data as fallback (when pair not in historical download)

## Next Steps

- Monitor feeder performance over time
- Consider increasing cache TTL for frequently accessed pairs
- May need to add more pairs to historical download if new pairs are added to feeders
