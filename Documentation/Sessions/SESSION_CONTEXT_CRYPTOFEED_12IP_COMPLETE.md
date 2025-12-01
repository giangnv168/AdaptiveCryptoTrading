# Session Context: Cryptofeed 12 IP Integration Complete
**Date:** November 24, 2025

## Summary
Successfully deployed Cryptofeed V2 with 12 IP support using NAT port routing. Two workers are now running, each using 6 source ports that are NAT-routed to different public IPs.

## Completed Tasks

### 1. Fixed Python Path Issue
- **Problem:** Service files were using `/home/freqai/freqtrade/.venv/bin/python` which doesn't exist on feeder hosts
- **Solution:** Changed to `/usr/bin/python3`
- **Error Code:** 203/EXEC (executable not found)

### 2. Deployed Cryptofeed V2 Workers
- **Worker 0 (Host 73):** ACTIVE ✓
  - Service: `cryptofeed-12ip-w0`
  - Source ports: 8081,8082,8083,8084,8085,8086
  - Worker ID: 0 / Total Workers: 2
  
- **Worker 1 (Host 108):** ACTIVE ✓
  - Service: `cryptofeed-12ip-w1`
  - Source ports: 8081,8082,8083,8084,8085,8086
  - Worker ID: 1 / Total Workers: 2

### 3. Feeder Integration Status
All feeders have Cryptofeed-enabled code (5 cryptofeed_api references):
- Host 73, 74, 79, 108: ✓ Main feeders
- Host 75, 120, 121, 122, 123, 124: ✓ Sub-feeders

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CRYPTOFEED V2 WITH 12 IPS                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Host 73 (Worker 0)              Host 108 (Worker 1)        │
│  ├─ Port 8081 → IP1              ├─ Port 8081 → IP7         │
│  ├─ Port 8082 → IP2              ├─ Port 8082 → IP8         │
│  ├─ Port 8083 → IP3              ├─ Port 8083 → IP9         │
│  ├─ Port 8084 → IP4              ├─ Port 8084 → IP10        │
│  ├─ Port 8085 → IP5              ├─ Port 8085 → IP11        │
│  └─ Port 8086 → IP6              └─ Port 8086 → IP12        │
│                                                             │
│  → 81-82 coins each              → 81-82 coins each         │
│    (worker_id 0)                   (worker_id 1)            │
│                                                             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │   InfluxDB     │
              │ (cryptofeed db)│
              └────────────────┘
                       │
                       ▼
              ┌────────────────────┐
              │    10 Feeders      │
              │ (cryptofeed_api)   │
              │ Query funding rate │
              │ from InfluxDB      │
              └────────────────────┘
```

## Service Files

### Worker 0 (Host 73)
```ini
[Unit]
Description=Cryptofeed InfluxDB Writer V2 - Worker 0 (Host 73 with 6 NAT IPs)
After=network.target

[Service]
Type=simple
User=freqai
WorkingDirectory=/home/freqai/freqtrade
ExecStart=/usr/bin/python3 /home/freqai/freqtrade/user_data/services/cryptofeed_influxdb_writer_v2.py --source-ports 8081,8082,8083,8084,8085,8086 --worker-id 0 --total-workers 2 --history-limit 1500
Restart=always
RestartSec=60
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

### Worker 1 (Host 108)
```ini
[Unit]
Description=Cryptofeed InfluxDB Writer V2 - Worker 1 (Host 108 with 6 NAT IPs)
After=network.target

[Service]
Type=simple
User=freqai
WorkingDirectory=/home/freqai/freqtrade
ExecStart=/usr/bin/python3 /home/freqai/freqtrade/user_data/services/cryptofeed_influxdb_writer_v2.py --source-ports 8081,8082,8083,8084,8085,8086 --worker-id 1 --total-workers 2 --history-limit 1500
Restart=always
RestartSec=60
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

## Management Commands

```bash
# Check worker status
sshpass -p "NewHopes@168" ssh freqai@192.168.3.73 "systemctl status cryptofeed-12ip-w0"
sshpass -p "NewHopes@168" ssh freqai@192.168.3.108 "systemctl status cryptofeed-12ip-w1"

# View logs
sshpass -p "NewHopes@168" ssh freqai@192.168.3.73 "journalctl -u cryptofeed-12ip-w0 -f"
sshpass -p "NewHopes@168" ssh freqai@192.168.3.108 "journalctl -u cryptofeed-12ip-w1 -f"

# Restart workers
sshpass -p "NewHopes@168" ssh -t freqai@192.168.3.73 "echo 'NewHopes@168' | sudo -S systemctl restart cryptofeed-12ip-w0"
sshpass -p "NewHopes@168" ssh -t freqai@192.168.3.108 "echo 'NewHopes@168' | sudo -S systemctl restart cryptofeed-12ip-w1"
```

## Next Steps

1. **Monitor for a few hours** to ensure stable operation
2. **Verify data in InfluxDB** - Check that OHLCV and funding data is being written
3. **Test feeder integration** - Confirm feeders can query funding rates from InfluxDB
4. **Enable on boot** (if not already):
   ```bash
   sshpass -p "NewHopes@168" ssh -t freqai@192.168.3.73 "echo 'NewHopes@168' | sudo -S systemctl enable cryptofeed-12ip-w0"
   sshpass -p "NewHopes@168" ssh -t freqai@192.168.3.108 "echo 'NewHopes@168' | sudo -S systemctl enable cryptofeed-12ip-w1"
   ```

## Issues Resolved
- ✓ Python venv path issue (203/EXEC error)
- ✓ Service deployment to both hosts
- ✓ V2 script copied to Host 108

## Files Created/Modified
- `/tmp/cryptofeed-12ip-w0.service` → deployed to Host 73
- `/tmp/cryptofeed-12ip-w1.service` → deployed to Host 108
- `user_data/services/cryptofeed_influxdb_writer_v2.py` → copied to Host 108
