# BOT1 Optuna Optimization Fix - Deployment Guide

**Date:** 2025-11-28  
**Server:** BOT1 (192.168.3.71)  
**Status:** READY FOR DEPLOYMENT  
**Priority:** HIGH

---

## 🎯 OBJECTIVE

Apply the same successful Optuna fixes from BOT2 to BOT1 to enable:
- ✅ Real profit calculations (0.3-0.5% per trial)
- ✅ 12 parallel threads for faster optimization
- ✅ ~17-30 minute optimization time for 60 pairs
- ✅ Accurate open_date field mapping from InfluxDB

---

## 🐛 ROOT CAUSE

**Fix #1: open_date Field Mapping (Line ~375)**
- **Problem:** `record.values.get('open_date')` returns incorrect/null timestamps
- **Impact:** All profit calculations = 0, optimizer can't work properly
- **Solution:** Change to `record.get_time()` to get correct timestamp from InfluxDB

**Fix #2: Parallel Processing**
- **Problem:** Single-threaded optimization is too slow (2+ hours for 60 pairs)
- **Impact:** Optimization takes too long, parameters become stale
- **Solution:** Add `n_jobs=12` to enable 12 parallel trial threads

---

## 📦 WHAT WILL BE FIXED

### Target File
`/home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py`

### Fix #1: open_date Field Mapping
**Location:** Line ~375 in `_load_bot_trades` method

**Before (BROKEN):**
```python
'open_date': record.values.get('open_date'),
```

**After (FIXED):**
```python
'open_date': record.get_time(),
```

### Fix #2: Parallel Processing
**Location:** Line ~450 in `optimize_pair` method

**Before (SLOW):**
```python
study.optimize(objective, n_trials=n_trials, show_progress_bar=False)
```

**After (FAST):**
```python
study.optimize(objective, n_trials=n_trials, n_jobs=12, show_progress_bar=False)
```

---

## 🚀 DEPLOYMENT STEPS

### Step 1: Connect to BOT1
```bash
ssh ubuntu@192.168.3.71
```

### Step 2: Verify Script is Present
```bash
ls -lh /tmp/apply_bot1_fixes_locally.sh
```

Expected output:
```
-rwxr-xr-x 1 ubuntu ubuntu 4.0K Nov 28 13:59 /tmp/apply_bot1_fixes_locally.sh
```

### Step 3: Run Deployment Script
```bash
sudo bash /tmp/apply_bot1_fixes_locally.sh
```

### Step 4: Monitor the Output
The script will:
1. ✅ Create backup of current file
2. ✅ Apply Fix #1 (open_date mapping)
3. ✅ Apply Fix #2 (n_jobs=12)
4. ✅ Validate Python syntax
5. ✅ Restart optuna-bot1-standalone service
6. ✅ Check service status
7. ✅ Display recent logs
8. ✅ Verify success indicators

---

## ✅ EXPECTED OUTPUT

### Successful Deployment
```
🚀 BOT1 Optuna Optimizer - Fix Deployment (Local)
==================================================

Target: BOT1 (192.168.3.71)
Fixes to apply:
  1. open_date field mapping (record.get_time())
  2. n_jobs=12 for parallel processing

Step 1: Creating backup...
✅ Backup created

Step 2: Applying Fix #1 - open_date field mapping...
✅ Fix #1 applied: record.values.get('open_date') → record.get_time()

Step 3: Applying Fix #2 - n_jobs=12 parallel processing...
✅ Fix #2 applied: n_jobs=12 added to study.optimize()

Step 4: Verifying Python syntax...
✅ Syntax validation passed

Step 5: Restarting optuna-bot1-standalone service...
✅ Service restarted

Step 6: Service Status
----------------------
● optuna-bot1-standalone.service - BOT1 Optuna Parameter Optimizer
   Loaded: loaded (/etc/systemd/system/optuna-bot1-standalone.service)
   Active: active (running) since Thu 2025-11-28 14:00:00 +07

Step 7: Recent Logs (last 30 lines)
------------------------------------
2025-11-28 14:00:00 - INFO - 🚀 BOT1 Optuna Parameter Optimization
2025-11-28 14:00:01 - INFO - ✅ InfluxDB connected
2025-11-28 14:00:02 - INFO - 📊 Optimizing AAVE/USDT:USDT (1/60)
2025-11-28 14:00:03 - INFO -    ✅ Loaded 50000 candles for AAVE/USDT:USDT
2025-11-28 14:00:04 - INFO - 📊 AAVE/USDT:USDT: 45 BOT1 trades (train: 31, test: 14)

Step 8: Checking for SUCCESS indicators...
-------------------------------------------
✅ OHLCV candle loading detected!
✅ Trial optimization detected!
✅ BOT1 mentions found in logs!

==========================================
✅ SUCCESS: Service is running!
==========================================

Fixes applied:
  ✅ Fix #1: open_date = record.get_time()
  ✅ Fix #2: n_jobs=12 parallel threads

Next steps:
  - Monitor logs for 5-10 minutes
  - Check for real profit values (0.3-0.5% per trial)
  - Verify optimization completes in ~17-30 minutes for 60 pairs

Monitor with: sudo journalctl -u optuna-bot1-standalone -f
```

---

## 🔍 VERIFICATION STEPS

### Check Service Status
```bash
sudo systemctl status optuna-bot1-standalone
```

**Expected:** `Active: active (running)`

### Monitor Real-Time Logs
```bash
sudo journalctl -u optuna-bot1-standalone -f
```

### Look for Success Indicators

**✅ OHLCV Loading:**
```
✅ Loaded 50000 candles for AAVE/USDT:USDT
```

**✅ Real Profit Values:**
```
Trial 1/100: params=[SL=-1.5%, TP=8.0%, VM=1.2] profit=0.32%
Trial 2/100: params=[SL=-2.0%, TP=10.0%, VM=1.1] profit=0.45%
```

**✅ Parallel Processing:**
```
Trial 1/100: ... profit=0.32%
Trial 2/100: ... profit=0.28%
Trial 3/100: ... profit=0.51%
(Multiple trials running simultaneously)
```

**✅ BOT1 Trade Loading:**
```
📊 AAVE/USDT:USDT: 45 BOT1 trades (train: 31, test: 14)
```

---

## 🚨 TROUBLESHOOTING

### Issue 1: Service Fails to Start

**Check Logs:**
```bash
sudo journalctl -u optuna-bot1-standalone -n 100 --no-pager
```

**Common Causes:**
- Python syntax error (script validates this before restart)
- InfluxDB connection failure
- Missing dependencies

**Solution:**
```bash
# Rollback to backup
sudo cp /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py.backup.* \
     /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py

# Restart service
sudo systemctl restart optuna-bot1-standalone
```

### Issue 2: Still Showing 0% Profit

**Check for open_date Fix:**
```bash
grep "record.get_time()" /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py
```

**Expected Output:**
```python
'open_date': record.get_time(),
```

**If Not Fixed:**
```bash
# Manually apply fix
sudo sed -i "s/'open_date': record.values.get('open_date'),/'open_date': record.get_time(),/g" \
    /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py

# Restart service
sudo systemctl restart optuna-bot1-standalone
```

### Issue 3: Optimization Still Takes Too Long

**Check for n_jobs Fix:**
```bash
grep "n_jobs=12" /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py
```

**Expected Output:**
```python
study.optimize(objective, n_trials=n_trials, n_jobs=12, show_progress_bar=False)
```

**If Not Fixed:**
```bash
# Manually apply fix
sudo sed -i 's/study.optimize(objective, n_trials=n_trials, show_progress_bar=False)/study.optimize(objective, n_trials=n_trials, n_jobs=12, show_progress_bar=False)/g' \
    /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py

# Restart service
sudo systemctl restart optuna-bot1-standalone
```

---

## 📊 PERFORMANCE COMPARISON

### Before Fixes (BOT1 Current State)
- ❌ Profit calculations: 0% (broken)
- ❌ Parallel processing: No (1 thread)
- ❌ Optimization time: N/A (not working)
- ❌ Service status: Likely failing or producing invalid results

### After Fixes (Expected BOT1 Performance)
- ✅ Profit calculations: 0.3-0.5% per trial (working)
- ✅ Parallel processing: Yes (12 threads)
- ✅ Optimization time: ~17-30 minutes for 60 pairs
- ✅ Service status: Running successfully

### BOT2 Current Performance (Reference)
- ✅ Profit calculations: 0.3-0.5% per trial (working)
- ✅ Parallel processing: Yes (12 threads)
- ✅ Optimization time: ~17-30 minutes for 60 pairs
- ✅ Service status: Running successfully

---

## 📁 FILES REFERENCE

### Deployment Script
- **Local:** `/tmp/apply_bot1_fixes_locally.sh` (on BOT1)
- **Backup:** Automatically created during deployment
- **Backup Pattern:** `optuna_optimizer_per_bot_standalone.py.backup.YYYYMMDD_HHMMSS`

### Target File
- **Path:** `/home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py`
- **Size:** ~31KB
- **Lines:** ~750

### Service
- **Name:** `optuna-bot1-standalone.service`
- **Config:** `/etc/systemd/system/optuna-bot1-standalone.service`
- **Log Command:** `sudo journalctl -u optuna-bot1-standalone -f`

---

## 🎯 SUCCESS CRITERIA

After deployment, BOT1 should show:

1. **Service Running**
   - `systemctl status optuna-bot1-standalone` shows `active (running)`

2. **Real Profit Values**
   - Logs show trial profits like `0.32%`, `0.45%`, NOT `0.00%`

3. **OHLCV Loading**
   - Logs show `✅ Loaded XXXXX candles for [PAIR]`

4. **BOT1 Trade Data**
   - Logs show `XX BOT1 trades` for each pair

5. **Parallel Processing**
   - Multiple trials running simultaneously (visible in logs)

6. **Reasonable Completion Time**
   - 60 pairs complete in ~17-30 minutes

---

## 📞 NEXT STEPS

### After Successful Deployment

1. **Monitor for 10 minutes**
   ```bash
   sudo journalctl -u optuna-bot1-standalone -f
   ```

2. **Verify first pair completes**
   - Should see optimization complete for first pair
   - Should see real profit values (not 0%)

3. **Check InfluxDB for results**
   - Results should be written to `OPTUNA_PARAMS` bucket
   - Tagged with `optimized_for_bot=BOT1`

4. **Compare with BOT2**
   - BOT1 performance should match BOT2
   - Both should show similar profit ranges
   - Both should complete in similar timeframes

### If Deployment Fails

1. **Capture logs**
   ```bash
   sudo journalctl -u optuna-bot1-standalone -n 200 --no-pager > /tmp/bot1_error_logs.txt
   ```

2. **Rollback to backup**
   ```bash
   sudo cp /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py.backup.* \
        /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py
   sudo systemctl restart optuna-bot1-standalone
   ```

3. **Report issue**
   - Share error logs
   - Describe specific symptoms

---

## ✅ DEPLOYMENT CHECKLIST

- [ ] SSH to BOT1 (192.168.3.71)
- [ ] Verify deployment script exists at `/tmp/apply_bot1_fixes_locally.sh`
- [ ] Run deployment script: `sudo bash /tmp/apply_bot1_fixes_locally.sh`
- [ ] Verify service status: `active (running)`
- [ ] Monitor logs for real profit values (not 0%)
- [ ] Confirm OHLCV candle loading in logs
- [ ] Verify BOT1 trade data loading in logs
- [ ] Check optimization completes in ~17-30 minutes
- [ ] Confirm results written to InfluxDB
- [ ] Compare performance with BOT2

---

**Deployment Prepared:** 2025-11-28 14:00 (Asia/Saigon)  
**Prepared By:** Cline AI Assistant  
**Status:** ✅ READY FOR EXECUTION

---

**IMPORTANT:** The deployment script is already on BOT1 at `/tmp/apply_bot1_fixes_locally.sh`. Simply SSH to BOT1 and run:

```bash
ssh ubuntu@192.168.3.71
sudo bash /tmp/apply_bot1_fixes_locally.sh
```

This will apply both critical fixes and restart the service automatically.
