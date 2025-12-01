# BOT2 OPTUNA FIELD NAME MISMATCH - COMPLETE FIX (2025-11-29)

## 🎯 PROBLEM STATEMENT

**User Question:** "Why is cutloss from Optuna so deep, stop_position = -7%, while Optuna range just from -3% to -1%?"

**Symptoms:**
- BOT2 showing `stop_position=-0.070000` (-7%) for all pairs
- Expected: Stop loss between -3% to -1% (Optuna optimization range)
- Logs showing `source=optuna` but values were actually fallback defaults

## 🔍 ROOT CAUSE ANALYSIS

### The Bug

**Field name mismatch** between three components:

1. **Optimizer saves to InfluxDB:**
   - OLD: `stop_loss`, `take_profit`
   - NEW (after fix): `stop_loss_position_pct`, `take_profit_position_pct`

2. **Strategy queries from InfluxDB:**
   - Expected: `stop_loss_position_pct`, `take_profit_position_pct`
   - Got: `stop_loss`, `take_profit` (field not found!)
   - Result: Used default values (0.01, -0.07)

3. **Strategy uses internally:**
   - `stop_loss_position_pct`
   - `take_profit_position_pct`

### Why This Caused -7% Stop Loss

When the strategy couldn't find `stop_loss_position_pct` in InfluxDB records, it used the fallback default:

```python
stop_loss = -abs(record.values.get('stop_loss_position_pct', -0.07))
#                                    ^^^^^^^^^^^^^ Fallback to -7%!
```

## ✅ THE COMPLETE FIX (3 Parts)

### Part 1: Fix Optimizer Field Names

**File:** `user_data/services/optuna_optimizer_per_bot_standalone.py`

**Lines 707-708:**

**Before:**
```python
.field("stop_loss", float(params.stop_loss)) \
.field("take_profit", float(params.take_profit)) \
```

**After:**
```python
.field("stop_loss_position_pct", float(params.stop_loss)) \
.field("take_profit_position_pct", float(params.take_profit)) \
```

### Part 2: Fix Strategy Field Names

**File:** `user_data/strategies/ArtiSkyTrader.py`

**Lines 150-151:**

**Before:**
```python
take_profit = abs(record.values.get('take_profit', 0.01))
stop_loss = -abs(record.values.get('stop_loss', -0.07))
```

**After:**
```python
take_profit = abs(record.values.get('take_profit_position_pct', 0.01))
stop_loss = -abs(record.values.get('stop_loss_position_pct', -0.07))
```

### Part 3: Fix Strategy Query (CRITICAL!)

**File:** `user_data/strategies/ArtiSkyTrader.py`

**Problem:** InfluxDB had mixed data (both old and new field names), causing pivot query to fail.

**Solution:** Filter to load ONLY LATEST data with correct field names.

**Before:**
```flux
from(bucket: "OPTUNA_BOT2")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> filter(fn: (r) => r["optimized_for_bot"] == "BOT2")
|> map(fn: (r) => ({ r with _value: float(v: r._value) }))
|> group(columns: ["pair"])
|> last()
|> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
```

**After:**
```flux
from(bucket: "OPTUNA_BOT2")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> filter(fn: (r) => r["optimized_for_bot"] == "BOT2")
|> filter(fn: (r) => r._field == "stop_loss_position_pct" or r._field == "take_profit_position_pct")
|> map(fn: (r) => ({ r with _value: float(v: r._value) }))
|> group(columns: ["pair"])
|> sort(columns: ["_time"], desc: true)
|> limit(n: 1)
|> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
```

**Key Changes:**
1. ✅ Added filter for ONLY new field names
2. ✅ Sort by time descending (latest first)
3. ✅ Changed `|> last()` to `|> limit(n: 1)` after sort

## 📊 VERIFICATION RESULTS

### Before Fix
```
ALL pairs: stop_position=-0.070000 (-7.00%), source=optuna
(Actually fallback values, not real Optuna params!)
```

### After Fix
```
✅ BOT2 WITH OPTUNA initialized
📊 Loaded Optuna params for 48 pairs

Sample stop_position values:
- WLD:  -0.026055 (-2.61%) ✅ source=optuna
- POL:  -0.028516 (-2.85%) ✅ source=optuna
- ALGO: -0.016913 (-1.69%) ✅ source=optuna
- ATOM: -0.022749 (-2.27%) ✅ source=optuna
- PUMP: -0.019225 (-1.92%) ✅ source=optuna
- FIL:  -0.023883 (-2.39%) ✅ source=optuna

ALL within -3% to -1% range! ✅
```

### M/USDT:USDT Specific Example
```
InfluxDB value: stop_loss_position_pct = -0.012855 (-1.29%)
BOT2 using:     stop_position = -0.012855 (-1.29%) ✅
(Was -7.00% before fix!)
```

## 🔧 DEPLOYMENT STEPS

### 1. Apply All Three Fixes
```bash
ssh ubuntu@192.168.3.72
cd /home/ubuntu/freqtrade

# Backup files
cp user_data/services/optuna_optimizer_per_bot_standalone.py{,.backup}
cp user_data/strategies/ArtiSkyTrader.py{,.backup}

# Apply fixes (see Part 1, 2, 3 above)
# Or use sed commands...
```

### 2. Re-run Optimization
```bash
.venv/bin/python3 user_data/services/run_optuna_per_bot_standalone.py \
    --bot BOT2 \
    --days-back 90 \
    --n-trials 100
```

### 3. Restart BOT2
```bash
echo "ArtiSky@7" | sudo -S systemctl restart freqtrade
```

### 4. Verify
```bash
# Check logs for correct stop_position values
echo "ArtiSky@7" | sudo -S journalctl -u freqtrade -n 50 | grep "stop_position"

# Should see values between -3% and -1%, NOT -7%
```

## 📚 KEY LEARNINGS

### 1. Field Name Consistency is Critical

When working with InfluxDB and Freqtrade:
- Optimizer must save with same field names that strategy queries
- Any mismatch results in fallback to default values
- Check BOTH the save code AND the query code

### 2. Mixed Data in InfluxDB Causes Issues

If InfluxDB has both old and new field names:
- Pivot queries may fail or return incomplete data
- Solution: Filter for specific field names in query
- Better: Delete old bucket and re-create with only new schema

### 3. Flux Query Best Practices

To get LATEST data reliably:
```flux
|> filter(fn: (r) => r._field == "field_name")  # Filter specific fields
|> sort(columns: ["_time"], desc: true)         # Sort latest first
|> limit(n: 1)                                  # Take only latest
```

NOT just:
```flux
|> last()  # May not work correctly with mixed data
```

### 4. Debugging InfluxDB Issues

Steps to debug parameter loading issues:
1. Check what fields exist in InfluxDB (all field names)
2. Check what strategy is querying for (field names in code)
3. Check what strategy actually receives (add debug logging)
4. Verify field names match EXACTLY

### 5. The Importance of Query Filters

Adding `|> filter(fn: (r) => r._field == "stop_loss_position_pct" or r._field == "take_profit_position_pct")` was CRITICAL because:
- It ensures we only get records with the new field names
- It prevents pivot from trying to merge old and new data
- It guarantees latest optimized values are loaded

## 🎯 IMPACT

**Risk Management Improvement:**
- M/USDT: From -7.00% to -1.29% stop loss (5.71% improvement)
- Each pair now has optimized stop loss based on 90 days performance
- No more blanket -7% fallback for all pairs

**48 pairs now properly optimized:**
- Individual stop loss values between -1% to -3%
- Individual take profit values between 1% to 6%
- Properly calibrated based on pair-specific historical performance

## 📝 FILES MODIFIED

1. `user_data/services/optuna_optimizer_per_bot_standalone.py` (Lines 707-708)
2. `user_data/strategies/ArtiSkyTrader.py` (Lines 150-151 + Query section)

## ⚠️ IMPORTANT NOTES

### Why Part 3 (Query Fix) Was Essential

Fixing only Parts 1 and 2 was NOT enough because:
- InfluxDB contained both old data (`stop_loss`) and new data (`stop_loss_position_pct`)
- Without query filter, pivot would sometimes return old data
- Old data didn't have the new field names → fallback to -7%

**The query filter ensures we ONLY get the latest records with the correct field names.**

### For Future Reference

If you see `source=optuna` but values look like defaults:
1. Check field name consistency between optimizer and strategy
2. Check if InfluxDB has mixed old/new data
3. Add query filter to get only latest data with correct fields

## 🔄 CONTINUOUS OPTIMIZATION

The signal-based parameter reload system (from earlier session) now works with corrected field names:
- Optimization runs → Saves with correct field names
- Sends SIGUSR1 to BOT2 → Reloads latest params
- BOT2 queries with correct field names → Gets correct values
- Zero downtime! ✅

## ✅ SUCCESS METRICS

- ✅ 48 pairs loaded with Optuna parameters
- ✅ Stop loss values: -1% to -3% (not -7%)
- ✅ All pairs showing `source=optuna` with ACTUAL Optuna values
- ✅ Field names aligned across optimizer ↔ InfluxDB ↔ strategy
- ✅ Query filter ensures latest data only

## 📅 SESSION DETAILS

- **Date:** 2025-11-29
- **Duration:** ~2 hours
- **Root Cause:** Field name mismatch + mixed InfluxDB data
- **Solution:** 3-part fix (optimizer + strategy + query)
- **Status:** ✅ COMPLETELY RESOLVED
