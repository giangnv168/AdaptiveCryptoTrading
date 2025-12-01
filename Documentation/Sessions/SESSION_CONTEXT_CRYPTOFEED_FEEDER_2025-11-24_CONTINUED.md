# Session Context: Cryptofeed Integration with Feeders (Continued)
## Date: November 24, 2025

## Current Issue
**Binance IP Ban**: `103.155.254.48` is banned due to "Way too many requests"
- This is happening because feeders are making direct Binance API calls
- Cryptofeed integration would solve this by using cached data from InfluxDB

## Work Completed This Session

### 1. Sub-feeder Management
- **Host 75**: feeder75 disabled, using only 75a, 75b, 75c
- **Host 120**: feeder120 disabled, using only 120a, 120b, 120c
- Both sets of sub-feeders are running but **WITHOUT Cryptofeed** integration

### 2. Compatibility Issue Identified
The `exchange.py` from Host 73 uses newer imports that don't exist on Host 75/120:
- **Host 73**: `from freqtrade.exchange.exchange_types import ...`
- **Host 75/120**: `from freqtrade.exchange.types import ...`

### 3. Patch Script Created
A patch script was prepared at `/tmp/patch_cryptofeed.sh` that would:
1. Add `_fetch_ohlcv_from_cryptofeed_api` method
2. Modify `_async_get_candle_history` to use Cryptofeed API with fallback

## Next Steps Required

### Immediate Action: Stop Feeders During Ban
The Binance IP ban will persist if feeders keep retrying. Consider stopping sub-feeders temporarily:

```bash
# Stop Host 75 sub-feeders
sshpass -p "NewHopes@168" ssh freqai@192.168.3.75 "echo 'NewHopes@168' | sudo -S systemctl stop freqtrade-feeder75a freqtrade-feeder75b freqtrade-feeder75c"

# Stop Host 120 sub-feeders  
sshpass -p "NewHopes@168" ssh freqai@192.168.3.120 "echo 'NewHopes@168' | sudo -S systemctl stop freqtrade-feeder120a freqtrade-feeder120b freqtrade-feeder120c"
```

### After Ban Expires: Apply Cryptofeed Patch
The patch needs to be applied to both hosts to enable Cryptofeed API usage:

**Host 75:**
```bash
sshpass -p "NewHopes@168" ssh freqai@192.168.3.75 "/tmp/patch_cryptofeed.sh"
```

**Host 120:**
```bash
# First copy the same script to Host 120
sshpass -p "NewHopes@168" scp /tmp/patch_cryptofeed.sh freqai@192.168.3.120:/tmp/
sshpass -p "NewHopes@168" ssh freqai@192.168.3.120 "chmod +x /tmp/patch_cryptofeed.sh && /tmp/patch_cryptofeed.sh"
```

### Config Already Applied
Both Host 75 and Host 120 already have `cryptofeed_api` enabled in their config files:
```json
"cryptofeed_api": {
    "enabled": true,
    "url": "http://192.168.3.73:8088"
}
```

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Cryptofeed Data Flow                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Binance ──WebSocket──► Cryptofeed Service ──► InfluxDB              │
│                            (Host 73)           (Host 6)              │
│                                                    │                 │
│                                                    ▼                 │
│                                          Cryptofeed API Server       │
│                                            (Host 73:8088)            │
│                                                    │                 │
│                    ┌──────────────────────────────┴──────────┐       │
│                    ▼                                          ▼      │
│            Sub-feeders 75a,b,c                       Sub-feeders     │
│              (Host 75)                              120a,b,c (120)   │
│                                                                      │
│  Benefits:                                                           │
│  - No direct Binance API calls for OHLCV                            │
│  - Eliminates IP ban issues                                          │
│  - Faster data access (local network)                               │
│  - Shared data across all feeders                                   │
└─────────────────────────────────────────────────────────────────────┘
```

## Files Modified/Created
- `/tmp/patch_cryptofeed.sh` - Patch script for Host 75/120 exchange.py
- Config files with `cryptofeed_api` enabled on both hosts

## Current Status
| Host | Service | Status | Cryptofeed |
|------|---------|--------|------------|
| 75 | feeder75 | Disabled | N/A |
| 75 | feeder75a,b,c | Running | ❌ Needs patch |
| 120 | feeder120 | Disabled | N/A |
| 120 | feeder120a,b,c | Running | ❌ Needs patch |
| 73 | Cryptofeed API | Running | ✅ Source |
