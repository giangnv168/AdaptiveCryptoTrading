# Optuna Per-Bot Optimization - Usage Guide
**Date:** November 28, 2025  
**Version:** 2.0.0 (Per-Bot)

---

## 🔄 WHAT CHANGED

### Previous Multi-Bot Weighted Approach
```
All bots' trades combined with weights:
- BOT3 trades: weight 3.0
- BOT1 signals: weight 1.0
- BOT2 signals: weight 1.0

Result: One set of parameters learned from all bots
```

### **NEW** Per-Bot Approach
```
Each bot optimized independently:
- BOT3 parameters ← BOT3 closed trades ONLY
- BOT2 parameters ← BOT2 closed trades ONLY
- BOT1 parameters ← BOT1 closed trades ONLY

Result: Each bot gets its own optimized parameters
```

---

## 📁 NEW FILES CREATED

### 1. Core Optimizer Engine
**File:** `user_data/services/optuna_optimizer_per_bot.py`

**Key Changes:**
- Removed multi-bot weighting system
- Added `target_bot` parameter
- Simplified trade loading (one bot at a time)
- Stores `optimized_for_bot` tag in InfluxDB

### 2. Runner Script
**File:** `user_data/services/run_optuna_per_bot.py`

**Features:**
- `--bot` parameter (BOT1, BOT2, or BOT3)
- Per-bot logging
- Per-bot config loading
- Clear output showing which bot is being optimized

---

## 🚀 HOW TO USE

### Option 1: Optimize for BOT3

```bash
# Full optimization (all pairs, 100 trials each)
python3 user_data/services/run_optuna_per_bot.py --bot BOT3

# Custom trials
python3 user_data/services/run_optuna_per_bot.py --bot BOT3 --n-trials 50

# Specific pairs only
python3 user_data/services/run_optuna_per_bot.py --bot BOT3 \
    --pairs BTC/USDT:USDT ETH/USDT:USDT SOL/USDT:USDT

# Test run (dry-run, no saving)
python3 user_data/services/run_optuna_per_bot.py --bot BOT3 --dry-run
```

**Logs:** `user_data/logs/optuna_optimizer_BOT3.log`

### Option 2: Optimize for BOT2

```bash
# Full optimization
python3 user_data/services/run_optuna_per_bot.py --bot BOT2

# Fewer trials (faster)
python3 user_data/services/run_optuna_per_bot.py --bot BOT2 --n-trials 50

# 60 days of data instead of 90
python3 user_data/services/run_optuna_per_bot.py --bot BOT2 --days-back 60
```

**Logs:** `user_data/logs/optuna_optimizer_BOT2.log`

### Option 3: Optimize for BOT1

```bash
# Full optimization
python3 user_data/services/run_optuna_per_bot.py --bot BOT1
```

**Logs:** `user_data/logs/optuna_optimizer_BOT1.log`

---

## 📊 EXPECTED OUTPUT

### Console Output Example

```
================================================================================
🚀 BOT2 Optuna Parameter Optimization (Per-Bot Version)
================================================================================
Started: 2025-11-28 01:00:00
Bot: BOT2
Data source: BOT2 closed trades ONLY
Config: user_data/config.json

📊 Pairs to optimize: 150
   - BTC/USDT:USDT
   - ETH/USDT:USDT
   - SOL/USDT:USDT
   ... and 147 more

📡 Connecting to InfluxDB...
✅ InfluxDB connection established

🔧 Initializing Optuna optimizer for BOT2...
✅ Optuna Parameter Optimizer initialized (Per-Bot Version)
   Target bot: BOT2
   Data source: BOT2 closed trades ONLY
   Storage: InfluxDB bucket 'OPTUNA_PARAMS'
✅ Optimizer initialized

🎯 Starting optimization...
   Days back: 90
   Trials per pair: 100
   Total pairs: 150
   Estimated time: 375.0 minutes

================================================================================
🎯 Optimizing BTC/USDT:USDT (1/150 = 0.7%)
================================================================================

📊 Loaded 234 BOT2 trades for BTC/USDT:USDT
📊 BTC/USDT:USDT: 234 BOT2 trades (train: 163, test: 71)
   ATR: 0.0234

   Trial 1/100: params=[SL=-2.45%, TP=3.21%, VM=1.12] profit=0.54% | Best so far=N/A | ETA=15.3min
   Trial 10/100: params=[SL=-1.89%, TP=5.67%, VM=0.95] profit=1.23% | Best so far=1.23% | ETA=13.7min
   ...
   Trial 100/100: params=[SL=-2.01%, TP=4.55%, VM=1.05] profit=2.11% | Best so far=2.45% | ETA=0.0min

🏁 Completed 100 trials in 15.4 minutes

🎯 Best params: SL=-2.01%, TP=4.55%, VM=1.05
📈 Train profit: 2.45%, Test profit: 1.89%

✅ BTC/USDT:USDT: SL=-2.01%, TP=4.55%, Profit=2.11%

... (149 more pairs)

================================================================================
✅ OPTIMIZATION COMPLETE
================================================================================
Bot: BOT2
Optimized pairs: 147

📊 RESULTS SUMMARY:
--------------------------------------------------------------------------------

BTC/USDT:USDT:
  Stop Loss: -2.01% (position-level)
  Take Profit: 4.55% (position-level)
  Volatility Mult: 1.05
  Backtested Profit: 2.11%
  Win Rate: 68.0%
  Profit Factor: 2.34
  Sharpe Ratio: 1.89
  Trades: 234 BOT2 trades
  Train Profit: 2.45%
  Test Profit: 1.89%

... (146 more pairs)

================================================================================
💾 Results saved to: InfluxDB bucket 'OPTUNA_PARAMS'
   Tag: optimized_for_bot=BOT2
Completed: 2025-11-28 06:44:12
================================================================================
```

---

## 💾 DATA STORAGE

### InfluxDB Structure

**Bucket:** `OPTUNA_PARAMS`  
**Measurement:** `optimized_parameters`

**Tags:**
- `pair`: "BTC/USDT:USDT"
- **`optimized_for_bot`: "BOT2"** ← NEW TAG!

**Fields:**
- `stop_loss`: -0.0201
- `take_profit`: 0.0455
- `volatility_multiplier`: 1.05
- `backtested_profit`: 0.0211
- `win_rate`: 0.68
- `profit_factor`: 2.34
- `sharpe_ratio`: 1.89
- `trades_count`: 234
- `train_profit`: 0.0245
- `test_profit`: 0.0189
- `optimization_trials`: 100

### Query by Bot

**Get BOT2 parameters:**
```flux
from(bucket: "OPTUNA_PARAMS")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> filter(fn: (r) => r.optimized_for_bot == "BOT2")
|> group(columns: ["pair"])
|> last()
```

**Get BOT3 parameters:**
```flux
from(bucket: "OPTUNA_PARAMS")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> filter(fn: (r) => r.optimized_for_bot == "BOT3")
|> group(columns: ["pair"])
|> last()
```

---

## 🔍 HOW BOT2 READS PARAMETERS

### Update BOT2 Strategy

**Current BOT2 strategy needs to filter by bot tag:**

```python
def _load_optuna_params(self):
    """Load Optuna parameters from InfluxDB."""
    query = f'''
    from(bucket: "OPTUNA_PARAMS")
    |> range(start: -90d)
    |> filter(fn: (r) => r._measurement == "optimized_parameters")
    |> filter(fn: (r) => r.optimized_for_bot == "BOT2")  # ← ADD THIS LINE
    |> group(columns: ["pair"])
    |> last()
    '''
    
    # Parse and load parameters...
```

**Same for BOT1 and BOT3:**
- BOT1 filters: `optimized_for_bot == "BOT1"`
- BOT3 filters: `optimized_for_bot == "BOT3"`

---

## 📈 VERIFICATION

### Step 1: Run Optimization for BOT2

```bash
python3 user_data/services/run_optuna_per_bot.py --bot BOT2
```

### Step 2: Check InfluxDB

```bash
# Check if parameters exist for BOT2
influx query '
from(bucket: "OPTUNA_PARAMS")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> filter(fn: (r) => r.optimized_for_bot == "BOT2")
|> count()
'
```

**Expected:** Should show parameters for BOT2

### Step 3: Update BOT2 Strategy

Add the `optimized_for_bot` filter to BOT2's `_load_optuna_params()` method.

### Step 4: Restart BOT2

```bash
sshpass -p 'ArtiSky@7' ssh ubuntu@192.168.3.72 \
  "echo 'ArtiSky@7' | sudo -S systemctl restart freqtrade"
```

### Step 5: Check BOT2 Logs

```bash
sshpass -p 'ArtiSky@7' ssh ubuntu@192.168.3.72 \
  "tail -50 /home/ubuntu/freqtrade/bot2.log | grep -i optuna"
```

**Expected:**
```
📊 Loaded Optuna params for 147 pairs  # Not 0!
```

### Step 6: Monitor Trades

New trades should show:
- Exit reason: `roi_optuna` (not `roi_fallback`)
- Exit reason: `stop_loss_optuna` (not `cutloss_fallback`)

---

## ⚡ PERFORMANCE

### Optimization Speed (Per Bot)

**BOT2 with 90 days of data:**
- Pairs with ≥20 trades: ~2-3 min per pair
- 150 pairs: ~5-6 hours total
- Memory usage: ~8GB

**BOT3 with 90 days of data:**
- More trades = slightly faster per pair
- Same total time: ~5-6 hours

**BOT1 with 90 days of data:**
- May have fewer trades
- Some pairs might skip (< 20 trades)
- Total time: ~3-4 hours

### Tips for Faster Optimization

1. **Reduce trials:**
   ```bash
   --n-trials 50  # Instead of 100
   ```

2. **Fewer days:**
   ```bash
   --days-back 60  # Instead of 90
   ```

3. **Specific pairs:**
   ```bash
   --pairs BTC/USDT:USDT ETH/USDT:USDT  # Top pairs only
   ```

4. **Run overnight:**
   ```bash
   nohup python3 user_data/services/run_optuna_per_bot.py --bot BOT2 &
   ```

---

## 🔧 TROUBLESHOOTING

### Issue: "Only X BOT2 trades for pair, need 20"

**Cause:** Not enough BOT2 trades for that pair

**Solutions:**
1. Wait for more trades
2. Reduce `MIN_TRADES_REQUIRED` in code
3. Skip that pair (automatically skipped)

### Issue: "Test profit negative, parameters may be overfitted"

**Cause:** Parameters don't generalize to test set

**Solutions:**
1. Increase `--n-trials` (more searching)
2. Increase `--days-back` (more data)
3. Accept that pair is not optimizable yet

### Issue: BOT2 still uses fallback after optimization

**Checks:**
1. Did optimization complete successfully?
2. Did you add the `optimized_for_bot` filter to BOT2 strategy?
3. Did you restart BOT2 after optimization?
4. Check InfluxDB has parameters with tag `optimized_for_bot=BOT2`

---

## 📋 MIGRATION CHECKLIST

### From Multi-Bot to Per-Bot

- [ ] Run optimizer for BOT3:
  ```bash
  python3 user_data/services/run_optuna_per_bot.py --bot BOT3
  ```

- [ ] Run optimizer for BOT2:
  ```bash
  python3 user_data/services/run_optuna_per_bot.py --bot BOT2
  ```

- [ ] Run optimizer for BOT1:
  ```bash
  python3 user_data/services/run_optuna_per_bot.py --bot BOT1
  ```

- [ ] Update BOT1 strategy to filter by `optimized_for_bot == "BOT1"`
- [ ] Update BOT2 strategy to filter by `optimized_for_bot == "BOT2"`
- [ ] Update BOT3 strategy to filter by `optimized_for_bot == "BOT3"`
- [ ] Restart all bots
- [ ] Verify parameters loaded (check logs)
- [ ] Monitor new trades (exit reasons should be `_optuna` not `_fallback`)

---

## 🎯 SUMMARY

### What This Solves

**Before:**
- BOT2 had no parameters (optimizer never ran)
- Even if run, would use multi-bot weighted approach
- BOT2 using fallback values

**After:**
- Each bot gets parameters from its own trades
- BOT2 parameters optimized from BOT2 performance
- No cross-contamination between bots
- Each bot's parameters reflect its actual trading style

### Next Steps

1. Run per-bot optimization for BOT2:
   ```bash
   python3 user_data/services/run_optuna_per_bot.py --bot BOT2
   ```

2. Update BOT2 strategy to filter by bot tag

3. Restart BOT2 and verify it loads parameters

4. Monitor trades - should see `_optuna` exit reasons!

---

**STATUS:** Per-bot optimization system ready to use  
**BENEFITS:** Each bot optimized for its own trading style  
**IMPACT:** BOT2 will finally use optimized parameters!
