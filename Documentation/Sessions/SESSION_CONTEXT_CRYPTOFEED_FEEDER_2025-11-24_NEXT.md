# Session Context - CryptoFeed Feeder Fix 2025-11-24 (Updated)

## Session Date
2025-11-24 16:27 (Vietnam Time)

## Summary
Successfully fixed CryptoFeed API warnings on Host 75 feeders by disabling CryptoFeed API integration.

## Issues Fixed This Session

### 1. CCXT Version Incompatibility (Host 75)
- **Problem**: Feeders crashed with "Freqtrade does not support isolated futures on Binance"
- **Cause**: CCXT 4.3.50 missing `features` attribute in Binance exchange object
- **Solution**: Upgraded CCXT from 4.3.50 to 4.5.20
- **Status**: ✅ FIXED

### 2. CryptoFeed API Warnings
- **Problem**: Feeders showing warnings "Cryptofeed API failed for ONDO/USDT:USDT 3m"
- **Cause**: CryptoFeed only has 163 pairs, feeders need more pairs not in subscription
- **Solution**: Disabled CryptoFeed API in feeder configs by setting `"cryptofeed_api": {"enabled": false}`
- **Status**: ✅ FIXED

### 3. JSON Config Corruption
- **Problem**: Initial sed command corrupted JSON config files
- **Solution**: Fixed with proper sed pattern to consolidate multi-line JSON block
- **Status**: ✅ FIXED

## Current Status

### Host 73 (192.168.3.73)
- **cryptofeed-native-writer**: ✅ Running (collecting data for 163 pairs)
- **cryptofeed-api**: ✅ Running on port 8088 (serving OHLCV data)

### Host 75 (192.168.3.75)
- **freqtrade-feeder75a**: ✅ Active (CryptoFeed disabled, using Binance API)
- **freqtrade-feeder75b**: ✅ Active (CryptoFeed disabled, using Binance API)
- **freqtrade-feeder75c**: ✅ Active (CryptoFeed disabled, using Binance API)
- **CCXT Version**: 4.5.20

## Config Changes Made

### Host 75 Feeder Configs
Files modified:
- `/home/freqai/freqtrade/config_feeder_75a.json`
- `/home/freqai/freqtrade/config_feeder_75b.json`
- `/home/freqai/freqtrade/config_feeder_75c.json`

Changed `cryptofeed_api` from:
```json
"cryptofeed_api": {
    "enabled": true,
    "url": "http://192.168.3.73:8088"
}
```
To:
```json
"cryptofeed_api": {"enabled": false}
```

## Architecture Decision
- **CryptoFeed** remains active on Host 73 collecting real-time tick data for 163 pairs
- **Feeders on Host 75** now use direct Binance API (not CryptoFeed) since they need more pairs
- This prevents retry warnings for pairs not in CryptoFeed subscription

## Next Steps
1. Consider enabling CryptoFeed only for feeders that exclusively use the 163 subscribed pairs
2. Or expand CryptoFeed subscription to include all feeder pairs

## Commands Used

### Upgrade CCXT on Host 75
```bash
/home/freqai/freqtrade/.env/bin/pip install --upgrade ccxt
```

### Fix JSON Config
```bash
for f in config_feeder_75a.json config_feeder_75b.json config_feeder_75c.json; do
  sed -i '/"cryptofeed_api": null,/{N;N;N;s/"cryptofeed_api": null,\n.*enabled.*\n.*url.*\n.*}/"cryptofeed_api": {"enabled": false}/}' /home/freqai/freqtrade/$f
done
```

### Restart Feeders
```bash
sudo systemctl restart freqtrade-feeder75a freqtrade-feeder75b freqtrade-feeder75c
