# Session Context - BOT2 Optuna Integration & Feeder Assessment
**Date:** November 27, 2025  
**Status:** BOT2 Optuna Deployed | Feeder Assessment Pending

---

## ✅ COMPLETED: BOT2 Optuna Integration

### What Was Done:
1. **Implemented Optuna integration for BOT2 (192.168.3.72)**
   - Loads per-pair optimized take profit/stop loss from InfluxDB `OPTUNA_PARAMS` bucket
   - Falls back to fixed parameters (-7% stop / +1% profit) if Optuna unavailable
   - Fixed 401 Unauthorized error (used correct InfluxDB token and org)
   - File: `BOT2_ArtiSkyTrader_Optuna_Fixed.py` (deployed by user)

### Key Changes:
```python
# Optuna parameter loading
def _load_optuna_params(self):
    query = '''
    from(bucket: "OPTUNA_PARAMS")
    |> range(start: -90d)
    |> filter(fn: (r) => r._measurement == "parameters")
    |> group(columns: ["pair"])
    |> last()
    '''
    # Loads take_profit_pct and stop_loss_pct per pair

# Exit logic using Optuna
def custom_exit(self, pair, trade, ...):
    params = self._get_exit_params_for_pair(pair)  # Optuna or fallback
    stop_account = params['stop_loss_position_pct'] * leverage
    target_account = params['take_profit_position_pct'] * leverage
    # Exit with 'stop_loss_optuna' or 'stop_loss_fallback' reasons
```

### Verification Status:
- **Deployment:** User deployed and restarted BOT2
- **Awaiting:** Verification logs showing `✅ BOT2 WITH OPTUNA (AUTH FIXED) INITIALIZING...`
- **Next Trade:** Should show exit reasons `stop_loss_optuna`/`stop_loss_fallback` (not `cutloss_fallback`)

---

## ⏳ PENDING: Feeder Optuna Assessment

### Current State:

**Host 75 (192.168.3.75) - LONG Feeders:**
- Services: feeder75a, feeder75b, feeder75c
- Strategy: `ProducerStrategyDualQuantile_LONG.py`
- **Current thresholds:** STATIC values
  ```python
  PROFIT_TARGET_PCT = 0.01      # 1% - STATIC!
  RISK_PROTECTION_PCT = 0.0     # 0% - STATIC!
  
  upside_threshold = current_close * (1 + self.PROFIT_TARGET_PCT)
  downside_threshold = current_low * (1 + self.RISK_PROTECTION_PCT)
  ```

**Host 120 (192.168.3.120) - SHORT Feeders:**
- Services: feeder120a, feeder120b, feeder120c  
- Strategy: `ProducerStrategyDualQuantile_SHORT.py`
- **Current thresholds:** STATIC values
  ```python
  PROFIT_TARGET_PCT = 0.01      # 1% - STATIC!
  RISK_PROTECTION_PCT = 0.0     # 0% - STATIC!
  
  # INVERSE LOGIC for SHORT:
  downside_threshold = current_close * (1 - self.PROFIT_TARGET_PCT)
  upside_threshold = current_high * (1 + self.RISK_PROTECTION_PCT)
  ```

### 🔑 CRITICAL FINDING: INVERSE LOGIC

**LONG feeders (Host 75):**
- Expects price to **GO UP**
- Logic: `predicted_close >= upside_threshold`
- Threshold: `close * (1 + PCT)` (e.g., 101% of price)

**SHORT feeders (Host 120):**
- Expects price to **GO DOWN**  
- Logic: `predicted_close <= downside_threshold`
- Threshold: `close * (1 - PCT)` (e.g., 99% of price)

**Implication:** Cannot use the same Optuna integration code for both - needs direction-aware logic!

---

## 🤔 ASSESSMENT NEEDED: Should Feeders Use Optuna?

### Option A: Keep Feeders Simple (Current State)
**Pros:**
- Simple, predictable behavior
- All pairs treated equally
- No additional complexity

**Cons:**
- Not optimized per pair
- BTC and small-cap alts use same 1% threshold
- Missing opportunity for per-pair optimization

### Option B: Implement Optuna for Feeders
**Pros:**
- Per-pair optimized signal thresholds
- Better signal quality
- Consistent with BOT2's approach

**Cons:**
- More complex implementation
- Must handle LONG vs SHORT inverse logic correctly
- Requires testing and validation

### Option C: Hybrid Approach
- Keep signal generation simple (static thresholds)
- Only use Optuna for exit logic (BOT2) ✅ Already done

---

## 📋 NEXT SESSION PROMPT

```
I need to assess whether to implement Optuna integration for signal generation thresholds on feeders:

**Context:**
- BOT2 (192.168.3.72) now uses Optuna for exit logic (deployed)
- Feeders on Host 75 (LONG) and Host 120 (SHORT) still use static 1% thresholds
- LONG and SHORT feeders use INVERSE logic for signal generation

**Files to Review:**
- Host 75: `ProducerStrategyDualQuantile_LONG.py` (LONG signals)
- Host 120: `ProducerStrategyDualQuantile_SHORT.py` (SHORT signals)

**Questions:**
1. Should feeders load per-pair thresholds from Optuna?
2. If yes, how to handle LONG vs SHORT inverse logic?
3. What's the benefit vs complexity trade-off?

**Reference:**
- Session context: `SESSION_CONTEXT_BOT2_OPTUNA_2025-11-27.md`
- BOT2 implementation: `BOT2_ArtiSkyTrader_Optuna_Fixed.py`
```

---

## 🔧 Infrastructure Details

### Servers:
- **192.168.3.75** (Host 75): feeder75a/b/c - LONG signals - 2× Tesla T4 GPUs
- **192.168.3.120** (Host 120): feeder120a/b/c - SHORT signals - 1× Tesla T4 GPU
- **192.168.3.72** (BOT2): SHORT consumer - Optuna-integrated ✅
- **192.168.3.71** (BOT1): LONG consumer
- **192.168.3.6**: InfluxDB (OPTUNA_PARAMS bucket)

### Access:
```bash
# Host 75
ssh freqai@192.168.3.75
Password: NewHopes@168
Strategy: /home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_LONG.py

# Host 120
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120
Strategy: /home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_SHORT.py

# BOT2
ssh ubuntu@192.168.3.72
Strategy: /home/ubuntu/freqtrade/user_data/strategies/ArtiSkyTrader.py
```

### InfluxDB:
```
URL: http://192.168.3.6:8086
Token: freqtradeInfluxToken2024
Org: freqtrade
Bucket: OPTUNA_PARAMS
Measurement: parameters
Fields: take_profit_pct, stop_loss_pct, win_rate, total_trades
Tags: pair
```

---

## 📊 Key Learnings

1. **BOT2 Exit Logic:** Successfully uses Optuna per-pair optimization (deployed)
2. **Feeder Signal Logic:** LONG and SHORT use inverse thresholds - must be handled differently
3. **Two Types of Optuna Integration:**
   - **Exit Logic** (BOT1/BOT2): When to close trades - ✅ Done for BOT2
   - **Signal Generation** (Feeders): When to send signals - ⏳ Pending assessment

---

## 🎯 Decision Framework for Next Session

**Evaluate:**
1. Current signal quality with static 1% thresholds
2. Per-pair variation in optimal thresholds (check Optuna data)
3. Implementation complexity for direction-aware logic
4. Testing requirements
5. Deployment risk

**Expected Outcome:**
- Clear recommendation: Keep simple OR Implement Optuna
- If Optuna: Implementation plan with direction-aware logic
- If simple: Justification and alternative improvements

---

**END OF SESSION CONTEXT**
