# Create BOT3 Optuna Service Following BOT2 Pattern

## CONTEXT

BOT2 Optuna optimizer was successfully deployed with:
- ✅ Real profit calculations (0.3-0.5% per trial)
- ✅ 12 parallel threads (n_jobs=12)
- ✅ ~30 minute optimization for 60 pairs
- ✅ All critical bugs pre-fixed

Now create BOT3 Optuna service following the EXACT SAME pattern to avoid all the bugs we fixed in BOT2.

---

## BOT3 REQUIREMENTS

**Determine first:**
- [ ] BOT3 server location (192.168.3.XX)
- [ ] BOT3 trading direction (LONG, SHORT, or hybrid)
- [ ] BOT3 strategy name
- [ ] BOT3 pairlist (use same 60 pairs from feeder120a/b/c?)
- [ ] BOT3 InfluxDB bucket name

---

## STEP-BY-STEP IMPLEMENTATION

### Step 1: Prepare Environment

```bash
# SSH to BOT3 server
ssh ubuntu@192.168.3.XX

# Navigate to freqtrade directory
cd /home/ubuntu/freqtrade

# Verify dependencies
python3 -c "import optuna; print('✅ Optuna installed')"
python3 -c "import influxdb_client; print('✅ InfluxDB client installed')"
```

### Step 2: Create Optimizer File

**IMPORTANT: Apply ALL BOT2 lessons learned from the start!**

```bash
# Copy the working BOT2 optimizer as template
cp user_data/services/optuna_optimizer_per_bot_standalone.py \
   user_data/services/optuna_optimizer_bot3_standalone.py
```

#### Critical Fixes to Apply Immediately

**Fix #1: open_date Field (Line ~375)**
```bash
# Verify this line uses record.get_time() NOT record.values.get()
sed -i "s/'open_date': record.values.get('open_date'),/'open_date': record.get_time(),/" \
  user_data/services/optuna_optimizer_bot3_standalone.py

# Verify the fix
grep -n "open_date.*record" user_data/services/optuna_optimizer_bot3_standalone.py
```

Expected output:
```
375:                        'open_date': record.get_time(),
```

**Fix #2: Parallel Processing (Line ~288)**
```bash
# Ensure n_jobs=12 is set
sed -i 's/show_progress_bar=False)/show_progress_bar=False, n_jobs=12)/' \
  user_data/services/optuna_optimizer_bot3_standalone.py

# Verify
grep -n "study.optimize.*n_jobs" user_data/services/optuna_optimizer_bot3_standalone.py
```

Expected output:
```
288:        study.optimize(objective, n_trials=n_trials, show_progress_bar=False, n_jobs=12)
```

**Fix #3: Update Bot Configuration**

Edit the optimizer file to change BOT2 references to BOT3:

```bash
# Update target_bot parameter
sed -i 's/target_bot = "BOT2"/target_bot = "BOT3"/' \
  user_data/services/optuna_optimizer_bot3_standalone.py

# Update InfluxDB bucket
sed -i 's/influx_bucket_trades = "BOT2_trades"/influx_bucket_trades = "BOT3_trades"/' \
  user_data/services/optuna_optimizer_bot3_standalone.py

# Verify changes
grep -n "target_bot\|influx_bucket" user_data/services/optuna_optimizer_bot3_standalone.py
```

### Step 3: Create Service Runner

```bash
cp user_data/services/run_optuna_per_bot_standalone.py \
   user_data/services/run_optuna_bot3_standalone.py

# Update to use BOT3 optimizer
sed -i 's/optuna_optimizer_per_bot_standalone/optuna_optimizer_bot3_standalone/' \
  user_data/services/run_optuna_bot3_standalone.py
```

### Step 4: Create Systemd Service

```bash
sudo nano /etc/systemd/system/optuna-bot3-standalone.service
```

**Service file content:**
```ini
[Unit]
Description=Optuna Parameter Optimizer for BOT3 (Standalone)
After=network.target influxdb.service

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/freqtrade
Environment="PATH=/home/ubuntu/freqtrade/.venv/bin:/usr/local/bin:/usr/bin:/bin"

ExecStart=/home/ubuntu/freqtrade/.venv/bin/python3 \
    /home/ubuntu/freqtrade/user_data/services/run_optuna_bot3_standalone.py

# Resource limits (same as BOT2)
CPUQuota=30%
MemoryLimit=19G

# Logging
StandardOutput=append:/home/ubuntu/freqtrade/user_data/logs/optuna_optimizer_BOT3.log
StandardError=append:/home/ubuntu/freqtrade/user_data/logs/optuna_optimizer_BOT3.log

# Auto-restart
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Step 5: Syntax Validation

```bash
# Validate optimizer syntax
python3 -m py_compile user_data/services/optuna_optimizer_bot3_standalone.py
echo "✅ Optimizer syntax valid"

# Validate runner syntax  
python3 -m py_compile user_data/services/run_optuna_bot3_standalone.py
echo "✅ Runner syntax valid"
```

### Step 6: Enable and Start Service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable service (start on boot)
sudo systemctl enable optuna-bot3-standalone

# Start service
sudo systemctl start optuna-bot3-standalone

# Check status
systemctl status optuna-bot3-standalone
```

### Step 7: Monitor Logs

```bash
# Follow logs in real-time
tail -f /home/ubuntu/freqtrade/user_data/logs/optuna_optimizer_BOT3.log
```

---

## VERIFICATION CHECKLIST

### Immediate Checks (First 2 minutes)

- [ ] Service status: `systemctl is-active optuna-bot3-standalone` → active
- [ ] No errors in logs
- [ ] Log shows: "🎯 Optimizing [PAIR] (1/60)"
- [ ] Log shows: "✅ Loaded XXXXX candles for [PAIR]"
- [ ] Log shows: "📊 Loaded XXX BOT3 trades for [PAIR]"

### After 5-10 Trials

- [ ] Trial profits are NON-ZERO (0.001-0.005 range)
- [ ] Trials complete out of order (proof of parallel)
- [ ] Multiple trials finish within same second
- [ ] No "value: 0.0" in trial results

### After 30+ Trials

- [ ] Best trial shows positive profit
- [ ] ETA shows ~1-2 minutes per pair
- [ ] CPU usage visible (30% quota being used)
- [ ] Memory usage stable (<19G)

---

## EXPECTED LOG OUTPUT

**Good (Correct Setup):**
```
2025-11-28 15:00:00 - INFO - 🎯 Optimizing BTC/USDT:USDT (1/60)
2025-11-28 15:00:01 - INFO -    ✅ Loaded 25977 candles for BTC/USDT:USDT
2025-11-28 15:00:02 - INFO -    📊 Loaded 450 BOT3 trades for BTC/USDT:USDT
[I 2025-11-28 15:00:15] Trial 22 finished with value: 0.003045...
[I 2025-11-28 15:00:15] Trial 21 finished with value: 0.002156...  ← Out of order!
[I 2025-11-28 15:00:16] Trial 24 finished with value: 0.003892...  ← Parallel!
```

**Bad (Has BOT2's Original Bugs):**
```
[I 2025-11-28 15:00:15] Trial 1 finished with value: 0.0  ← All zeros!
[I 2025-11-28 15:00:17] Trial 2 finished with value: 0.0
[I 2025-11-28 15:00:19] Trial 3 finished with value: 0.0
```

---

## TROUBLESHOOTING

### If Profits Are All 0.0

**Check Line 375:**
```bash
grep -n "'open_date'" user_data/services/optuna_optimizer_bot3_standalone.py
```

Must show:
```
375:                        'open_date': record.get_time(),
```

If it shows `record.values.get('open_date')`, that's the bug!

### If Trials Are Sequential (Not Parallel)

**Check Line 288:**
```bash
grep -n "n_jobs" user_data/services/optuna_optimizer_bot3_standalone.py
```

Must show:
```
288:        study.optimize(objective, n_trials=n_trials, show_progress_bar=False, n_jobs=12)
```

### If Service Fails to Start

1. Check syntax: `python3 -m py_compile [file]`
2. Check logs: `journalctl -u optuna-bot3-standalone -n 50`
3. Check permissions: `ls -la user_data/services/`

### If No Trades Found

- Verify BOT3_trades bucket exists in InfluxDB
- Check bucket name in optimizer file
- Verify BOT3 has closed trades in InfluxDB

---

## BOT2 LESSONS CHECKLIST

Before starting BOT3, verify you've applied:

- [ ] ✅ Use `record.get_time()` for open_date (Line 375)
- [ ] ✅ Add `n_jobs=12` for parallel processing (Line 288)
- [ ] ✅ Proper error handling for None values
- [ ] ✅ Correct InfluxDB bucket name
- [ ] ✅ Correct target_bot name
- [ ] ✅ Syntax validation before deployment
- [ ] ✅ Service properly configured (systemd)
- [ ] ✅ Logs directory exists and writable
- [ ] ✅ Resource limits set (CPU 30%, Memory 19G)

---

## SUCCESS CRITERIA

BOT3 Optuna is successfully deployed when:

✅ **Service:** Running without errors  
✅ **Profits:** 0.1%-0.5% per trial (NOT 0.0)  
✅ **Parallel:** Trials complete out of order  
✅ **Speed:** ~30 minutes for 60 pairs  
✅ **Logs:** Show candle loading and trade loading  
✅ **Performance:** Same as BOT2 (proven pattern)

---

## MONITORING COMMANDS

```bash
# Service status
systemctl status optuna-bot3-standalone

# Follow logs
tail -f /home/ubuntu/freqtrade/user_data/logs/optuna_optimizer_BOT3.log

# Check for errors
journalctl -u optuna-bot3-standalone -p err

# Resource usage
top -u ubuntu | grep python

# Check parallel execution
tail -50 logs/optuna_optimizer_BOT3.log | grep "Trial.*finished"
```

---

## FINAL NOTES

**This setup incorporates ALL lessons learned from BOT2:**
- Pre-fixed the open_date bug
- Pre-enabled parallel processing
- Pre-configured service properly
- Pre-validated syntax
- Pre-set resource limits

**Expected result:**  
BOT3 should work perfectly on first try, with no debugging needed!

**Time to deploy:**  
~15 minutes (vs BOT2's hours of debugging)

**Reference files:**  
- BOT2 optimizer: Proven working version
- BOT2 service: Proven working configuration
- This guide: All lessons learned applied

---

Start implementation and BOT3 should be optimizing parameters in ~30 minutes with real profits and 12x speedup!
