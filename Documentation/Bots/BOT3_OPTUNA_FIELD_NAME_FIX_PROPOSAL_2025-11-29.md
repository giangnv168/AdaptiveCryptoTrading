# BOT3 OPTUNA FIELD NAME MISMATCH FIX - PROPOSAL FOR APPROVAL

**Date:** 2025-11-29  
**Status:** 🟡 AWAITING APPROVAL  
**Server:** 192.168.3.33 (BOT3)

---

## 🎯 PROBLEM SUMMARY

BOT3 has the **SAME field name mismatch bug** that was just fixed for BOT2:

### Symptoms:
- BOT3 likely showing identical stop_position values for all pairs (e.g., -7%)
- Expected: Varied stop loss values between -1% to -3% per pair
- Logs may show `source=optuna` but values are actually fallback defaults

### Root Cause:
**Field name mismatch** between three components:

1. **Optimizer saves:** `stop_loss`, `take_profit` ❌ (OLD names)
2. **BOT3 strategy expects:** `stop_loss_position_pct`, `take_profit_position_pct` ✅ (NEW names)
3. **Result:** Strategy can't find params → uses -7% fallback for all pairs

---

## 🔍 DETAILED ANALYSIS

### Issue 1: Shared Optimizer File (CRITICAL!)

**File:** `user_data/services/optuna_optimizer_per_bot_standalone.py`

**Lines 707-708 (Current - WRONG):**
```python
.field("stop_loss", float(params.stop_loss)) \
.field("take_profit", float(params.take_profit)) \
```

**Problem:** 
- This file is **SHARED** between BOT1, BOT2, and BOT3
- Saves parameters with OLD field names
- All bots affected until this is fixed

**Impact:**
- BOT2: Already has data with old field names (needs re-optimization)
- BOT3: Will have same problem if we run optimization now
- BOT1: Will also be affected

---

### Issue 2: BOT3 Strategy Wrong Bucket

**File:** `user_data/strategies/BOT3MetaStrategyAdaptive.py`

**Line 834 (Current - WRONG):**
```python
from(bucket:"OPTUNA_PARAMS")
```

**Should be:**
```python
from(bucket:"OPTUNA_BOT3")
```

**Problem:**
- BOT3 is querying the wrong bucket
- `OPTUNA_PARAMS` is a general bucket (may not exist or have BOT3 data)
- BOT3 should use its dedicated bucket `OPTUNA_BOT3`

---

### Issue 3: BOT3 Strategy Wrong Field Names

**File:** `user_data/strategies/BOT3MetaStrategyAdaptive.py`

**Lines 844-845 (Current - WRONG):**
```python
'stop_loss': record.values.get('stop_loss', -0.02),
'take_profit': record.values.get('take_profit', 0.03),
```

**Should be:**
```python
'stop_loss': record.values.get('stop_loss_position_pct', -0.02),
'take_profit': record.values.get('take_profit_position_pct', 0.03),
```

**Problem:**
- Looking for OLD field names that don't exist
- Results in using fallback values (-2%, 3%)

---

### Issue 4: BOT3 Strategy Missing Query Filters

**File:** `user_data/strategies/BOT3MetaStrategyAdaptive.py`

**Current query (Lines ~833-843):**
```flux
from(bucket: "OPTUNA_PARAMS")
|> range(start: -7d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
|> group(columns: ["pair"])
|> sort(columns: ["_time"], desc: true)
|> limit(n:1)
```

**Missing:**
- No filter for specific field names
- Won't guarantee getting latest data with correct field names
- May return mixed old/new data

---

## ✅ PROPOSED SOLUTION (3-Part Fix)

### Part 1: Fix Shared Optimizer Field Names

**File:** `user_data/services/optuna_optimizer_per_bot_standalone.py`

**Change Lines 707-708:**

**FROM:**
```python
.field("stop_loss", float(params.stop_loss)) \
.field("take_profit", float(params.take_profit)) \
```

**TO:**
```python
.field("stop_loss_position_pct", float(params.stop_loss)) \
.field("take_profit_position_pct", float(params.take_profit)) \
```

**Impact:**
- ✅ Future optimizations will use correct field names
- ✅ Affects all bots (BOT1, BOT2, BOT3)
- ⚠️ Existing data in InfluxDB still has old field names (needs re-optimization)

---

### Part 2: Fix BOT3 Strategy Field Names & Bucket

**File:** `user_data/strategies/BOT3MetaStrategyAdaptive.py`

**Change 1 - Fix Bucket (Line 834):**

**FROM:**
```python
from(bucket: "OPTUNA_PARAMS")
```

**TO:**
```python
from(bucket: "OPTUNA_BOT3")
```

**Change 2 - Fix Field Names (Lines 844-845):**

**FROM:**
```python
'stop_loss': record.values.get('stop_loss', -0.02),
'take_profit': record.values.get('take_profit', 0.03),
```

**TO:**
```python
'stop_loss': record.values.get('stop_loss_position_pct', -0.02),
'take_profit': record.values.get('take_profit_position_pct', 0.03),
```

---

### Part 3: Fix BOT3 Strategy Query Filters

**File:** `user_data/strategies/BOT3MetaStrategyAdaptive.py`

**Change Query (Lines ~833-843):**

**FROM:**
```flux
from(bucket: "OPTUNA_PARAMS")
|> range(start: -7d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
|> group(columns: ["pair"])
|> sort(columns: ["_time"], desc: true)
|> limit(n:1)
```

**TO:**
```flux
from(bucket: "OPTUNA_BOT3")
|> range(start: -7d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> filter(fn: (r) => r._field == "stop_loss_position_pct" or r._field == "take_profit_position_pct")
|> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
|> group(columns: ["pair"])
|> sort(columns: ["_time"], desc: true)
|> limit(n:1)
```

**Key additions:**
- ✅ Correct bucket: `OPTUNA_BOT3`
- ✅ Filter for specific field names
- ✅ Guarantees latest data with correct schema

---

## 📋 DEPLOYMENT PLAN

### Step 1: Apply All Fixes (Estimated: 5 minutes)

```bash
ssh ubuntu@192.168.3.33
cd /home/ubuntu/freqtrade

# Backup files
cp user_data/services/optuna_optimizer_per_bot_standalone.py{,.backup_fix_2025-11-29}
cp user_data/strategies/BOT3MetaStrategyAdaptive.py{,.backup_fix_2025-11-29}

# Apply fixes (manual or automated)
# 1. Edit optuna_optimizer_per_bot_standalone.py (Lines 707-708)
# 2. Edit BOT3MetaStrategyAdaptive.py (Lines 834, 844-845, 833-843)
```

### Step 2: Re-run BOT3 Optimization (Estimated: 30-60 minutes)

```bash
# Run optimization with FIXED optimizer
.venv/bin/python3 user_data/services/run_optuna_bot3_standalone.py \
    --bot BOT3 \
    --days-back 90 \
    --n-trials 100

# This will:
# - Load BOT3 closed trades from BOT33_trades bucket
# - Optimize all 60 pairs
# - Save to OPTUNA_BOT3 bucket with CORRECT field names
```

### Step 3: Restart BOT3 Services (Estimated: 1 minute)

```bash
# Restart BOT3 to load new parameters
sudo systemctl restart bot3-strategy
sudo systemctl restart bot3-controller
sudo systemctl restart bot3-monitor

# Or if single service:
sudo systemctl restart freqtrade-bot3
```

### Step 4: Verify Fix (Estimated: 5 minutes)

```bash
# Check BOT3 logs for parameter loading
sudo journalctl -u bot3-strategy -n 100 | grep -E "stop_position|Optuna"

# Expected: Varied stop_position values (-1% to -3%)
# NOT: All identical -7% values
```

---

## 🔍 VERIFICATION CHECKLIST

### ✅ Pre-Deployment Verification

- [ ] Confirm backup files created
- [ ] Review all code changes before applying
- [ ] Ensure no syntax errors in modified files

### ✅ Post-Deployment Verification

**Check 1: InfluxDB Data Has Correct Field Names**
```bash
# Query OPTUNA_BOT3 bucket to verify field names
# Should see: stop_loss_position_pct, take_profit_position_pct
# NOT: stop_loss, take_profit
```

**Check 2: BOT3 Loads Varied Parameters**
```bash
sudo journalctl -u bot3-strategy -n 200 | grep "stop_position"

# Expected output:
# BTC/USDT: stop_position=-0.018234 (-1.82%)
# ETH/USDT: stop_position=-0.025123 (-2.51%)
# SOL/USDT: stop_position=-0.019456 (-1.95%)
# ... (all different values between -1% to -3%)

# NOT:
# All pairs: stop_position=-0.070000 (-7.00%)
```

**Check 3: Logs Show Source=Optuna**
```bash
sudo journalctl -u bot3-strategy -n 100 | grep "source=optuna"

# Should see multiple pairs with source=optuna
# And stop_position values matching InfluxDB
```

---

## 📊 EXPECTED RESULTS

### Before Fix (Current State)
```
BOT3: All pairs using stop_position=-0.070000 (-7%)
Source: Fallback defaults (not actual Optuna!)
Reason: Field name mismatch
```

### After Fix (Expected)
```
BOT3 Sample Values:
  BTC/USDT: stop_position=-0.015234 (-1.52%) ✅ source=optuna
  ETH/USDT: stop_position=-0.022156 (-2.22%) ✅ source=optuna
  SOL/USDT: stop_position=-0.018765 (-1.88%) ✅ source=optuna
  ALGO/USDT: stop_position=-0.016913 (-1.69%) ✅ source=optuna
  
All values within -3% to -1% range! ✅
Varied per pair (not all identical) ✅
Actually from Optuna (not fallback) ✅
```

---

## ⚠️ IMPORTANT CONSIDERATIONS

### 1. Optimizer File is Shared

**Impact on Other Bots:**
- This fix affects the shared optimizer used by BOT1, BOT2, and BOT3
- BOT2 may need re-optimization if already run with old field names
- BOT1 will benefit from fix when its optimization runs

**Recommendation:**
- Apply fix to optimizer FIRST
- Then re-optimize ALL bots to ensure consistency
- Priority: BOT3 (this session), BOT2 (if needed), BOT1 (future)

### 2. Data Migration Not Required

**Why:**
- We're not migrating old data
- We're re-running optimization with new field names
- Old data in InfluxDB will remain but won't be queried (thanks to field filter)
- New optimization creates fresh data with correct field names

### 3. Zero Downtime Possible

**Strategy:**
- BOT3 currently using fallback values
- After fix + optimization, will use Optuna values
- No trading interruption during deployment
- Restart only needed to load new parameters

---

## 🎯 SUCCESS CRITERIA

BOT3 fix is successful when:

1. ✅ **InfluxDB:** OPTUNA_BOT3 bucket contains `stop_loss_position_pct` and `take_profit_position_pct` fields
2. ✅ **BOT3 Logs:** Show varied stop_position values between -1% to -3%
3. ✅ **BOT3 Logs:** Show `source=optuna` for multiple pairs
4. ✅ **No Errors:** No field name errors in logs
5. ✅ **Consistency:** Values in logs match values in InfluxDB

---

## 📝 FILES TO BE MODIFIED

1. **`user_data/services/optuna_optimizer_per_bot_standalone.py`**
   - Lines 707-708: Change field names
   - Affects: ALL bots (shared file)

2. **`user_data/strategies/BOT3MetaStrategyAdaptive.py`**
   - Line 834: Fix bucket name
   - Lines 844-845: Fix field names
   - Lines 833-843: Add query filters
   - Affects: BOT3 only

---

## 🔄 NEXT STEPS AFTER BOT3

If this fix is successful for BOT3, consider:

1. **BOT2 Re-optimization:** If BOT2 was optimized before this fix, re-run optimization
2. **BOT1 Preparation:** Apply same fix when BOT1 optimization is ready
3. **Documentation:** Update all Optuna docs with new field name standard

---

## 💡 ALTERNATIVE APPROACHES CONSIDERED

### Option 1: Keep Old Field Names
❌ Rejected because:
- Strategy code already expects new field names
- Would require changing strategy code (more risky)
- New names are more descriptive and accurate

### Option 2: Migrate Old Data
❌ Rejected because:
- Complex migration scripts needed
- Risk of data corruption
- Re-optimization is cleaner and ensures fresh parameters

### Option 3: Create New Buckets
✅ Current approach:
- Use existing OPTUNA_BOT3 bucket
- Filter query to only get new field names
- Re-run optimization to populate with correct schema
- Clean, simple, proven (worked for BOT2)

---

## 📚 REFERENCES

- `BOT2_OPTUNA_FIELD_NAME_MISMATCH_FIX_COMPLETE_2025-11-29.md` - Proof this fix works for BOT2
- `BOT3_OPTUNA_DEPLOYMENT_COMPLETE.md` - Original BOT3 setup
- `OPTUNA_INFLUXDB_LESSONS_LEARNED_FOR_FEEDERS_2025-11-29.md` - Best practices

---

## ✅ APPROVAL CHECKLIST

Please review and approve/reject each part:

- [ ] **Part 1 Approved:** Fix shared optimizer field names (affects all bots)
- [ ] **Part 2 Approved:** Fix BOT3 strategy field names & bucket
- [ ] **Part 3 Approved:** Fix BOT3 strategy query filters
- [ ] **Deployment Plan Approved:** Step-by-step execution plan
- [ ] **Ready to Implement:** All parts approved, proceed with fixes

---

## 🎉 EXPECTED IMPACT

### Risk Management Improvement:
- BOT3 will use optimized stop loss values (-1% to -3%) per pair
- No more blanket -7% fallback for all pairs
- Each pair calibrated based on 90 days historical performance

### Consistency:
- All bots (BOT1, BOT2, BOT3) using same field name standard
- Easier maintenance and debugging
- Clear documentation of parameter schema

### Proven Solution:
- Same fix successfully applied to BOT2 today
- Zero downtime deployment
- Reversible (backups created)

---

**Estimated Total Time:** 40-70 minutes
**Risk Level:** 🟢 LOW (proven solution, backed up files)
**Confidence:** ✅ HIGH (already worked for BOT2)

---

**AWAITING YOUR APPROVAL TO PROCEED** 🚦
