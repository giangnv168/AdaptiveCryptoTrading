# FEEDER OPTUNA OPTIMIZATION - IMPLEMENTATION PLAN

**Date:** 2025-11-29  
**Status:** AWAITING APPROVAL

## 📋 EXECUTIVE SUMMARY

This document outlines the complete implementation plan for Optuna parameter optimization for Freqtrade feeders (feeder75a, feeder75b, feeder75c, feeder120a, feeder120b, feeder120c).

**Objective:** Optimize LightGBM hyperparameters for safety prediction models to improve prediction accuracy and model performance.

**Key Learning from BOT2:** Field name consistency is CRITICAL! All field names must be defined as constants and used consistently across optimizer, storage, and feeder code.

---

## ✅ PHASE 1-3: COMPLETED (Schema + Optimizer)

### Files Created

1. **`user_data/services/feeder_optuna_schema.py`**
   - Defines all field names as constants
   - Documents InfluxDB schema
   - Provides helper functions for validation
   - Prevents field name mismatch issues

2. **`user_data/services/optuna_feeder_optimizer.py`**
   - LightGBM hyperparameter optimizer
   - Optimizes per-pair parameters
   - Saves to InfluxDB with consistent field names
   - Evaluates MAE and safety accuracy

3. **`user_data/services/run_optuna_feeder_optimizer.py`**
   - CLI runner script
   - Supports all 6 feeders
   - Configurable trials, days back, pairs

### Parameters Being Optimized

| Parameter | Range | Description |
|-----------|-------|-------------|
| `learning_rate` | 0.001 - 0.1 | LightGBM learning rate |
| `n_estimators` | 100 - 1000 | Number of trees |
| `num_leaves` | 15 - 63 | Leaves per tree |
| `max_depth` | 3 - 12 | Tree depth |
| `min_child_samples` | 10 - 50 | Min samples per leaf |
| `reg_alpha` | 0.0 - 1.0 | L1 regularization |
| `reg_lambda` | 0.0 - 1.0 | L2 regularization |
| `quantile_alpha` | 0.05 - 0.20 | Quantile for safety prediction |
| `feature_fraction` | 0.6 - 1.0 | Feature sampling |
| `bagging_fraction` | 0.6 - 1.0 | Bagging fraction |

### InfluxDB Schema

**Measurement:** `optimized_parameters`

**Tags:**
- `feeder_id`: Which feeder (feeder75a, etc.)
- `pair`: Trading pair (BTC/USDT:USDT, etc.)
- `optimized_for`: Same as feeder_id
- `model_type`: "lightgbm_quantile"

**Fields (Parameters):** All 10 parameters above

**Fields (Metrics):**
- `backtested_mae`: Full backtest MAE
- `validation_mae`: Validation MAE
- `train_mae`: Training MAE
- `test_mae`: Test MAE
- `safety_accuracy`: Safety prediction accuracy
- `false_positive_rate`: FPR
- `false_negative_rate`: FNR
- `training_samples`: Sample count
- `optimization_trials`: Trial count
- `best_trial`: Best trial number
- `optimization_time`: Optimization duration

**Buckets:**
- `OPTUNA_FEEDER75A`
- `OPTUNA_FEEDER75B`
- `OPTUNA_FEEDER75C`
- `OPTUNA_FEEDER120A`
- `OPTUNA_FEEDER120B`
- `OPTUNA_FEEDER120C`

---

## 🔄 PHASE 4: FEEDER INTEGRATION (PENDING APPROVAL)

### What Needs to Be Done

Modify `user_data/strategies/LightGBMSafetyFeederStrategy.py` to:

1. **Load Optuna parameters from InfluxDB on startup**
   - Query latest optimized parameters per pair
   - Use filtered query to avoid old/mixed data
   - Validate parameters before use

2. **Apply parameters to model initialization**
   - Use loaded parameters instead of hardcoded values
   - Fall back to defaults if parameters not found
   - Log which parameters are being used

3. **Add debug logging**
   - Log parameter loading process
   - Show actual values loaded per pair
   - Warn if using fallback defaults

4. **Support parameter reload (SIGUSR1)**
   - Allow zero-downtime parameter updates
   - Reload latest parameters without restarting feeder

### Implementation Approach

#### Step 1: Add InfluxDB Connection to Strategy

```python
# In __init__:
from influxdb_client import InfluxDBClient
from user_data.services.feeder_optuna_schema import *

self.influx_client = InfluxDBClient(
    url='http://192.168.3.6:8086',
    token='...',
    org='ArtiSky'
)
self.optuna_params = {}  # Cache for parameters
```

#### Step 2: Load Parameters on Startup

```python
def _load_optuna_parameters(self):
    """Load latest Optuna parameters from InfluxDB."""
    feeder_id = self.config.get('bot_name', 'feeder75a')
    bucket = get_bucket_name(feeder_id)
    
    # Build field filter using schema constants
    field_filter = build_field_filter_query()
    
    query = f'''
    from(bucket: "{bucket}")
    |> range(start: -90d)
    |> filter(fn: (r) => r._measurement == "{INFLUX_MEASUREMENT}")
    |> filter(fn: (r) => r.{TAG_FEEDER_ID} == "{feeder_id}")
    |> filter(fn: (r) => {field_filter})
    |> group(columns: ["{TAG_PAIR}"])
    |> sort(columns: ["_time"], desc: true)
    |> limit(n: 1)
    |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
    '''
    
    # Execute query and parse results
    # Store in self.optuna_params[pair]
```

#### Step 3: Use Parameters in Model Init

```python
def _initialize_model(self, pair: str):
    """Initialize LightGBM model with Optuna parameters."""
    
    # Get parameters for this pair
    params = self.optuna_params.get(pair)
    
    if params:
        logger.info(f"🎯 Using Optuna params for {pair}")
        logger.debug(f"   LR={params[FIELD_LEARNING_RATE]:.4f}, "
                    f"Trees={params[FIELD_N_ESTIMATORS]:.0f}")
    else:
        logger.warning(f"⚠️  No Optuna params for {pair}, using defaults")
        params = get_default_params()
    
    # Create model with loaded parameters
    model = lgb.LGBMRegressor(
        objective='quantile',
        alpha=params[FIELD_QUANTILE_ALPHA],
        n_estimators=int(params[FIELD_N_ESTIMATORS]),
        num_leaves=int(params[FIELD_NUM_LEAVES]),
        learning_rate=params[FIELD_LEARNING_RATE],
        # ... etc
    )
    
    return model
```

#### Step 4: Add Validation Logging

```python
logger.info(f"✅ Loaded Optuna params for {len(self.optuna_params)} pairs")

for pair, params in self.optuna_params.items():
    is_valid, missing = validate_params(params)
    if not is_valid:
        logger.error(f"❌ {pair}: Missing fields: {missing}")
    else:
        logger.debug(f"   ✅ {pair}: All parameters valid")
```

---

## 🧪 PHASE 5: TESTING & DEPLOYMENT (PENDING APPROVAL)

### Testing Steps

1. **Create InfluxDB Buckets**
   ```bash
   # Create buckets for each feeder
   influx bucket create -n OPTUNA_FEEDER75A
   influx bucket create -n OPTUNA_FEEDER75B
   # ... etc
   ```

2. **Run Test Optimization (Single Pair)**
   ```bash
   # Test with one pair first
   .venv/bin/python3 user_data/services/run_optuna_feeder_optimizer.py \
       --feeder feeder75a \
       --pairs BTC/USDT:USDT \
       --n-trials 10 \
       --days-back 30
   ```

3. **Verify InfluxDB Data**
   ```bash
   # Query to check saved parameters
   influx query '
   from(bucket: "OPTUNA_FEEDER75A")
   |> range(start: -1h)
   |> filter(fn: (r) => r._measurement == "optimized_parameters")
   |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
   '
   ```

4. **Test Feeder Parameter Loading**
   - Start feeder with modified code
   - Check logs for "Loaded Optuna params for X pairs"
   - Verify actual parameter values in debug logs
   - Confirm no fallback warnings

5. **Full Optimization Run**
   ```bash
   # Optimize all pairs for one feeder
   .venv/bin/python3 user_data/services/run_optuna_feeder_optimizer.py \
       --feeder feeder75a \
       --n-trials 100 \
       --days-back 90
   ```

### Deployment Checklist

- [ ] InfluxDB buckets created for all 6 feeders
- [ ] Test optimization completed successfully
- [ ] Parameters saved to InfluxDB correctly
- [ ] Field names verified (no typos)
- [ ] Feeder code modified and tested
- [ ] Parameter loading verified in logs
- [ ] No schema collision errors
- [ ] Actual values loaded (not fallbacks)
- [ ] All 6 feeders deployed with Optuna integration
- [ ] Production optimization run completed
- [ ] Monitoring in place

---

## 🛡️ CRITICAL SAFEGUARDS (BOT2 Lessons Applied)

### 1. Field Name Consistency ✅

**Problem (BOT2):** Optimizer saved `stop_loss`, strategy queried `stop_loss_position_pct`  
**Solution:** All field names defined as constants in schema file

```python
# ✅ CORRECT
from feeder_optuna_schema import FIELD_LEARNING_RATE

point.field(FIELD_LEARNING_RATE, float(value))  # Optimizer
params.get(FIELD_LEARNING_RATE)  # Feeder
```

### 2. All Values as Float ✅

**Problem (BOT2):** Mixed int/float caused schema collisions  
**Solution:** Cast everything to float

```python
# ✅ CORRECT
point.field(FIELD_N_ESTIMATORS, float(500))  # Even integers!
```

### 3. Filtered Queries ✅

**Problem (BOT2):** Mixed old/new data caused unpredictable results  
**Solution:** Filter for specific field names

```python
# ✅ CORRECT
|> filter(fn: (r) => r._field == "learning_rate" or r._field == "n_estimators" ...)
|> sort(columns: ["_time"], desc: true)
|> limit(n: 1)  # NOT last()!
```

### 4. Validation Logging ✅

**Problem (BOT2):** Fallback values looked like Optuna values  
**Solution:** Validate and log actual values

```python
# ✅ CORRECT
if params is None:
    logger.warning(f"⚠️  No Optuna params for {pair} - using defaults")
else:
    logger.info(f"✅ Loaded Optuna params: LR={params['learning_rate']}")
```

---

## 📊 EXPECTED OUTCOMES

### Success Criteria

✅ Feeder logs show: `"Loaded Optuna params for X pairs"`  
✅ Debug logs show actual parameter values (not defaults)  
✅ Parameters vary per pair (not all identical)  
✅ Values in feeder match values in InfluxDB  
✅ No schema collision errors  
✅ Consistent behavior across restarts  
✅ Improved MAE and safety accuracy metrics

### Performance Metrics

Track these metrics before/after Optuna:
- **MAE (Mean Absolute Error):** Lower is better
- **Safety Accuracy:** Higher is better
- **False Positive Rate:** Lower is better
- **False Negative Rate:** Lower is better

---

## 🚀 EXECUTION TIMELINE

| Phase | Task | Estimated Time | Status |
|-------|------|----------------|--------|
| 1 | Schema Design | 1 hour | ✅ COMPLETE |
| 2 | Optimizer Implementation | 2 hours | ✅ COMPLETE |
| 3 | Runner Script | 1 hour | ✅ COMPLETE |
| 4 | Feeder Integration | 2 hours | ⏳ PENDING APPROVAL |
| 5 | Testing | 3 hours | ⏳ PENDING APPROVAL |
| 6 | Deployment | 1 hour | ⏳ PENDING APPROVAL |
| **TOTAL** | | **~10 hours** | **30% Complete** |

---

## ❓ QUESTIONS FOR REVIEW

Before proceeding with Phase 4-6, please confirm:

1. **Approach:** Is the overall approach acceptable?
   - Schema-first design
   - Separate optimizer script
   - Feeder integration with InfluxDB loading

2. **Parameters:** Are the 10 parameters to optimize correct?
   - Should we add/remove any?
   - Are the search ranges appropriate?

3. **Integration:** Is modifying the feeder strategy acceptable?
   - Will add InfluxDB connection
   - Will load parameters on startup
   - Will use parameters in model initialization

4. **Testing:** Should we test on a single feeder first?
   - Recommend starting with feeder75a
   - Run 1-2 pairs with 10 trials as proof-of-concept
   - Then scale to all feeders

5. **Deployment:** Any specific concerns?
   - Downtime requirements?
   - Backup strategy?
   - Rollback plan?

---

## 📝 NEXT STEPS (AFTER APPROVAL)

1. Review this implementation plan
2. Answer the questions above
3. Approve proceeding with Phase 4-6
4. I will implement feeder integration
5. Test with sample data
6. Deploy to production

---

## 📚 REFERENCE DOCUMENTS

- `OPTUNA_INFLUXDB_LESSONS_LEARNED_FOR_FEEDERS_2025-11-29.md` - BOT2 lessons
- `BOT2_OPTUNA_FIELD_NAME_MISMATCH_FIX_COMPLETE_2025-11-29.md` - Field name fixes
- `user_data/services/feeder_optuna_schema.py` - Schema definition
- `user_data/services/optuna_feeder_optimizer.py` - Optimizer implementation
- `user_data/services/run_optuna_feeder_optimizer.py` - Runner script

---

**Status:** Ready for review and approval before proceeding with feeder integration.
