# BOT2 Cutloss & Take Profit Optuna Troubleshooting - COMPLETE
**Date:** November 28, 2025  
**Issue:** BOT2 not using Optuna cutloss and take profit - using fallback values instead

---

## 🔍 EXECUTIVE SUMMARY

**Status:** ✅ ROOT CAUSE IDENTIFIED  
**Severity:** Medium - BOT2 is trading but using non-optimized parameters

### The Problem
BOT2 is exiting trades with `cutloss_fallback` and `roi_fallback` instead of using Optuna-optimized per-pair parameters.

### The Root Cause
**The Optuna optimizer service exists but has NEVER been installed or run!**

---

## 📊 INVESTIGATION RESULTS

### 1. BOT2 Strategy Status ✅

**Deployed Strategy:** `/home/ubuntu/freqtrade/user_data/strategies/ArtiSkyTrader.py`

```python
class ConsumerStrategy(IStrategy):
    """
    ✅ BOT2 STRATEGY WITH OPTUNA INTEGRATION (November 27, 2025 - Auth Fixed)
    - Reads optimized take profit/stop loss from InfluxDB OPTUNA_PARAMS
    - Falls back to simple fixed values if Optuna unavailable
    - Uses CORRECT InfluxDB token and org
    """
```

**Verdict:** ✅ Optuna integration IS properly implemented

### 2. BOT2 Optuna Loading ✅

**Log Evidence:**
```
2025-11-28 00:24:50,887 - ArtiSkyTrader - INFO - 📊 Loaded Optuna params for 0 pairs
```

**Exit Reasons in Logs:**
- `cutloss_fallback` - Using fallback stop loss
- `roi_fallback` - Using fallback take profit

**Verdict:** ✅ BOT2 IS trying to load Optuna parameters, but found 0 pairs

### 3. InfluxDB Parameters ❌

**Query Result:**
```
Total records found: 0

⚠️  NO OPTUNA PARAMETERS FOUND IN INFLUXDB!
This is why BOT2 is using fallback values.
```

**Bucket:** `OPTUNA_PARAMS`  
**Measurement:** `parameters`  
**Records:** **0** (empty!)

**Verdict:** ❌ No Optuna parameters exist in database

### 4. Optuna Optimizer Service ❌

**Service File:** `/home/ubuntu/freqtrade/optuna-optimizer.service` (exists)  
**Script:** `/home/ubuntu/freqtrade/user_data/services/run_optuna_optimizer.py` (exists)  
**Service Status:** `Unit optuna-optimizer.service could not be found.`

**Verdict:** ❌ Service exists but is NOT INSTALLED

---

## 🎯 ROOT CAUSE CONFIRMED

### The Complete Chain of Events

```
Optuna Optimizer Service
  ├─ Service file exists ✅
  ├─ Script exists ✅
  └─ Service NOT INSTALLED ❌
        ↓
  Optimizer has NEVER RUN ❌
        ↓
  NO parameters written to InfluxDB ❌
        ↓
  BOT2 loads 0 Optuna parameters ❌
        ↓
  BOT2 uses fallback values ❌
        ↓
  Exit reasons: cutloss_fallback, roi_fallback
```

### Why BOT2 is Using Fallback Values

**BOT2's logic is working CORRECTLY:**

1. Tries to load Optuna parameters from InfluxDB ✅
2. Finds 0 parameters (because optimizer never ran) ✅
3. Falls back to default values as designed ✅
   - `FALLBACK_STOP_LOSS_POSITION = -0.07` (7% stop loss)
   - `FALLBACK_TAKE_PROFIT_POSITION = 0.01` (1% take profit)

**This is the DESIGNED behavior when Optuna is unavailable!**

---

## 🔧 SOLUTION

### Step 1: Install the Optuna Optimizer Service

```bash
# Copy service file to systemd directory
sudo cp /home/ubuntu/freqtrade/optuna-optimizer.service /etc/systemd/system/

# Reload systemd to recognize the new service
sudo systemctl daemon-reload

# Enable the service to start on boot
sudo systemctl enable optuna-optimizer.service
```

### Step 2: Run the Optuna Optimizer

**Option A: Run Once Manually**
```bash
# Run the optimizer script directly
python3 /home/ubuntu/freqtrade/user_data/services/run_optuna_optimizer.py

# Check logs
tail -f /home/ubuntu/freqtrade/user_data/logs/optuna_optimizer.log
```

**Option B: Start the Service**
```bash
# Start the service
sudo systemctl start optuna-optimizer.service

# Check status
sudo systemctl status optuna-optimizer.service

# Check logs
sudo journalctl -u optuna-optimizer.service -f
```

### Step 3: Verify Optuna Parameters Were Created

```bash
# Run the check script
python3 /tmp/check_optuna.py
```

**Expected Output:**
```
Pair: BTC/USDT:USDT, Field: take_profit_pct, Value: 0.02
Pair: BTC/USDT:USDT, Field: stop_loss_pct, Value: -0.05
Pair: ETH/USDT:USDT, Field: take_profit_pct, Value: 0.018
...

Total records found: 150+
```

### Step 4: Verify BOT2 Picks Up the Parameters

**Wait 30 minutes** (BOT2 reloads parameters every 30 minutes) OR restart BOT2:

```bash
sshpass -p 'ArtiSky@7' ssh ubuntu@192.168.3.72 \
  "echo 'ArtiSky@7' | sudo -S systemctl restart freqtrade"
```

**Check BOT2 logs:**
```bash
sshpass -p 'ArtiSky@7' ssh ubuntu@192.168.3.72 \
  "tail -f /home/ubuntu/freqtrade/bot2.log | grep -i optuna"
```

**Expected Log:**
```
📊 Loaded Optuna params for 150 pairs  # Not 0!
```

### Step 5: Monitor New Trades

**New exit reasons should be:**
- `stop_loss_optuna` (instead of `cutloss_fallback`)
- `roi_optuna` (instead of `roi_fallback`)

**Example:**
```
🛑 BOT2 SHORT STOP: VET/USDT:USDT @ -4.38% (optuna)  # Not fallback!
🎯 BOT2 SHORT TARGET: ZEC/USDT:USDT @ 3.19% (optuna)  # Not fallback!
```

---

## 📋 FALLBACK VALUES REFERENCE

### Current Fallback Parameters

**File:** `BOT2_ArtiSkyTrader_Optuna_Fixed.py`

```python
FALLBACK_STOP_LOSS_POSITION = -0.07      # -7% position level
FALLBACK_TAKE_PROFIT_POSITION = 0.01     # +1% position level
```

**Account Level (with leverage):**
- Leverage 1x: -7% stop / +1% profit
- Leverage 3x: -21% stop / +3% profit
- Leverage 5x: -35% stop / +5% profit

### Why These Are Not Optimal

1. **Same for all pairs** - BTC and small-cap alts use identical thresholds
2. **Not data-driven** - Based on arbitrary fixed values
3. **No per-pair optimization** - Ignores individual pair characteristics

---

## 🔬 TECHNICAL DETAILS

### Optuna Optimizer Service Configuration

**Service File:** `/etc/systemd/system/optuna-optimizer.service`

```ini
[Unit]
Description=BOT3 Optuna Parameter Optimizer
After=network.target influxdb.service
Wants=network.target

[Service]
Type=oneshot
User=ubuntu
WorkingDirectory=/home/ubuntu/freqtrade
Environment="PATH=/home/ubuntu/.local/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=/usr/bin/python3 /home/ubuntu/freqtrade/user_data/services/run_optuna_optimizer.py
StandardOutput=append:/home/ubuntu/freqtrade/user_data/logs/optuna_optimizer.log
StandardError=append:/home/ubuntu/freqtrade/user_data/logs/optuna_optimizer_error.log
Restart=on-failure
RestartSec=30s

[Install]
WantedBy=multi-user.target
```

### BOT2 Parameter Loading Logic

**File:** `BOT2_ArtiSkyTrader_Optuna_Fixed.py`

```python
def _get_exit_params_for_pair(self, pair: str) -> dict:
    # Reload params every 30 minutes
    if self.influx_available and time.time() - self.last_params_load > 1800:
        self._load_optuna_params()
    
    # Priority 1: Optuna per-pair params
    if pair in self.optuna_params:
        params = self.optuna_params[pair]
        return {
            'take_profit_position_pct': params['take_profit_position_pct'],
            'stop_loss_position_pct': params['stop_loss_position_pct'],
            'source': 'optuna',  # Exit reason will be 'roi_optuna' or 'stop_loss_optuna'
            ...
        }
    
    # Priority 2: Fallback
    return {
        'take_profit_position_pct': self.FALLBACK_TAKE_PROFIT_POSITION,
        'stop_loss_position_pct': self.FALLBACK_STOP_LOSS_POSITION,
        'source': 'fallback',  # Exit reason will be 'roi_fallback' or 'cutloss_fallback'
        ...
    }
```

### InfluxDB Schema

**Bucket:** `OPTUNA_PARAMS`  
**Organization:** `ArtiSky`  
**Token:** `uvAgH7X2WJxE_RGmZC4wUEuNLsZX2e9Uyhxg1lB9euW4I2qV6RjUZmQlzSTOJPoHbO2tLvarifULUCgvArVEXA==`

**Data Structure:**
```
Measurement: parameters
Tags:
  - pair (e.g., "BTC/USDT:USDT")
Fields:
  - take_profit_pct (float, position level)
  - stop_loss_pct (float, position level, negative)
  - win_rate (float, 0-1)
  - total_trades (int)
  - profit_factor (float)
```

---

## 📈 EXPECTED OUTCOMES

### After Installing Optuna Optimizer

1. **InfluxDB populated** with per-pair optimized parameters
2. **BOT2 loads parameters** on next reload (30 min or restart)
3. **Exit reasons change** from `fallback` to `optuna`
4. **Per-pair optimization** applied to all trades
5. **Improved performance** (data-driven exits)

### Performance Comparison

**Before (Fallback):**
- All pairs: 1% take profit, 7% stop loss
- No pair-specific optimization
- Generic risk/reward ratio

**After (Optuna):**
- Each pair: Individually optimized thresholds
- Based on historical performance data
- Adaptive to pair volatility and characteristics

---

## 🚨 IMPORTANT NOTES

### BOT2 is Currently Working Correctly

**BOT2 is NOT broken!** It's working exactly as designed:
- Optuna integration implemented ✅
- Falls back gracefully when Optuna unavailable ✅
- Continues trading with safe default values ✅

### The Issue is the Missing Optimizer

**The Optuna optimizer was never started:**
- Service file created but not installed
- Script exists but never executed
- No parameters generated

### No Immediate Action Required

**BOT2 can continue trading** with fallback values while you:
1. Install the optimizer service
2. Let it generate parameters
3. Wait for BOT2 to pick them up

**No downtime needed!**

---

## 📝 CHECKLIST FOR RESOLUTION

### Prerequisites
- [ ] InfluxDB is running and accessible at `192.168.3.6:8086`
- [ ] InfluxDB bucket `OPTUNA_PARAMS` exists
- [ ] BOT2 has sufficient trade history for optimization
- [ ] Python dependencies installed (`influxdb_client`, `optuna`)

### Installation Steps
- [ ] Copy service file to `/etc/systemd/system/`
- [ ] Run `sudo systemctl daemon-reload`
- [ ] Run `sudo systemctl enable optuna-optimizer.service`
- [ ] Start the service or run script manually
- [ ] Check logs for successful execution
- [ ] Verify parameters exist in InfluxDB
- [ ] Wait 30 minutes or restart BOT2
- [ ] Monitor BOT2 logs for Optuna parameter loading
- [ ] Verify new trades use `optuna` exit reasons

### Verification
- [ ] `python3 /tmp/check_optuna.py` shows parameters
- [ ] BOT2 logs show "Loaded Optuna params for N pairs" (N > 0)
- [ ] New trades exit with `roi_optuna` or `stop_loss_optuna`
- [ ] Exit percentages vary per pair (not all 1%/7%)

---

## 🔗 RELATED FILES

**BOT2 Strategy:**
- Location: `192.168.3.72:/home/ubuntu/freqtrade/user_data/strategies/ArtiSkyTrader.py`
- Fixed Version: `/home/ubuntu/freqtrade/BOT2_ArtiSkyTrader_Optuna_Fixed.py`

**Optuna Service:**
- Service File: `/home/ubuntu/freqtrade/optuna-optimizer.service`
- Script: `/home/ubuntu/freqtrade/user_data/services/run_optuna_optimizer.py`
- Logs: `/home/ubuntu/freqtrade/user_data/logs/optuna_optimizer.log`

**Documentation:**
- Analysis: `BOT2_FALLBACK_ISSUE_ANALYSIS_2025-11-27.md`
- Session Context: `SESSION_CONTEXT_BOT2_OPTUNA_2025-11-27.md`
- This Document: `BOT2_CUTLOSS_OPTUNA_TROUBLESHOOTING_COMPLETE_2025-11-28.md`

---

## 📞 QUICK REFERENCE

### Check if Optuna Parameters Exist
```bash
python3 /tmp/check_optuna.py
```

### Install Optuna Service
```bash
sudo cp optuna-optimizer.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable optuna-optimizer.service
sudo systemctl start optuna-optimizer.service
```

### Check BOT2 Optuna Status
```bash
sshpass -p 'ArtiSky@7' ssh ubuntu@192.168.3.72 \
  "tail -50 /home/ubuntu/freqtrade/bot2.log | grep -i optuna"
```

### Restart BOT2 (to reload parameters immediately)
```bash
sshpass -p 'ArtiSky@7' ssh ubuntu@192.168.3.72 \
  "echo 'ArtiSky@7' | sudo -S systemctl restart freqtrade"
```

---

**STATUS:** Investigation Complete ✅  
**SOLUTION:** Install and run Optuna optimizer service  
**NEXT STEP:** User to execute solution steps above
