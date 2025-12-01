# OPTUNA + INFLUXDB LESSONS LEARNED - FEEDER OPTIMIZATION GUIDE (2025-11-29)

## 📚 PURPOSE

This document captures critical lessons learned from debugging BOT2's Optuna parameter loading issues. Use this as a reference when implementing Optuna optimization for feeders to avoid the same pitfalls.

## 🎯 CORE PRINCIPLES

### 1. Field Name Consistency is CRITICAL

**Rule:** Field names must be IDENTICAL across all three layers:
- ✅ Optimizer saves to InfluxDB
- ✅ InfluxDB stores the data
- ✅ Strategy/Feeder queries from InfluxDB

**Example of what went wrong:**
```python
# Optimizer saved:
.field("stop_loss", value)

# Strategy queried:
record.values.get('stop_loss_position_pct', default)
#                  ^^^^^^^^^^^^^^^^^^^^^^^ MISMATCH!
# Result: Always got default value, never the optimized value!
```

**How to avoid:**
1. Define field names as constants in one place
2. Use the same constants everywhere
3. Document the schema clearly

### 2. InfluxDB Schema Must Be Clean

**Problem:** Mixed old/new field names in same bucket causes pivot queries to fail.

**Example:**
```
OPTUNA_BOT2 bucket contains:
- Old records: stop_loss, take_profit
- New records: stop_loss_position_pct, take_profit_position_pct

Query with pivot() returns unpredictable results!
```

**Solutions:**
- **Option A:** Delete old bucket and recreate (clean slate)
- **Option B:** Use query filters to select specific field names only
- **Option C:** Version your buckets (OPTUNA_BOT2_V1, OPTUNA_BOT2_V2)

### 3. Flux Query Best Practices

**BAD Query (unreliable with mixed data):**
```flux
from(bucket: "OPTUNA_BOT2")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> group(columns: ["pair"])
|> last()  # ❌ May return old data!
|> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
```

**GOOD Query (guarantees latest data with correct fields):**
```flux
from(bucket: "OPTUNA_BOT2")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> filter(fn: (r) => r.optimized_for_bot == "BOT2")
|> filter(fn: (r) => r._field == "stop_loss_position_pct" or r._field == "take_profit_position_pct")
|> group(columns: ["pair"])
|> sort(columns: ["_time"], desc: true)  # ✅ Latest first
|> limit(n: 1)                            # ✅ Take only latest
|> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
```

**Key differences:**
1. ✅ Filter for specific field names
2. ✅ Sort by time descending
3. ✅ Use `limit(n: 1)` instead of `last()`

## 🔧 IMPLEMENTATION CHECKLIST FOR FEEDERS

### Phase 1: Schema Design

**1.1 Define Your Parameter Schema**

Create a schema document (e.g., `FEEDER_OPTUNA_SCHEMA.md`):

```python
# Feeder Optuna Parameters Schema

MEASUREMENT = "optimized_parameters"

# Tags (for filtering)
TAGS = {
    "feeder_id": "feeder75a",  # Which feeder
    "pair": "BTC/USDT:USDT",   # Which trading pair
    "optimized_for": "feeder75a"  # Tag for easy filtering
}

# Fields (actual parameter values)
FIELDS = {
    # Model hyperparameters
    "learning_rate": float,
    "num_estimators": float,  # ⚠️ Cast to float!
    "max_depth": float,        # ⚠️ Cast to float!
    
    # Quantile parameters
    "quantile_long": float,
    "quantile_short": float,
    
    # Performance metrics
    "backtested_profit": float,
    "win_rate": float,
    "sharpe_ratio": float,
    "optimization_trials": float,  # ⚠️ Cast to float!
    
    # Timestamp
    "_time": datetime  # InfluxDB auto-adds
}
```

**1.2 Critical Rules:**

✅ **All numeric values MUST be float** (not int!)
- InfluxDB stores integers differently than floats
- Mixed int/float causes schema collision errors
- Always cast: `float(value)`

✅ **Field names must be descriptive and unique**
- ❌ Bad: `param1`, `param2`, `value`
- ✅ Good: `learning_rate`, `num_estimators`, `quantile_long`

✅ **Use tags for filtering, fields for values**
- Tags: Identifiers (feeder_id, pair, model_type)
- Fields: Numeric values (parameters, metrics)

### Phase 2: Optimizer Implementation

**2.1 Write Data with Consistent Field Names**

```python
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

def save_optuna_params(feeder_id, pair, params):
    """
    Save Optuna parameters to InfluxDB.
    
    CRITICAL: Field names here MUST match what feeder queries for!
    """
    point = Point("optimized_parameters")
    
    # Add tags (for filtering)
    point.tag("feeder_id", feeder_id)
    point.tag("pair", pair)
    point.tag("optimized_for", feeder_id)
    
    # Add fields (actual values)
    # ⚠️ CRITICAL: Use EXACT same field names that feeder will query!
    point.field("learning_rate", float(params.learning_rate))
    point.field("num_estimators", float(params.num_estimators))
    point.field("max_depth", float(params.max_depth))
    point.field("quantile_long", float(params.quantile_long))
    point.field("quantile_short", float(params.quantile_short))
    
    # Add metrics
    point.field("backtested_profit", float(params.backtested_profit))
    point.field("win_rate", float(params.win_rate))
    point.field("sharpe_ratio", float(params.sharpe_ratio))
    point.field("optimization_trials", float(params.optimization_trials))
    
    # Write to InfluxDB
    with InfluxDBClient(url=INFLUX_URL, token=INFLUX_TOKEN, org=INFLUX_ORG) as client:
        write_api = client.write_api(write_options=SYNCHRONOUS)
        write_api.write(bucket=f"OPTUNA_{feeder_id}", record=point)
        
    logger.info(f"✅ Saved Optuna params for {pair} to OPTUNA_{feeder_id}")
```

**2.2 Important Notes:**

- **Cast everything to float:** `float(value)` prevents schema collisions
- **Use descriptive field names:** Not `sl`, but `stop_loss_position_pct`
- **Tag vs Field:** Use tags for filtering, fields for values
- **Bucket naming:** Use dedicated buckets per feeder (OPTUNA_FEEDER75A)

### Phase 3: Feeder Parameter Loading

**3.1 Query with Field Name Consistency**

```python
def load_optuna_params(feeder_id):
    """
    Load latest Optuna parameters from InfluxDB.
    
    CRITICAL: Field names in get() MUST match what optimizer saved!
    """
    query = f'''
    from(bucket: "OPTUNA_{feeder_id}")
    |> range(start: -90d)
    |> filter(fn: (r) => r._measurement == "optimized_parameters")
    |> filter(fn: (r) => r["optimized_for"] == "{feeder_id}")
    |> filter(fn: (r) => 
        r._field == "learning_rate" or 
        r._field == "num_estimators" or 
        r._field == "max_depth" or 
        r._field == "quantile_long" or 
        r._field == "quantile_short"
    )
    |> group(columns: ["pair"])
    |> sort(columns: ["_time"], desc: true)
    |> limit(n: 1)
    |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
    '''
    
    with InfluxDBClient(url=INFLUX_URL, token=INFLUX_TOKEN, org=INFLUX_ORG) as client:
        result = client.query_api().query(query)
        
        params = {}
        for table in result:
            for record in table.records:
                pair = record.values.get('pair')
                
                # ⚠️ CRITICAL: Use EXACT same field names as optimizer saved!
                params[pair] = {
                    'learning_rate': record.values.get('learning_rate', 0.01),  # ✅ Matches optimizer
                    'num_estimators': int(record.values.get('num_estimators', 100)),  # Convert back to int if needed
                    'max_depth': int(record.values.get('max_depth', 5)),
                    'quantile_long': record.values.get('quantile_long', 0.95),
                    'quantile_short': record.values.get('quantile_short', 0.05),
                }
                
        logger.info(f"✅ Loaded Optuna params for {len(params)} pairs")
        return params
```

**3.2 Critical Query Elements:**

1. **Filter for specific field names:**
   ```flux
   |> filter(fn: (r) => r._field == "learning_rate" or r._field == "num_estimators")
   ```
   This prevents loading old/mixed data!

2. **Sort and limit:**
   ```flux
   |> sort(columns: ["_time"], desc: true)
   |> limit(n: 1)
   ```
   Guarantees you get the LATEST optimization results!

3. **Pivot for easy access:**
   ```flux
   |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
   ```
   Converts fields into columns for easy access

### Phase 4: Debugging & Validation

**4.1 Verification Checklist**

After implementing Optuna for feeders, verify:

```bash
# 1. Check what fields are in InfluxDB
influx query '
from(bucket: "OPTUNA_FEEDER75A")
|> range(start: -1h)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> group(columns: ["_field"])
|> count()
' | jq '.[] | .values | .[0] | ._field'

# Expected output: All your field names (learning_rate, num_estimators, etc.)
```

```python
# 2. Verify feeder loaded correct values
import logging
logging.basicConfig(level=logging.DEBUG)

# Check feeder logs for:
# ✅ "Loaded Optuna params for X pairs"
# ✅ Actual parameter values (not defaults)
```

**4.2 Common Issues & Solutions**

| Symptom | Cause | Solution |
|---------|-------|----------|
| Feeder using default params | Field name mismatch | Check optimizer saves same field names feeder queries |
| Schema collision errors | Mixed int/float types | Cast all values to `float()` |
| Old parameters loaded | No filter for latest | Add `sort() + limit(1)` to query |
| Random/inconsistent values | Mixed old/new data in bucket | Add field name filter to query |
| "Field not found" errors | Typo in field names | Use constants for field names |

**4.3 Debug Logging Template**

Add this to your feeder code:

```python
logger.info(f"🔍 DEBUG: Loading Optuna params from OPTUNA_{feeder_id}")
logger.info(f"🔍 DEBUG: Query returned {len(result)} tables")

for i, table in enumerate(result):
    logger.info(f"🔍 DEBUG: Table {i} has {len(table.records)} records")
    for j, record in enumerate(table.records):
        logger.info(f"🔍 DEBUG: Record {j} keys: {list(record.values.keys())}")
        logger.info(f"🔍 DEBUG: Record {j} values: {record.values}")
        
        # Verify field names
        pair = record.values.get('pair')
        lr = record.values.get('learning_rate')
        logger.info(f"🔍 DEBUG: {pair} - learning_rate={lr}")
```

## 🚨 CRITICAL LESSONS FROM BOT2 DEBUGGING

### Lesson 1: Field Name Mismatch is Silent & Deadly

**What happened:**
- Optimizer saved: `stop_loss`
- Strategy queried: `stop_loss_position_pct`
- No error! Just silently used default: `-0.07` (-7%)
- Looked like it was working (`source=optuna`) but wasn't!

**How to prevent:**
1. Define field names as constants
2. Import constants in both optimizer and feeder
3. Add validation logs showing actual values loaded

### Lesson 2: Mixed Data Breaks Pivot Queries

**What happened:**
- InfluxDB had both old (`stop_loss`) and new (`stop_loss_position_pct`) data
- Pivot query randomly returned old or new data
- Sometimes worked, sometimes didn't (non-deterministic!)

**How to prevent:**
1. Delete old bucket when changing schema
2. Filter queries for specific field names only
3. Version your buckets (OPTUNA_V1, OPTUNA_V2)

### Lesson 3: Integer/Float Schema Collisions

**What happened:**
- First save: `trades_count = 100` (integer)
- Second save: `trades_count = 100.0` (float)
- InfluxDB error: "field type conflict"
- All optimization failed!

**How to prevent:**
- Always cast to float: `float(value)`
- Document schema with types
- Test with both int-like and decimal values

### Lesson 4: `last()` is Unreliable with Mixed Data

**What happened:**
- Used `|> last()` to get latest record
- With mixed old/new data, returned unpredictable results
- Sometimes old data, sometimes new data

**How to prevent:**
- Use `|> sort(columns: ["_time"], desc: true) |> limit(n: 1)`
- This explicitly gets the absolute latest record

### Lesson 5: Fallback Values Hide Problems

**What happened:**
- Default fallback: `record.values.get('field_name', -0.07)`
- When field not found, used `-0.07`
- Logs showed `source=optuna` (misleading!)
- No error, just wrong values

**How to prevent:**
1. Use None as fallback: `record.values.get('field_name', None)`
2. Check if None and log warning
3. Better: Fail loudly if critical params missing

## 📝 FEEDER OPTIMIZATION IMPLEMENTATION TEMPLATE

### Step 1: Define Schema

```python
# feeder_optuna_schema.py

INFLUX_MEASUREMENT = "optimized_parameters"

# Field names as constants (single source of truth!)
FIELD_LEARNING_RATE = "learning_rate"
FIELD_NUM_ESTIMATORS = "num_estimators"
FIELD_MAX_DEPTH = "max_depth"
FIELD_QUANTILE_LONG = "quantile_long"
FIELD_QUANTILE_SHORT = "quantile_short"

# Metrics
FIELD_BACKTESTED_PROFIT = "backtested_profit"
FIELD_WIN_RATE = "win_rate"
FIELD_SHARPE_RATIO = "sharpe_ratio"

# All field names for easy validation
ALL_FIELDS = [
    FIELD_LEARNING_RATE,
    FIELD_NUM_ESTIMATORS,
    FIELD_MAX_DEPTH,
    FIELD_QUANTILE_LONG,
    FIELD_QUANTILE_SHORT,
    FIELD_BACKTESTED_PROFIT,
    FIELD_WIN_RATE,
    FIELD_SHARPE_RATIO,
]
```

### Step 2: Optimizer Saves Data

```python
from feeder_optuna_schema import *

def save_params(feeder_id, pair, params):
    point = Point(INFLUX_MEASUREMENT)
    point.tag("feeder_id", feeder_id)
    point.tag("pair", pair)
    
    # Use constants!
    point.field(FIELD_LEARNING_RATE, float(params.learning_rate))
    point.field(FIELD_NUM_ESTIMATORS, float(params.num_estimators))
    point.field(FIELD_MAX_DEPTH, float(params.max_depth))
    # ...
    
    client.write_api().write(bucket=f"OPTUNA_{feeder_id}", record=point)
```

### Step 3: Feeder Loads Data

```python
from feeder_optuna_schema import *

def load_params(feeder_id):
    # Build field filter from constants
    field_filter = " or ".join([f'r._field == "{f}"' for f in ALL_FIELDS])
    
    query = f'''
    from(bucket: "OPTUNA_{feeder_id}")
    |> range(start: -90d)
    |> filter(fn: (r) => r._measurement == "{INFLUX_MEASUREMENT}")
    |> filter(fn: (r) => {field_filter})
    |> sort(columns: ["_time"], desc: true)
    |> limit(n: 1)
    |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")
    '''
    
    result = client.query_api().query(query)
    
    params = {}
    for table in result:
        for record in table.records:
            pair = record.values.get('pair')
            
            # Use constants!
            params[pair] = {
                'learning_rate': record.values.get(FIELD_LEARNING_RATE),
                'num_estimators': int(record.values.get(FIELD_NUM_ESTIMATORS)),
                # ...
            }
            
            # Validate
            if params[pair]['learning_rate'] is None:
                logger.error(f"❌ Missing {FIELD_LEARNING_RATE} for {pair}!")
                
    return params
```

## ✅ FINAL CHECKLIST FOR FEEDER OPTIMIZATION

Before deploying Optuna for feeders:

- [ ] Schema defined with field names as constants
- [ ] All numeric values cast to `float()`
- [ ] Optimizer and feeder import same constants
- [ ] Query includes field name filter
- [ ] Query uses `sort() + limit(1)` not just `last()`
- [ ] Validation logging shows actual loaded values
- [ ] Tested with sample data
- [ ] Verified field names match in InfluxDB
- [ ] Documented schema in markdown file
- [ ] Added debug logging for troubleshooting

## 🎯 SUCCESS CRITERIA

You'll know it's working when:

✅ Feeder logs show: "Loaded Optuna params for X pairs"  
✅ Debug logs show actual parameter values (not defaults)  
✅ Parameters vary per pair (not all identical)  
✅ Values match what optimizer saved to InfluxDB  
✅ No schema collision errors  
✅ Consistent behavior across restarts  

## 📚 REFERENCE DOCUMENTS

- `BOT2_OPTUNA_FIELD_NAME_MISMATCH_FIX_COMPLETE_2025-11-29.md` - Detailed debugging session
- `BOT2_OPTUNA_SCHEMA_COLLISION_FIX_COMPLETE.md` - Integer/Float type issues
- `OPTUNA_DETAILED_LOGIC_REVIEW_2025-11-28.md` - Optuna integration patterns

---

**Remember:** The devil is in the details! Field name consistency is EVERYTHING.

Use this guide as a reference when implementing Optuna for feeders, and you'll avoid the painful debugging we went through with BOT2! 🚀
