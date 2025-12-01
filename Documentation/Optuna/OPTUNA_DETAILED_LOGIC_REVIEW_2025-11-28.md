# Optuna Optimizer Detailed Logic Review
**Date:** November 28, 2025  
**Purpose:** Complete breakdown of Optuna optimization logic for BOT2 parameter generation

---

## 📋 CURRENT OPTUNA SERVICES OVERVIEW

### Available Services

```
optuna-optimizer.service   (590 bytes)  - Systemd oneshot service
optuna-optimizer.timer     (210 bytes)  - Systemd timer (scheduled execution)
optuna-continuous.service  (1,056 bytes) - DEPRECATED (should be deleted)
```

### Available Scripts

```
optuna_optimizer_service.py      (38KB, Nov 26 10:08) ⭐ LATEST - Core optimizer engine
run_optuna_optimizer.py          (8KB, Nov 22 23:54)  ⭐ LATEST - Service runner
optuna_standalone.py             (14KB, Nov 23 18:34) - Standalone version (simpler)
optuna_multibot_enhanced.py      (21KB, Nov 23 18:36) - Multi-bot version
optuna_continuous_optimizer.py   (15KB, Nov 25 23:13) - Continuous service (deprecated)
optuna_zec_test.py               (1.3KB)               - Testing script
```

### **RECOMMENDED LATEST VERSION**

**Primary:** `optuna_optimizer_service.py` (Nov 26, 38KB)  
**Runner:** `run_optuna_optimizer.py` (Nov 22, 8KB)

These two files work together as the official Optuna optimization system.

---

## 🎯 COMPLETE WORKFLOW

### High-Level Flow

```
┌─────────────────────────────────────────────────┐
│  1. RUN OPTIMIZER                               │
│     python3 run_optuna_optimizer.py             │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  2. LOAD CONFIGURATION                          │
│     - optuna_config.json                        │
│     - config_BOT3.json (for pair whitelist)     │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  3. CONNECT TO INFLUXDB                         │
│     - InfluxDBTradeReader (BOT3 trades)         │
│     - Write to OPTUNA_PARAMS bucket             │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  4. INITIALIZE OPTIMIZER                        │
│     - OptunaParameterOptimizer                  │
│     - Multi-bot mode (BOT1/BOT2/BOT3)           │
│     - Weights: BOT3=3.0, BOT1=1.0, BOT2=1.0     │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  5. FOR EACH PAIR (e.g., BTC/USDT)              │
│     a. Load historical trades (90 days)         │
│     b. Normalize leverage (position-level)      │
│     c. Load OHLCV data (volatility)             │
│     d. Calculate ATR (Average True Range)       │
│     e. Split train/test (70%/30%)               │
│     f. Run Optuna trials (100 trials)           │
│     g. Validate on test set                     │
│     h. Save best parameters to InfluxDB         │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  6. SAVE RESULTS TO INFLUXDB                    │
│     Bucket: OPTUNA_PARAMS                       │
│     Measurement: optimized_parameters           │
│     Fields: stop_loss, take_profit, etc.        │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  7. BOT2 READS PARAMETERS                       │
│     - Loads from OPTUNA_PARAMS every 30 min    │
│     - Uses per-pair optimized values            │
│     - Falls back to defaults if not found       │
└─────────────────────────────────────────────────┘
```

---

## 🔬 DETAILED LOGIC BREAKDOWN

### **1. OptunaParameterOptimizer Class**

**Location:** `user_data/services/optuna_optimizer_service.py`

#### Key Configuration

```python
# Search space for parameters
STOP_LOSS_RANGE = (-0.03, -0.01)      # -3% to -1% (position-level)
TAKE_PROFIT_RANGE = (0.01, 0.20)      # +1% to +20% (position-level)
VOLATILITY_MULT_RANGE = (0.8, 1.5)    # ATR multiplier

# Critical constraints (prevents unprofitable parameters)
MIN_RISK_REWARD_RATIO = 0.8           # Min TP/SL ratio
MIN_TAKE_PROFIT = 0.015               # Min 1.5% TP

# Data requirements
MIN_TRADES_REQUIRED = 20              # Min trades needed for optimization
TRAIN_TEST_SPLIT = 0.7                # 70% train, 30% test
MAX_CANDLES_TO_WALK = 12              # Max hold time: 60 minutes (12 × 5min)

# Multi-bot weights (when enabled)
BOT3_WEIGHT = 3.0                     # Highest (actual executed trades)
BOT1_WEIGHT = 1.0                     # Lower (signals only)
BOT2_WEIGHT = 1.0                     # Lower (signals only)
```

#### Initialization

```python
def __init__(self, influx_reader, config_path, use_multi_bot=True, ...):
    """
    Sets up:
    1. InfluxDB reader (for loading trade history)
    2. FreqTrade config (for pair whitelist, OHLCV data)
    3. Multi-bot configuration (weights)
    4. OHLCV cache (for volatility calculations)
    5. InfluxDB writer (for saving results)
    """
```

**Key Features:**
- No dependency on FreqTrade ExchangeResolver (avoids config issues)
- Uses InfluxDB for all trade data
- Loads OHLCV from local feather files (`datadir`)
- Supports multi-bot weighted optimization

---

### **2. Main Optimization Loop**

#### `optimize_all_pairs()`

```python
def optimize_all_pairs(pairs, days_back=90, n_trials=100):
    """
    For each pair:
    1. Show progress (X/Y pairs, ETA in minutes)
    2. Call optimize_pair()
    3. Track success/failure
    4. Save all results to InfluxDB
    5. Print summary
    """
```

**Progress Tracking:**
```
🎯 Optimizing BTC/USDT:USDT (1/150 = 0.7%)
⏱️ Progress: 0 completed, avg 0.0 min/pair
📊 ETA: 0.0 minutes remaining

📊 BTC/USDT:USDT: 234 trades (train: 163, test: 71)
   ATR: 0.0234

   Trial 1/100: params=[SL=-2.45%, TP=3.21%, VM=1.12] profit=0.54% | Best so far=N/A | ETA=15.3min
   Trial 10/100: params=[SL=-1.89%, TP=5.67%, VM=0.95] profit=1.23% | Best so far=1.23% | ETA=13.7min
   ...
   Trial 100/100: params=[SL=-2.01%, TP=4.55%, VM=1.05] profit=2.11% | Best so far=2.45% | ETA=0.0min

🏁 Completed 100 trials in 15.4 minutes

🎯 Best params: SL=-2.01%, TP=4.55%, VM=1.05
📈 Train profit: 2.45%, Test profit: 1.89%

✅ BTC/USDT:USDT: SL=-2.01%, TP=4.55%, Profit=2.11%
```

---

### **3. Per-Pair Optimization**

#### `optimize_pair(pair, days_back, n_trials)`

**Step-by-step:**

**Step 1: Load & Normalize Trades**
```python
trades = _load_and_normalize_trades(pair, days_back)

# What happens:
# - Loads trades from InfluxDB (BOT3 or multi-bot)
# - CRITICAL FIX: Normalizes to position-level
#   position_profit = account_profit / leverage
# - Applies weights (if multi-bot enabled)
# - Returns list of normalized trades
```

**Example normalized trade:**
```python
{
    'pair': 'BTC/USDT:USDT',
    'open_rate': 50000.0,
    'close_rate': 51000.0,
    'leverage': 3.0,
    'account_profit': 0.06,        # 6% account-level (3x leverage)
    'position_profit': 0.02,       # 2% position-level (normalized)
    'weight': 3.0,                 # BOT3 weight
    'source': 'BOT3',
    'is_short': False,
    'open_date': '2025-11-01T10:00:00',
    ...
}
```

**Step 2: Load OHLCV Data**
```python
ohlcv = _load_ohlcv(pair, days_back)

# What happens:
# - Loads from FreqTrade data directory
# - Format: feather (not JSON)
# - Candle type: futures (not spot)
# - Timeframe: 5m (matches BOT3 strategy)
# - Caches for performance
```

**Step 3: Calculate Volatility (ATR)**
```python
atr = _calculate_atr(ohlcv, period=14)

# What happens:
# - Calculates Average True Range (14 periods)
# - Normalizes by current price
# - Used to adjust stops for volatile pairs
# - Default: 0.05 (5% volatility) if calculation fails
```

**Step 4: Train/Test Split**
```python
split_idx = int(len(trades) * 0.7)
train_trades = trades[:split_idx]   # 70% for optimization
test_trades = trades[split_idx:]     # 30% for validation
```

**Step 5: Run Optuna Trials**
```python
study = optuna.create_study(direction='maximize')
study.optimize(objective_function, n_trials=100)

# What happens in each trial:
# 1. Optuna suggests parameters (SL, TP, VM)
# 2. Check constraints (R/R ratio, min TP)
# 3. Backtest on training data
# 4. Return profit (or -inf if constraints violated)
# 5. Optuna learns and suggests better parameters
```

**Step 6: Validate on Test Set**
```python
train_profit = study.best_value
test_profit = _backtest_parameters(test_trades, ...)

# CRITICAL: Test profit must be positive!
if test_profit <= 0:
    return None  # Reject overfitted parameters
```

**Step 7: Return Optimized Parameters**
```python
return OptimizedParameters(
    pair=pair,
    stop_loss=-0.0201,              # -2.01% position-level
    take_profit=0.0455,             # 4.55% position-level
    volatility_multiplier=1.05,
    backtested_profit=0.0211,       # 2.11% total profit
    win_rate=0.68,                  # 68% win rate
    profit_factor=2.34,
    sharpe_ratio=1.89,
    trades_count=234,
    train_profit=0.0245,            # 2.45% on training set
    test_profit=0.0189,             # 1.89% on test set (positive!)
    last_optimized='2025-11-28T00:00:00',
    optimization_trials=100
)
```

---

### **4. Objective Function (Optuna Trial)**

#### `_objective_function(trial, pair, trades, ohlcv, atr)`

**What Optuna Does:**

```python
# Optuna suggests parameters to try
stop_loss = trial.suggest_float('stop_loss', -0.03, -0.01)      # e.g., -0.0234
take_profit = trial.suggest_float('take_profit', 0.01, 0.20)    # e.g., 0.0567
volatility_mult = trial.suggest_float('volatility_multiplier', 0.8, 1.5)  # e.g., 1.12

# CRITICAL CONSTRAINT: Risk/Reward Ratio
risk_reward = take_profit / abs(stop_loss)
# Example: 0.0567 / 0.0234 = 2.42

if risk_reward < 0.8:
    return float('-inf')  # Reject this trial

# CRITICAL CONSTRAINT: Minimum Take Profit
if take_profit < 0.015:  # Less than 1.5%
    return float('-inf')  # Reject this trial

# Backtest with these parameters
total_profit = _backtest_parameters(trades, ohlcv, stop_loss, take_profit, volatility_mult, atr)

return total_profit  # Optuna maximizes this
```

**Why Constraints are Critical:**

Without constraints, Optuna might find:
- **SL=-1%, TP=1%** → R/R=1.0 → Seems good
- But: Small TP means **many small wins**, **few big losses**
- Result: **Unprofitable over time!**

With constraints:
- **SL=-2%, TP=4%** → R/R=2.0 → Good R/R
- **TP > 1.5%** → Ensures meaningful profits
- Result: **Profitable trades!**

---

### **5. Backtesting Logic**

#### `_backtest_parameters(trades, ohlcv, stop_loss, take_profit, volatility_mult, atr)`

**Step 1: Adjust Stops for Volatility**
```python
adjusted_sl = stop_loss * volatility_mult * (1 + atr)
adjusted_tp = take_profit * volatility_mult * (1 + atr)

# Example for volatile pair (ATR=0.05):
# SL = -0.02 * 1.1 * (1 + 0.05) = -0.0231 (-2.31%)
# TP =  0.04 * 1.1 * (1 + 0.05) =  0.0462 (4.62%)
```

**Step 2: Simulate Each Trade**
```python
for trade in trades:
    sim = _simulate_trade(trade, ohlcv, adjusted_sl, adjusted_tp)
    
    # Apply weight (multi-bot mode)
    weight = trade.get('weight', 1.0)
    weighted_profit = sim.profit_pct * weight
    
    total_weighted_profit += weighted_profit
    total_weight += weight
```

**Step 3: Return Weighted Average**
```python
return total_weighted_profit / total_weight
```

---

### **6. Trade Simulation**

#### `_simulate_trade(trade, ohlcv, stop_loss, take_profit)`

**Logic:**

```python
# Get actual profit from historical trade
actual_profit = trade['position_profit']  # e.g., 0.0234 (2.34%)

# Check if stops would have been hit
hit_stop = actual_profit <= stop_loss     # e.g., 2.34% > -2% → False
hit_target = actual_profit >= take_profit # e.g., 2.34% < 4% → False

# Determine simulated exit
if hit_stop:
    exit_profit = stop_loss      # Stopped out
elif hit_target:
    exit_profit = take_profit    # Target hit
else:
    exit_profit = actual_profit  # Trade ended naturally

return TradeSimulation(
    profit_pct=exit_profit,      # e.g., 2.34%
    is_win=(exit_profit > 0),    # True
    hit_stop=False,
    hit_target=False,
    ...
)
```

**Note:** Simplified implementation uses actual close. Full version would walk through OHLCV candles to check exact stop/target hits.

---

### **7. Multi-Bot Weighted Optimization**

#### How It Works

**Enabled by default:**
```python
use_multi_bot = True
bot3_weight = 3.0  # Actual executed trades (highest quality)
bot1_weight = 1.0  # LONG signals (supplementary)
bot2_weight = 1.0  # SHORT signals (supplementary)
```

**Loading Trades:**
```python
# BOT3: Executed trades (100 trades × 3.0 weight = 300 weighted samples)
bot3_trades = influx.get_recent_closed_trades(bot_name='BOT3', ...)
for trade in bot3_trades:
    trade['weight'] = 3.0
    trade['source'] = 'BOT3'

# BOT1: LONG signals (50 signals × 1.0 weight = 50 weighted samples)
bot1_signals = influx.get_recent_closed_trades(bot_name='BOT1', ...)
for signal in bot1_signals:
    signal['weight'] = 1.0
    signal['source'] = 'BOT1'

# BOT2: SHORT signals (50 signals × 1.0 weight = 50 weighted samples)
bot2_signals = influx.get_recent_closed_trades(bot_name='BOT2', ...)
for signal in bot2_signals:
    signal['weight'] = 1.0
    signal['source'] = 'BOT2'

# Total: 200 trades, but 400 weighted samples (BOT3 has 3x influence)
```

**Why This Matters:**

- **BOT3 trades** are real, executed with actual market conditions
- **BOT1/BOT2 signals** provide additional data but weren't executed
- **Weighting** ensures optimization focuses on actual performance while learning from signals

---

### **8. Saving Results to InfluxDB**

#### `save_results(results)`

**Data Structure:**

```python
# InfluxDB Point structure
Bucket: OPTUNA_PARAMS
Measurement: optimized_parameters

Tags:
  - pair: "BTC/USDT:USDT"

Fields:
  - stop_loss: -0.0201          (float)
  - take_profit: 0.0455         (float)
  - volatility_multiplier: 1.05 (float)
  - backtested_profit: 0.0211   (float)
  - win_rate: 0.68              (float)
  - profit_factor: 2.34         (float)
  - sharpe_ratio: 1.89          (float)
  - trades_count: 234           (int)
  - train_profit: 0.0245        (float)
  - test_profit: 0.0189         (float)
  - optimization_trials: 100    (int)
  - multi_bot_enabled: true     (bool)
  - bot3_weight: 3.0            (float)
  - bot1_weight: 1.0            (float)
  - bot2_weight: 1.0            (float)

Timestamp: 2025-11-28T00:00:00Z
```

**Fallback:**
- If InfluxDB write fails → Save to JSON file
- Location: `user_data/services/optimized_parameters.json`

---

### **9. BOT2 Reading Parameters**

#### How BOT2 Uses These Parameters

**In BOT2 Strategy:**
```python
def _load_optuna_params(self):
    query = '''
    from(bucket: "OPTUNA_PARAMS")
    |> range(start: -90d)
    |> filter(fn: (r) => r._measurement == "optimized_parameters")
    |> group(columns: ["pair"])
    |> last()
    '''
    
    for table in result:
        for record in table.records:
            pair = record.values.get('pair')
            self.optuna_params[pair] = {
                'take_profit_position_pct': record.values.get('take_profit'),
                'stop_loss_position_pct': record.values.get('stop_loss'),
                ...
            }

def custom_exit(self, pair, trade, current_profit, ...):
    params = self._get_exit_params_for_pair(pair)
    
    # Convert position-level to account-level
    leverage = trade.leverage
    stop_account = params['stop_loss_position_pct'] * leverage
    target_account = params['take_profit_position_pct'] * leverage
    
    if current_profit <= stop_account:
        return 'stop_loss_optuna'  # Not 'cutloss_fallback'!
    
    if current_profit >= target_account:
        return 'roi_optuna'  # Not 'roi_fallback'!
```

---

## 🚀 HOW TO RUN THE OPTIMIZER

### Option 1: Manual Run (Recommended for First Time)

```bash
# Run the optimizer
python3 user_data/services/run_optuna_optimizer.py

# Check logs
tail -f user_data/logs/optuna_optimizer.log
```

### Option 2: Specific Pairs Only

```bash
# Optimize only BTC and ETH
python3 user_data/services/run_optuna_optimizer.py \
    --pairs BTC/USDT:USDT ETH/USDT:USDT
```

### Option 3: Dry Run (Test Without Saving)

```bash
# Run but don't save results
python3 user_data/services/run_optuna_optimizer.py --dry-run
```

### Option 4: Install as Service

```bash
# Copy service file
sudo cp optuna-optimizer.service /etc/systemd/system/

# Reload systemd
sudo systemctl daemon-reload

# Enable service
sudo systemctl enable optuna-optimizer.service

# Run once
sudo systemctl start optuna-optimizer.service

# Check status
sudo systemctl status optuna-optimizer.service
```

### Option 5: Schedule with Timer

```bash
# Copy timer file
sudo cp optuna-optimizer.timer /etc/systemd/system/

# Enable timer (runs periodically)
sudo systemctl enable --now optuna-optimizer.timer

# Check timer status
sudo systemctl list-timers optuna-optimizer.timer
```

---

## 📊 EXPECTED OUTPUT

### Console Output

```
================================================================================
🚀 BOT3 Optuna Parameter Optimization
================================================================================
Started: 2025-11-28 00:30:00

📊 Pairs to optimize: 150
   - BTC/USDT:USDT
   - ETH/USDT:USDT
   - ...

📡 Connecting to InfluxDB...
✅ InfluxDB connection established

🔧 Initializing Optuna optimizer...
✅ Optimizer initialized

🎯 Starting optimization...
   Days back: 90
   Trials per pair: 100
   Train/test split: 0.7
   Total pairs: 150
   Estimated time: 375.0 minutes

================================================================================
🎯 Optimizing BTC/USDT:USDT (1/150 = 0.7%)
================================================================================

📊 BTC/USDT:USDT: 234 trades (train: 163, test: 71)
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

⏱️ Total optimization time: 374.2 minutes

================================================================================
✅ OPTIMIZATION COMPLETE
================================================================================
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
  Trades: 234
  Train Profit: 2.45%
  Test Profit: 1.89%

... (146 more pairs)

================================================================================
💾 Results saved to: InfluxDB bucket 'OPTUNA_PARAMS'
Completed: 2025-11-28 06:44:12
================================================================================
```

---

## 🔑 KEY FEATURES

### 1. Position-Level Learning ✅
**Fixes leverage normalization bug**

```python
# Before (WRONG):
account_profit = 0.06  # 6% with 3x leverage
# Would optimize around 6%, but that's account-level!

# After (CORRECT):
position_profit = 0.06 / 3.0 = 0.02  # 2% position-level
# Optimizes around 2%, then multiplies by leverage when used
```

### 2. Volatility Awareness ✅
**Adjusts stops for market conditions**

```python
# Stable pair (ATR=0.01):
adjusted_sl = -0.02 * 1.0 * (1 + 0.01) = -0.0202  # Tight stops OK

# Volatile pair (ATR=0.10):
adjusted_sl = -0.02 * 1.0 * (1 + 0.10) = -0.0220  # Wider stops needed
```

### 3. Walk-Forward Validation ✅
**Prevents overfitting**

```python
# Train on 70% → Test on 30%
# Only save if test profit > 0
```

### 4. Risk/Reward Constraints ✅
**Ensures profitability**

```python
# Rejects bad R/R ratios
# Rejects tiny take profits
# Forces sensible parameters
```

### 5. Multi-Bot Learning ✅
**Learns from multiple sources**

```python
# BOT3: 3x weight (real trades)
# BOT1: 1x weight (LONG signals)
# BOT2: 1x weight (SHORT signals)
```

---

## 🐛 TROUBLESHOOTING

### Issue: No Parameters Generated

**Check 1: Insufficient trades**
```bash
# Need at least 20 trades per pair
# Check: tail -f user_data/logs/optuna_optimizer.log | grep "Only"
# Output: "⚠️  Only 15 trades for XYZ/USDT, need 20"
```

**Solution:** Wait for more trades or reduce `MIN_TRADES_REQUIRED`

**Check 2: Test profit negative**
```bash
# Check logs for validation failures
# Output: "⚠️  Test profit negative (-0.12%), parameters may be overfitted"
```

**Solution:** Increase `n_trials` or adjust constraints

### Issue: InfluxDB Write Fails

**Check 1: Bucket exists**
```bash
influx bucket list | grep OPTUNA_PARAMS
```

**Check 2: Token has write access**
```bash
# Test write
influx write -b OPTUNA_PARAMS -o ArtiSky \
    -t "uvAgH7X2WJxE_RGmZC4wUEuNLsZX2e9Uyhxg1lB9euW4I2qV6RjUZmQlzSTOJPoHbO2tLvarifULUCgvArVEXA==" \
    "test value=1"
```

**Solution:** Check token permissions, verify bucket exists

---

## 📁 CONFIGURATION FILES

### optuna_config.json (Required)

```json
{
  "optimization": {
    "days_back": 90,
    "n_trials": 100,
    "train_test_split": 0.7
  },
  "pairs": {
    "use_config_whitelist": true,
    "whitelist": [],
    "blacklist": []
  },
  "multi_bot": {
    "enabled": true,
    "bot3_weight": 3.0,
    "bot1_weight": 1.0,
    "bot2_weight": 1.0
  },
  "logging": {
    "level": "INFO",
    "file": "user_data/logs/optuna_optimizer.log"
  }
}
```

---

## 📈 PERFORMANCE METRICS

### Optimization Speed

- **Per pair:** ~2-3 minutes (100 trials)
- **150 pairs:** ~5-6 hours
- **Bottleneck:** OHLCV loading and backtesting

### Memory Usage

- **~500MB base** (InfluxDB client, FreqTrade imports)
- **+50MB per pair** (OHLCV cache)
- **Total for 150 pairs:** ~8GB

### Recommendations

- Run on BOT3 server (has 16GB RAM)
- Run during low-activity hours
- Monitor with `htop` during execution

---

## ✅ VERIFICATION CHECKLIST

After running optimizer:

- [ ] Check logs: `tail -f user_data/logs/optuna_optimizer.log`
- [ ] Verify InfluxDB: `python3 /tmp/check_optuna.py`
- [ ] Should show: `Total records found: 150+` (not 0!)
- [ ] Restart BOT2: `systemctl restart freqtrade`
- [ ] Check BOT2 logs: `grep optuna bot2.log`
- [ ] Should show: `📊 Loaded Optuna params for 150 pairs`
- [ ] Monitor new trades: Exit reasons should be `roi_optuna`, `stop_loss_optuna`

---

**STATUS:** Comprehensive logic review complete  
**NEXT STEP:** Run optimizer to generate parameters for BOT2
