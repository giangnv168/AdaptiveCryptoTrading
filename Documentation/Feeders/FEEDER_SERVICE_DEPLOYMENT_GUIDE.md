# 10 Feeder Service Deployment Guide
**Date:** 2025-11-23  
**Status:** Ready for Deployment

---

## Overview

This guide provides step-by-step instructions for converting all 10 FreqTrade feeders from manual processes to systemd services. This conversion ensures:

- ✅ Automatic startup on boot
- ✅ Automatic restart on failure
- ✅ Protection from terminal disconnection
- ✅ Centralized logging via journalctl
- ✅ Easy management (start/stop/restart/status)

---

## Current Status

### Running Feeders (5/10) - URGENT Priority
These feeders have been running as manual processes for **11+ days** without systemd protection:

| Feeder | IP | Strategy | Status | Memory |
|--------|-----|----------|--------|--------|
| 74 | 192.168.3.74 | LONG_74_MOMENTUM | ✅ Running (11d) | 12.6 GB |
| 79 | 192.168.3.79 | LONG_79_VOLATILITY | ✅ Running (11d) | 12.8 GB |
| 120 | 192.168.3.120 | SHORT-PRODUCER | ✅ Running (11d) | 49 GB ⚠️ |
| 121 | 192.168.3.121 | SHORT_121_MEANREV | ✅ Running (11d) | 10.8 GB |
| 124 | 192.168.3.124 | SHORT_124_MTF | ✅ Running (11d) | 13.5 GB |

### Stopped Feeders (5/10) - Medium Priority
These feeders are currently not running and need investigation:

| Feeder | IP | Status | Action Needed |
|--------|-----|--------|---------------|
| 73 | 192.168.3.73 | ❌ Stopped | Investigate config & strategy |
| 75 | 192.168.3.75 | ❌ Stopped | Investigate config & strategy |
| 108 | 192.168.3.108 | ❌ Stopped | Investigate config & strategy |
| 122 | 192.168.3.122 | ❌ Stopped | Investigate config & strategy |
| 123 | 192.168.3.123 | ❌ Stopped | Investigate config & strategy |

---

## Files Created

All necessary files have been created in `/home/ubuntu/freqtrade/`:

### Service Files (5 Running Feeders)
```
✓ freqtrade-feeder74.service   - LONG-MOMENTUM
✓ freqtrade-feeder79.service   - LONG-VOLATILITY  
✓ freqtrade-feeder120.service  - SHORT-PRODUCER (60GB memory limit)
✓ freqtrade-feeder121.service  - SHORT-MEANREV
✓ freqtrade-feeder124.service  - SHORT-MTF
```

### Deployment Scripts
```
✓ deploy_feeder_services.sh      - Deploy all 5 services to feeders
✓ migrate_feeder_to_service.sh   - Migrate individual running process to service
✓ investigate_stopped_feeders.sh - Investigate stopped feeders
✓ check_all_feeders.sh           - Check status of all feeders (existing)
```

---

## Phase 1: Deploy Services for Running Feeders

### Step 1: Deploy Service Files

Run the deployment script to install services on all 5 running feeders:

```bash
cd /home/ubuntu/freqtrade
./deploy_feeder_services.sh
```

This script will:
1. Copy service files to each feeder's `/tmp/` directory
2. Move them to `/etc/systemd/system/`
3. Reload systemd daemon
4. Enable services for boot-start

**Expected Output:**
```
✓ Service file copied successfully
✓ Service installed successfully
✓ Systemd daemon reloaded
✓ Service enabled for boot-start
✓ Feeder XX deployment complete!
```

### Step 2: Verify Service Installation

On each feeder, verify the service is installed:

```bash
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.74 \
  "systemctl list-unit-files | grep freqtrade-feeder"
```

Should show:
```
freqtrade-feeder74.service    enabled
```

### Step 3: Migrate Running Processes

**⚠️ IMPORTANT:** This will briefly stop the manual process and start the service.

Migrate each feeder one at a time:

```bash
# Feeder 74
./migrate_feeder_to_service.sh 74

# Wait for confirmation, then proceed to next
./migrate_feeder_to_service.sh 79
./migrate_feeder_to_service.sh 120
./migrate_feeder_to_service.sh 121
./migrate_feeder_to_service.sh 124
```

The migration script will:
1. Find the running manual process
2. Verify service is installed and enabled
3. Gracefully stop the manual process (SIGTERM)
4. Wait for clean exit (max 30 seconds)
5. Start the systemd service
6. Verify service is running

**Expected Output:**
```
✓ Found process PID: XXXXX
✓ Service is enabled
✓ SIGTERM sent to PID XXXXX
✓ Process exited cleanly after X seconds
✓ Service started
✓ Service is active and running
✓ Migration Complete!
```

### Step 4: Verify All Services Running

Check status of all migrated services:

```bash
# Quick check all feeders
./check_all_feeders.sh

# Detailed check individual feeders
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.74 \
  "echo 'NewHopes@168' | sudo -S systemctl status freqtrade-feeder74.service"
```

---

## Phase 2: Handle Feeder 120 Memory Issue

Feeder 120 is using **49 GB** memory (4x normal). Before final migration, investigate:

```bash
# SSH to feeder 120
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.120

# Check FreqAI model directory size
cd /home/freqai/freqtrade
du -sh user_data/models/*

# Check if there are old model files
ls -lh user_data/models/ | head -20

# Check memory usage details
ps aux --sort=-%mem | head -10
```

**Possible Actions:**
1. **Clean old models:** Remove old FreqAI model files
2. **Restart fresh:** Stop process, clear models, start service
3. **Accept as normal:** If this feeder needs more memory for its strategy

---

## Phase 3: Start Stopped Feeders

### Step 1: Investigate Stopped Feeders

Run investigation script:

```bash
./investigate_stopped_feeders.sh > stopped_feeders_investigation.txt
cat stopped_feeders_investigation.txt
```

This will check each stopped feeder for:
- Available config files
- Strategy files
- Bash history
- Startup scripts

### Step 2: Determine Configuration

Based on the investigation, determine for each feeder:
1. **Strategy name:** e.g., `ArtiSkyTrader_Feeder_LONG_73_TREND`
2. **Config file:** e.g., `config_freqai_LightGBM_LONG_73.json`
3. **FreqAI model:** e.g., `LightGBMClassifier_Optuna`

### Step 3: Create Service Files

Based on findings, create service files for stopped feeders. Example for Feeder 73:

```ini
[Unit]
Description=FreqTrade Feeder 73 (LONG-TREND)
After=network.target

[Service]
Type=simple
User=freqai
Group=freqai
WorkingDirectory=/home/freqai/freqtrade
Environment="PATH=/home/freqai/freqtrade/.env/bin:/usr/bin"

ExecStart=/home/freqai/freqtrade/.env/bin/freqtrade trade \
    --dry-run \
    --strategy ArtiSkyTrader_Feeder_LONG_73_TREND \
    --config config_freqai_LightGBM_LONG_73.json \
    --freqaimodel LightGBMClassifier_Optuna \
    --logfile feeder73.log

Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
MemoryMax=16G

[Install]
WantedBy=multi-user.target
```

### Step 4: Deploy and Start

For each stopped feeder:

```bash
# 1. Copy service file
sshpass -p 'NewHopes@168' scp freqtrade-feeder73.service \
  freqai@192.168.3.73:/tmp/

# 2. Install service
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.73 \
  "echo 'NewHopes@168' | sudo -S mv /tmp/freqtrade-feeder73.service /etc/systemd/system/ && \
   sudo systemctl daemon-reload && \
   sudo systemctl enable freqtrade-feeder73.service && \
   sudo systemctl start freqtrade-feeder73.service"

# 3. Check status
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.73 \
  "echo 'NewHopes@168' | sudo -S systemctl status freqtrade-feeder73.service"

# 4. Monitor logs
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.73 \
  "echo 'NewHopes@168' | sudo -S journalctl -u freqtrade-feeder73.service -f"
```

---

## Service Management Commands

### Check Service Status
```bash
# On feeder (as freqai user)
sudo systemctl status freqtrade-feeder74.service

# From BOT3
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.74 \
  "echo 'NewHopes@168' | sudo -S systemctl status freqtrade-feeder74.service"
```

### View Logs
```bash
# Recent logs
sudo journalctl -u freqtrade-feeder74.service -n 100 --no-pager

# Follow logs (real-time)
sudo journalctl -u freqtrade-feeder74.service -f

# Logs since specific time
sudo journalctl -u freqtrade-feeder74.service --since "1 hour ago"
```

### Restart Service
```bash
sudo systemctl restart freqtrade-feeder74.service
```

### Stop Service
```bash
sudo systemctl stop freqtrade-feeder74.service
```

### Start Service
```bash
sudo systemctl start freqtrade-feeder74.service
```

### Disable Service (prevent boot-start)
```bash
sudo systemctl disable freqtrade-feeder74.service
```

---

## Verification Checklist

After deployment, verify:

### For Each Feeder:
- [ ] Service file exists: `/etc/systemd/system/freqtrade-feeder{XX}.service`
- [ ] Service is enabled: `systemctl is-enabled freqtrade-feeder{XX}.service`
- [ ] Service is active: `systemctl is-active freqtrade-feeder{XX}.service`
- [ ] Process is running: `ps aux | grep freqtrade`
- [ ] Log file is being written: `ls -lh feeder{XX}.log`
- [ ] Memory usage is reasonable: `<20 GB` (except feeder 120)

### System-Wide:
- [ ] All 10 feeders have systemd services
- [ ] All 10 feeders are enabled for boot-start
- [ ] All 10 feeders are currently running
- [ ] Aggregator (192.168.3.80) receiving signals from all 10
- [ ] BOT1 and BOT2 receiving ensemble signals

---

## Troubleshooting

### Service Won't Start

Check logs:
```bash
sudo journalctl -u freqtrade-feeder74.service -n 50 --no-pager
```

Common issues:
1. **Config file not found:** Check config path in service file
2. **Strategy not found:** Check strategy name and file exists
3. **Python environment:** Verify `.env/bin/freqtrade` path is correct
4. **Permissions:** Ensure `freqai` user owns `/home/freqai/freqtrade`

### Service Keeps Restarting

Check if error in strategy or config:
```bash
# View logs
sudo journalctl -u freqtrade-feeder74.service -f

# Test command manually
cd /home/freqai/freqtrade
.env/bin/freqtrade trade --dry-run \
  --strategy ArtiSkyTrader_Feeder_LONG_74_MOMENTUM \
  --config config_freqai_LightGBM_LONG_74.json \
  --freqaimodel LightGBMClassifier_Optuna
```

### High Memory Usage

Monitor memory:
```bash
watch -n 5 'ps aux --sort=-%mem | head -10'
```

If memory keeps growing:
1. Check for memory leaks in strategy
2. Clear old FreqAI model files
3. Increase `MemoryMax` in service file
4. Restart service regularly via timer

---

## Next Steps After Deployment

### 1. Monitor for 24 Hours
- Check logs for errors
- Monitor memory usage
- Verify signals reaching Aggregator
- Confirm BOT1/BOT2 receiving ensemble votes

### 2. Test Restart Behavior
```bash
# Test service restart
sudo systemctl restart freqtrade-feeder74.service

# Test server reboot (if safe)
sudo reboot

# After reboot, verify all services started
systemctl list-units --type=service --state=running | grep freqtrade
```

### 3. Optimize Configuration
- Adjust memory limits if needed
- Fine-tune restart delays
- Add watchdog timers if desired

### 4. Document Specializations
Once all feeders running, document:
- What market condition each specializes in
- How they combine in ensemble
- Performance metrics per feeder

---

## Emergency Rollback

If something goes wrong during migration:

### For Single Feeder:
```bash
# Stop service
sudo systemctl stop freqtrade-feeder74.service

# Start manually as before
cd /home/freqai/freqtrade
.env/bin/freqtrade trade --dry-run \
  --strategy ArtiSkyTrader_Feeder_LONG_74_MOMENTUM \
  --config config_freqai_LightGBM_LONG_74.json \
  --freqaimodel LightGBMClassifier_Optuna \
  --logfile dryrun.log
```

### Disable Service:
```bash
sudo systemctl disable freqtrade-feeder74.service
sudo systemctl stop freqtrade-feeder74.service
```

---

## Success Criteria

Deployment is successful when:

✅ **All 10 feeders running as systemd services**  
✅ **All services enabled for boot-start**  
✅ **No errors in journalctl logs**  
✅ **Memory usage stable (<20 GB per feeder)**  
✅ **Aggregator receiving signals from all 10 feeders**  
✅ **BOT1/BOT2 executing trades based on ensemble votes**  
✅ **Services survive server reboot**  
✅ **Services auto-restart on failure**  

---

## Reference Information

### Credentials
```
Feeders: freqai / NewHopes@168
BOT1-3:  ubuntu / ArtiSky@7
```

### IP Addresses
```
Feeders: 192.168.3.73, 74, 75, 79, 108, 120, 121, 122, 123, 124
Aggregator: 192.168.3.80
InfluxDB: 192.168.3.6
BOT1: 192.168.3.71 (LONG)
BOT2: 192.168.3.72 (SHORT)
BOT3: 192.168.3.33 (Meta-learner)
```

### Architecture
```
10 Feeders → Aggregator (Ensemble voting) → BOT1/BOT2 → InfluxDB → BOT3
```

---

**Document Version:** 1.0  
**Last Updated:** 2025-11-23  
**Author:** Deployment automation for 10-feeder FreqTrade system
