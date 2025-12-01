# Optuna Logic Deep Dive - Why Stop Loss is Tight

**Date:** 2025-11-23  
**Issue:** Current stop loss at -0.50% seems too tight  
**Root Cause:** Old parameters from BEFORE timezone fix

---

## CURRENT INFLUXDB PARAMETERS (OLD LOGIC)

```
Saved: 2025-11-23 00:22:37 (BEFORE timezone fix at 07:47)

AAVE/USDT:USDT    SL=-0.50%  TP=+1.84%
ADA/USDT:USDT     SL=-0.51%  TP=+1.92%
ALGO/USDT:USDT    SL=-0.52%  TP=+2.03%
...
```

**These are from the BROKEN logic before timezone fix!**

### What Was Wrong:
1. ❌ Timezone mismatch → all simulations failed
2. ❌ Returned all -999 values → optimizer picked whatever
3. ❌ Not based on actual OHLCV candle movement
4. ❌ Stops are way too tight for crypto volatility

---

## OPTUNA OPTIMIZATION LOGIC (Fixed Version)

### Step 1: Define Search Ranges (Line 213-214)

```python
def objective(trial):
    # Position-level ranges (before leverage)
    sl = trial.suggest_float('sl', -0.04, -0.01)   # -4% to -1%
    tp = trial.suggest_float('tp', 0.005, 0.03)    # +0.5% to +3%
```

**At 3x Leverage (account level):**
- Stop Loss: **-12% to -3%** (account)
- Take Profit: **+1.5% to +9%** (account)

### Step 2: Simulate Each Parameter Combination

For each trial (50 trials total):

1. **Pick random stop & target** within ranges
2. **Load training entries** (70% of historical trades)
3. **For each entry:**
   - Get entry time and price
   - Load OHLCV candles starting from entry
   - Walk through next 100 candles (5min candles = 8.3 hours max)
   - **Check each candle:**
     ```python
     if candle['low'] <= (entry_price * (1 + stop_loss)):
         return stop_loss  # Hit stop
     
     if candle['high'] >= (entry_price * (1 + take_profit)):
         return take_profit  # Hit target
     ```
   - Return actual exit profit
4. **Calculate average profit** across all simulated trades
5. **Score this trial** based on average profit

### Step 3: Find Best Parameters

Optuna tries to **maximize average profit** by:
- Testing different stop/target combinations
- Simulating how each would have performed historically
- Finding the combination with highest average profit

### Step 4: Validate on Test Set

- Use remaining 30% of trades (not seen during optimization)
- Simulate with best parameters
- Calculate actual profit, win rate, profit factor
- Only save if test profit is positive

---

## WHY CURRENT STOPS ARE TOO TIGHT

### Problem: Old Results from Broken Simulation

The -0.50% stops in InfluxDB are from **before the timezone fix**:

**Before Fix (what happened at 00:22:37):**
```python
# Line 128 BEFORE fix:
entry_time_dt = pd.to_datetime(entry_time)  # tz-naive

# Comparison fails:
entry_candles = ohlcv[ohlcv['date'] >= entry_time_dt]
# TypeError: Cannot compare tz-naive and tz-aware
# Result: Returns empty dataframe → all simulations fail → returns None

# Optimizer gets:
all returns = None → scores as -999 → picks arbitrary parameters
```

The optimizer wasn't actually using OHLCV data, so it found unrealistic stops!

---

## PROPER RANGES (After Fix)

### Why -4% to -1% Range Makes Sense:

From 3,642 actual BOT3 trades analysis:

**Historical Loss Distribution:**
```
Average loss: -1.46%
Median loss: -1.31%
Worst 10% (P10): -2.49%
Worst loss: -6.12%
```

**Recommended Stop Range Logic:**
```
Minimum (-1%):  Tighter than average loss
                → Will stop many trades prematurely
                → Reduces profits
                
Sweet Spot (-1.5% to -2.5%):
                → Wider than average loss (-1.46%)
                → Allows for normal volatility
                → Prevents catastrophic losses
                
Maximum (-4%):  Very loose stop
                → Catches most volatility
                → But may let losers run too long
```

### Why +0.5% to +3% Range Makes Sense:

**Historical Win Distribution:**
```
Average win: +0.88%
Median win: +0.80%
Best 10% (P90): +1.57%
Best win: +5.65%
```

**Recommended Target Range Logic:**
```
Minimum (+0.5%): Very conservative
                → Takes profits early
                → High win rate but small wins
                
Sweet Spot (+0.8% to +1.5%):
                → Around actual average/P90
                → Achievable in most market conditions
                → Balances win rate vs profit size
                
Maximum (+3%):  Ambitious target
                → Lets winners run
                → Lower win rate but bigger wins
```

---

## OPTUNA'S ACTUAL OPTIMIZATION PROCESS

### Example: Why Optuna Might Choose Different Stops

**Scenario A: Tight Stop + Tight Target**
```
Stop: -1.0%, Target: +0.8%
Simulation Results:
- 70 trades tested
- 52 winners (74% win rate)
- Average profit per trade: +0.35%
- Total score: +0.35% ← Optimizer uses this

Logic: Tight stops cut losses fast, tight targets lock profits fast
Win rate high, but profits small
```

**Scenario B: Loose Stop + Loose Target**  
```
Stop: -2.5%, Target: +1.8%
Simulation Results:
- 70 trades tested
- 38 winners (54% win rate)  
- Average profit per trade: +0.42%
- Total score: +0.42% ← Better! Optimizer prefers this

Logic: Wider stops allow for volatility, wider targets catch bigger moves
Win rate lower, but profits larger, net result better
```

**Optuna picks Scenario B** because it maximizes average profit!

---

## WHAT HAPPENS WITH FIXED CODE

### Proper Simulation Flow:

```python
Entry: BTC at $100,000 on 2025-11-12 09:15:54
Stop: -2% = $98,000
Target: +1.5% = $101,500

Walk through candles:
09:15:00 → H: $100,200, L: $99,800  # No hit
09:20:00 → H: $100,500, L: $99,500  # No hit  
09:25:00 → H: $101,100, L: $99,900  # No hit
09:30:00 → H: $101,600, L: $100,200 # HIGH HIT TARGET!
Return: +1.5% profit ✅

Average across 70 trades: +0.42%
This combination scores well!
```

### Compare to Tight Stops (Old Results):

```python
Entry: BTC at $100,000
Stop: -0.5% = $99,500  ← TOO TIGHT!
Target: +1.8% = $101,800

Walk through candles:
09:15:00 → H: $100,200, L: $99,800  # No hit
09:20:00 → H: $100,500, L: $99,500  # LOW TOUCHES STOP!
Return: -0.5% loss ❌

With tight stops, many trades stop out unnecessarily
Even though price later went to target, we exit at stop
This is why -0.50% is too tight for crypto!
```

---

## CRYPTO VOLATILITY CONSIDERATIONS

### Intraday Volatility Examples:

**BTC typical 5-minute movement:**
- Quiet periods: ±0.1% to ±0.3%
- Normal trading: ±0.3% to ±0.8%
- Volatile periods: ±0.8% to ±2.0%
- News/events: ±2.0% to ±5.0%

**With -0.50% stop:**
- Gets stopped out in NORMAL trading volatility
- Any -0.5% wick stops the trade
- Doesn't allow strategy to work
- Destroys win rate

**With -1.5% to -2.0% stop:**
- Survives normal volatility
- Only stops on adverse moves
- Allows strategy to work
- Balances protection vs opportunity

---

## RECOMMENDATIONS

### Option 1: **Increase Stop Range** (Recommended)

Edit `optuna_standalone.py` line 213:

```python
# CURRENT (good ranges):
sl = trial.suggest_float('sl', -0.04, -0.01)  # -4% to -1%

# If you want even looser stops:
sl = trial.suggest_float('sl', -0.06, -0.015)  # -6% to -1.5%

# More conservative (tighter than current):
sl = trial.suggest_float('sl', -0.03, -0.015)  # -3% to -1.5%
```

**At 3x leverage:**
- -6% to -1.5% position = -18% to -4.5% account
- -3% to -1.5% position = -9% to -4.5% account

### Option 2: **Use Stateless Manager** (Already Working)

The Stateless Manager is already running well:
```
Stop: -1.48% (position) = -4.44% @ 3x (account)
Target: +1.00% (position) = +3.00% @ 3x (account)
Win Rate: 61%
Status: PROVEN 9+ hours stable
```

This is a good middle-ground and is WORKING!

### Option 3: **Re-run with Fixed Code**

```bash
python3 user_data/services/optuna_standalone.py
```

With the timezone fix, Optuna will now:
1. ✅ Actually use OHLCV candle data
2. ✅ Test stops in -4% to -1% range (realistic)
3. ✅ Find optimal balance based on actual price action
4. ✅ Likely recommend -1.5% to -2.5% stops (wider than old -0.50%)

---

## COMPARISON TABLE

| Approach | Stop (Position) | Stop @ 3x (Account) | Status | Based On |
|----------|----------------|-------------------|---------|----------|
| **Old Optuna Results** | -0.50% | -1.5% | ❌ Too Tight | Broken simulation |
| **Stateless Manager** | -1.48% | -4.44% | ✅ Working | Final outcomes |
| **Fixed Optuna (Expected)** | -1.5% to -2.5% | -4.5% to -7.5% | ⏳ Needs Run | OHLCV candles |
| **Historical Average Loss** | -1.46% | -4.38% | 📊 Reference | 3,642 trades |

---

## CONCLUSION

### Why Stop is Tight:
**The -0.50% stops in InfluxDB are from the OLD broken version before timezone fix at 00:22:37.**

They're NOT based on actual OHLCV simulation - they're arbitrary because all simulations failed!

### What to Do:

**Recommended Action:**
```bash
# Re-run Optuna with fixed code
python3 user_data/services/optuna_standalone.py

# Expected results after fix:
# Stop: -1.5% to -2.5% (position)
# Stop @ 3x: -4.5% to -7.5% (account)
# Much more reasonable for crypto volatility!
```

**Or Keep Current:**
```
Stateless Manager at -1.48% / -4.44% @ 3x
Already proven to work well
61% win rate, stable 9+ hours
This is actually a good stop level!
```

### Key Insight:

**Crypto needs breathing room!** A -0.50% stop doesn't account for:
- Normal intraday volatility (±0.5% common)
- Spread/slippage
- Wick hunting by market makers  
- Strategy development time

**The fixed Optuna will find the optimal balance between:**
- Protection (cutting losses)
- Opportunity (letting strategy work)
- Based on real price action in OHLCV candles

---

**DON'T USE THE -0.50% STOPS - THEY'RE FROM BROKEN CODE!**  
**RE-RUN WITH FIXED CODE TO GET REALISTIC PARAMETERS! 🎯**
