# BOT1 Optuna Deployment - Manual Instructions

**Date:** 2025-11-28  
**Status:** READY FOR MANUAL DEPLOYMENT  

---

## 🎯 SITUATION

BOT1 doesn't have the standalone Optuna optimizer installed yet. We need to deploy the complete fixed version from scratch.

## 📦 FILES READY FOR DEPLOYMENT

On the current server (192.168.3.33), these files are ready:

1. **Fixed Optimizer File:**
   - Path: `/tmp/optuna_optimizer_per_bot_standalone_fixed.py`
   - Size: ~31KB
   - Fixes Applied: ✅ open_date mapping, ✅ n_jobs=12

2. **Run Script:**
   - Path: `/home/ubuntu/freqtrade/user_data/services/run_optuna_per_bot_standalone.py`

---

## 🚀 DEPLOYMENT STEPS

### Option 1: Use SCP from Current Server (Recommended)

**Step 1: From your local machine or current server, copy files to BOT1:**

```bash
# Copy the fixed optimizer file
scp /tmp/optuna_optimizer_per_bot_standalone_fixed.py ubuntu@192.168.3.71:/home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py

# Copy the run script
scp /home/ubuntu/freqtrade/user_data/services/run_optuna_per_bot_standalone.py ubuntu@192.168.3.71:/home/ubuntu/freqtrade/user_data/services/
```

**Step 2: SSH to BOT1 and create the service file:**

```bash
ssh ubuntu@192.168.3.71
```

Then run:

```bash
sudo tee /etc/systemd/system/optuna-bot1-standalone.service > /dev/null <<EOF
[Unit]
Description=BOT1 Optuna Parameter Optimizer
After=network.target influxdb.service

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/freqtrade
Environment="PATH=/home/ubuntu/freqtrade/.venv/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=/home/ubuntu/freqtrade/.venv/bin/python3 /home/ubuntu/freqtrade/user_data/services/run_optuna_per_bot_standalone.py --bot BOT1
Restart=always
RestartSec=60
StandardOutput=journal
StandardError=journal
SyslogIdentifier=optuna-bot1

[Install]
WantedBy=multi-user.target
EOF
```

**Step 3: Enable and start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable optuna-bot1-standalone
sudo systemctl start optuna-bot1-standalone
```

**Step 4: Verify it's running:**

```bash
sudo systemctl status optuna-bot1-standalone
sudo journalctl -u optuna-bot1-standalone -n 50 --no-pager
```

---

### Option 2: Manual File Copy (Alternative)

If SCP doesn't work, you can manually copy the file content:

**Step 1: On current server, display the file:**
```bash
cat /tmp/optuna_optimizer_per_bot_standalone_fixed.py
```

**Step 2: On BOT1, create the file:**
```bash
ssh ubuntu@192.168.3.71
nano /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py
# Paste the content and save (Ctrl+X, Y, Enter)
```

**Step 3: Continue with service creation (same as Option 1, Step 2-4)**

---

## ✅ SUCCESS INDICATORS

After starting the service, you should see in the logs:

```
✅ InfluxDB connected
🔄 Starting optimization for XX pairs
🎯 Optimizing [PAIR] (1/60)
✅ Loaded XXXXX candles for [PAIR]
📊 [PAIR]: XX BOT1 trades (train: XX, test: XX)
Trial 1/100: params=[SL=-1.5%, TP=8.0%, VM=1.2] profit=0.32%
```

**Key indicators of success:**
- ✅ Service status: `active (running)`
- ✅ Real profit values: `0.32%`, `0.45%` (NOT `0.00%`)
- ✅ OHLCV loading: `✅ Loaded XXXXX candles`
- ✅ BOT1 trade data: `XX BOT1 trades`
- ✅ Multiple trials running in parallel

---

## 🔍 MONITORING

Watch the logs in real-time:
```bash
sudo journalctl -u optuna-bot1-standalone -f
```

---

## 🚨 TROUBLESHOOTING

**If service fails to start:**
```bash
# Check logs for errors
sudo journalctl -u optuna-bot1-standalone -n 100 --no-pager

# Check file exists
ls -lh /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py

# Validate Python syntax
cd /home/ubuntu/freqtrade
source .venv/bin/activate
python3 -m py_compile user_data/services/optuna_optimizer_per_bot_standalone.py
```

**If still showing 0% profits:**
```bash
# Verify Fix #1 was applied
grep "record.get_time()" /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py
# Should show: 'open_date': record.get_time(),
```

**If optimization is slow:**
```bash
# Verify Fix #2 was applied
grep "n_jobs=12" /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py
# Should show: study.optimize(objective, n_trials=n_trials, n_jobs=12, show_progress_bar=False)
```

---

## 📊 EXPECTED PERFORMANCE

After successful deployment:
- ✅ Profit calculations: 0.3-0.5% per trial (matching BOT2)
- ✅ Parallel processing: 12 threads
- ✅ Optimization time: ~17-30 minutes for 60 pairs
- ✅ Service runs continuously, re-optimizing every 6 hours

---

## 📞 SUMMARY

1. Copy files to BOT1 using SCP
2. Create service file on BOT1
3. Enable and start service
4. Monitor logs for success indicators

**All fixes are already applied to the file - just deploy it!**

✅ Fix #1: open_date = record.get_time()  
✅ Fix #2: n_jobs=12 parallel threads
