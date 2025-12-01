# BOT2 Optuna Parameter Mismatch - Root Cause Analysis
**Date:** November 28, 2025  
**Server:** BOT2 (192.168.3.72)  
**Status:** 🔴 CRITICAL ISSUE IDENTIFIED

## Executive Summary

The Optuna optimizer is running successfully and completing trials, but BOT2 is unable to read the optimized parameters due to **mismatched InfluxDB schema** between the optimizer (writer) and strategy (reader).

**Result:** BOT2 loads parameters for **0 pairs** and falls back to hardcoded values, despite 50+ optimization trials being completed.

---

## 🔍 Detailed Findings

### 1. Optuna Optimizer Status ✅
- **Service:** `optuna-bot2-standalone.service`
- **Status:** Active, running for 3h 35min
- **Trials Completed:** 50+ trials
- **Best Parameters Found (Trial 49):**
  - Stop Loss: -1.01%
  - Take Profit: 8.60%
  - Value: 0.004760

### 2. BOT2 Strategy Status ⚠️
- **Status:** Running since 10:29 (~6 hours)
- **Parameters Loaded:** **0 pairs** (critical issue)
- **Current Behavior:** Using fallback parameters only
- **Evidence:**
  ```
  2025-11-28 16:00:01 - 📊 Loaded Optuna params for 0 pairs
  2025-11-28 16:30:02 - 📊 Loaded Optuna params for 0 pairs
  2025-11-28 16:37:40 - 🎯 BOT2 SHORT TARGET: M/USDT:USDT @ 1.01% (fallback)
  ```

---

## 🔴 Root Cause: Schema Mismatch

### Issue #1: Measurement Name Mismatch

**Optimizer WRITES:**
```python
# File: user_data/services/optuna_optimizer_per_bot_standalone.py
self.influx_measurement = "optimized_parameters"  # Line 74

point = Point(self.influx_measurement)  # Uses "optimized_parameters"
```

**BOT2 Strategy READS:**
```python
# File: BOT2_ArtiSkyTrader_Optuna_Fixed.py
query = '''
from(bucket: "OPTUNA_PARAMS")
|> filter(fn: (r) => r._measurement == "parameters")  # ❌ WRONG!
'''
```

❌ **Mismatch:** `"optimized_parameters"` vs `"parameters"`

---

### Issue #2: Field Name Mismatch

**Optimizer WRITES:**
```python
.field("stop_loss", float(params.stop_loss))
.field("take_profit", float(params.take_profit))
```

**BOT2 Strategy READS:**
```python
take_profit = abs(record.values.get('take_profit_pct', 0.01))  # ❌ WRONG!
stop_loss = -abs(record.values.get('stop_loss_pct', 0.07))     # ❌ WRONG!
```

❌ **Mismatch:** 
- Optimizer: `"stop_loss"`, `"take_profit"`
- BOT2: `"stop_loss_pct"`, `"take_profit_pct"`

---

### Issue #3: Missing Bot Filter

**Optimizer WRITES:**
```python
.tag("optimized_for_bot", self.target_bot)  # Tags with "BOT2"
```

**BOT2 Strategy READS:**
```python
# No filter for optimized_for_bot tag! ❌
query = '''
from(bucket: "OPTUNA_PARAMS")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "parameters")
|> group(columns: ["pair"])
|> last()
'''
```

⚠️ **Missing:** No filter for `optimized_for_bot == "BOT2"`

---

### Issue #4: Optimizer May Not Be Completing

**Log Analysis:**
```
2025-11-28 13:01:32 - ✅ InfluxDB connected
2025-11-28 13:01:32 - Storage: InfluxDB bucket 'OPTUNA_PARAMS'
[... many trial logs ...]
# ❌ NO "OPTIMIZATION COMPLETE" or "Results saved" messages!
```

The optimizer connects and runs trials but may not be:
1. Completing the optimization cycle
2. Saving results to InfluxDB
3. Handling errors silently

---

## 📊 Complete Schema Comparison

| Component | Measurement | Tags | Fields |
|-----------|------------|------|--------|
| **Optimizer Writes** | `optimized_parameters` | `pair`, `optimized_for_bot` | `stop_loss`, `take_profit`, `volatility_multiplier`, `win_rate`, etc. |
| **BOT2 Reads** | `parameters` ❌ | `pair` only ⚠️ | `stop_loss_pct` ❌, `take_profit_pct` ❌ |

---

## 🔧 Required Fixes

### Fix #1: Update BOT2 Strategy Query (CRITICAL)

**Location:** `BOT2_ArtiSkyTrader_Optuna_Fixed.py`, method `_load_optuna_params()`

**Change FROM:**
```python
query = '''
from(bucket: "OPTUNA_PARAMS")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "parameters")
|> group(columns: ["pair"])
|> last()
'''
```

**Change TO:**
```python
query = '''
from(bucket: "OPTUNA_PARAMS")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> filter(fn: (r) => r["optimized_for_bot"] == "BOT2")
|> group(columns: ["pair"])
|> last()
'''
```

### Fix #2: Update Field Names (CRITICAL)

**Change FROM:**
```python
take_profit = abs(record.values.get('take_profit_pct', 0.01))
stop_loss = -abs(record.values.get('stop_loss_pct', 0.07))
```

**Change TO:**
```python
take_profit = abs(record.values.get('take_profit', 0.01))
stop_loss = abs(record.values.get('stop_loss', -0.07))  # Note: optimizer stores as negative
```

### Fix #3: Verify Optimizer Completion

Check if optimizer is successfully completing and saving results:
1. Monitor log for "OPTIMIZATION COMPLETE" messages
2. Verify InfluxDB write success
3. Check for any silent errors in optimizer service

---

## 🚀 Implementation Steps

### Step 1: Fix BOT2 Strategy on Remote Server
```bash
# SSH to BOT2 server
ssh ubuntu@192.168.3.72

# Navigate to strategy directory
cd /home/ubuntu/freqtrade/user_data/strategies

# Edit the ConsumerStrategy file (BOT2 strategy)
nano ConsumerStrategy.py  # or whatever the actual filename is

# Make the changes to _load_optuna_params() method
```

### Step 2: Restart BOT2
```bash
# Restart BOT2 trading bot to load new strategy
sudo systemctl restart freqtrade-bot2  # or whatever the service name is
```

### Step 3: Verify Parameter Loading
```bash
# Monitor BOT2 logs
tail -f /home/ubuntu/freqtrade/bot2.log | grep -i "optuna\|loaded"

# Should see: "📊 Loaded Optuna params for X pairs" where X > 0
```

### Step 4: Monitor Performance
```bash
# Watch for Optuna parameters being used instead of fallback
tail -f /home/ubuntu/freqtrade/bot2.log | grep -i "optuna\|fallback"

# Should see: "roi_optuna" and "stop_loss_optuna" instead of "roi_fallback"
```

---

## 📈 Expected Results After Fix

### Before Fix:
```
📊 Loaded Optuna params for 0 pairs
🎯 BOT2 SHORT TARGET: M/USDT:USDT @ 1.01% (fallback)
exit_reason: 'cutloss_fallback'
exit_reason: 'roi_fallback'
```

### After Fix:
```
📊 Loaded Optuna params for 163 pairs
🎯 BOT2 SHORT TARGET: BTC/USDT:USDT @ 8.60% (optuna)
exit_reason: 'roi_optuna'
exit_reason: 'stop_loss_optuna'
```

---

## 🔍 Verification Checklist

- [ ] BOT2 strategy file updated with correct measurement name
- [ ] BOT2 strategy file updated with correct field names
- [ ] BOT2 strategy file updated with bot filter
- [ ] BOT2 service restarted
- [ ] BOT2 logs show "Loaded Optuna params for X pairs" where X > 0
- [ ] BOT2 logs show "optuna" source instead of "fallback"
- [ ] Trades using optimized parameters (roi_optuna, stop_loss_optuna)
- [ ] Monitor performance improvement over next 24-48 hours

---

## 📝 Notes

1. **Urgent:** This is a critical issue preventing optimized parameters from being used
2. **Impact:** BOT2 has been trading suboptimally with fallback parameters despite having optimized values available
3. **Timeline:** Fix should be deployed immediately
4. **Risk:** Low risk - fallback will still work if InfluxDB is unavailable
5. **Testing:** Can be tested in dry-run mode first if desired

---

## 📞 Support

For questions or issues during deployment:
- Check this document's implementation steps
- Review BOT2 logs: `/home/ubuntu/freqtrade/bot2.log`
- Review Optuna logs: `/home/ubuntu/freqtrade/user_data/logs/optuna_optimizer_BOT2.log`
- Verify InfluxDB connectivity: `http://192.168.3.6:8086`
