# Session Context: Cryptofeed + InfluxDB Integration
## Date: 2025-11-24 03:49 (Vietnam Time) - UPDATED

## COMPLETED ✅

### Auto-Reconnect Logic Added to cryptofeed_influxdb_writer.py
New features implemented:

1. **Heartbeat Monitoring** (every 60 seconds)
   - Tracks last update time for each symbol
   - Monitors 10 major pairs (BTC, ETH, SOL, XRP, DOGE, ADA, AVAX, LINK, LTC, DOT)

2. **Stale Data Detection**
   - Threshold: 10 minutes without updates = stale
   - If 5+ major pairs are stale → triggers reconnect

3. **Auto-Reconnect with Exponential Backoff**
   - Base delay: 30 seconds
   - Max attempts: 5
   - Delay doubles each attempt (30s → 60s → 120s → 240s → 300s)
   - Caps at 5 minutes max delay

4. **Statistics Tracking**
   - Tracks reconnect count and last reconnect time
   - Logs heartbeat status every 60 seconds

## CURRENT STATUS

### ⚠️ ISSUE: Major Pairs Missing OHLCV (Prior to Fix)
**Problem**: BTC, ETH, SOL, XRP, DOGE, ADA, AVAX, LINK, LTC, DOT have stale OHLCV data

| Pair | Last OHLCV Time | Status |
|------|-----------------|--------|
| BTC/USDT:USDT | 2025-11-23 20:03 UTC | ❌ ~7h stale |
| GALA/USDT:USDT | 2025-11-23 20:36 UTC | ✅ Recent |

**Root Cause**: Cryptofeed websocket on Host 73 partially disconnected from major pair streams.

## DEPLOYMENT REQUIRED

### Step 1: Copy updated file to Host 73
```bash
scp user_data/services/cryptofeed_influxdb_writer.py freqai@192.168.3.73:/home/freqai/freqtrade/user_data/services/
```

### Step 2: Restart the service
```bash
ssh freqai@192.168.3.73
sudo systemctl restart cryptofeed-influxdb
sudo journalctl -u cryptofeed-influxdb -f
```

### Step 3: Verify heartbeat is working
Look for logs like:
```
✓ Heartbeat OK - All major pairs receiving data. Stats: Candles=XXX, Funding=XXX, Reconnects=0
```

## ARCHITECTURE

```
Host 73 (Clean IP: 103.155.254.55)
├── cryptofeed-influxdb.service ✅ NOW WITH AUTO-RECONNECT
│   ├── On startup: Downloads 1500 historical candles
│   ├── Real-time: WebSocket streaming
│   ├── Heartbeat: Every 60 seconds checks data freshness
│   └── Auto-reconnect: If 5+ major pairs stale → reconnect
│
InfluxDB (192.168.3.6:8086 - 360-day retention)
├── ohlcv_3m / ohlcv_15m / ohlcv_1h
├── funding_rate ✅ Working
└── mark_price ✅ Working
│
Feeders (75, 120, 121, 124)
└── influxdb-data-sync.timer (every 3 min)
    └── Syncs to local feather files

freqtrade/exchange/exchange.py
└── _fetch_funding_rate_history() ✅ Reads from InfluxDB first
```

## KEY FILES

| File | Purpose | Status |
|------|---------|--------|
| `user_data/services/cryptofeed_influxdb_writer.py` | Main data producer with auto-reconnect | ✅ Updated |
| `freqtrade/exchange/exchange.py` | Funding rate from InfluxDB (line ~2200) | ✅ Done |
| `user_data/services/influxdb_data_sync.py` | Sync InfluxDB → feather | ✅ Deployed |
| `user_data/services/influxdb_dataprovider.py` | Funding rate reader | ✅ Deployed |
| `user_data/services/influxdb_ohlcv_reader.py` | OHLCV reader | ✅ Deployed |

## AUTO-RECONNECT CONFIGURATION

```python
# user_data/services/cryptofeed_influxdb_writer.py
HEARTBEAT_INTERVAL = 60        # Check every 60 seconds
STALE_THRESHOLD_MINUTES = 10   # Data stale after 10 minutes
MAX_RECONNECT_ATTEMPTS = 5     # Max reconnect tries
RECONNECT_DELAY_BASE = 30      # Base delay (exponential backoff)

# Major pairs monitored:
MAJOR_PAIRS = [
    "BTC-USDT-PERP", "ETH-USDT-PERP", "SOL-USDT-PERP", "XRP-USDT-PERP", 
    "DOGE-USDT-PERP", "ADA-USDT-PERP", "AVAX-USDT-PERP", "LINK-USDT-PERP",
    "LTC-USDT-PERP", "DOT-USDT-PERP"
]
```

## EXPECTED LOG OUTPUT

### Healthy Operation
```
✓ Heartbeat OK - All major pairs receiving data. Stats: Candles=1500, Funding=163, Reconnects=0
```

### When Stale Data Detected
```
⚠️ 3 major pairs have stale data: ['BTC-USDT-PERP', 'ETH-USDT-PERP', 'SOL-USDT-PERP']
```

### When Reconnect Triggered
```
❌ 6/10 major pairs stale - triggering reconnect!
🔄 Reconnect attempt 1/5 in 30s...
🚀 Starting Cryptofeed real-time streaming...
```

## NEXT STEPS

1. ⚡ **Deploy to Host 73** - Copy updated file and restart service
2. Monitor logs to verify auto-reconnect works
3. Consider adding Telegram/Slack alerts for reconnects
