# BOT2 OHLCV Optimization Bug Report - 2025-11-28

## 🎯 ORIGINAL TASK
Troubleshoot why BOT2 not using Optuna cutloss/take-profit parameters (using roi_fallback instead).

---

## ✅ WHAT WAS ACCOMPLISHED

### 1. Root Cause Identified
- **Problem:** BOT2 using `roi_fallback` instead of custom_stop/custom_profit
- **Cause:** Optuna parameters not being generated for BOT2
- **Reason:** BOT2's Optuna service dependent on BOT3 controller

### 2. Solution Implemented
**Created standalone Optuna system for BOT2:**
- Location: `/home/ubuntu/freqtrade/user_data/services/` on BOT2
- Files deployed:
  - `run_optuna_per_bot_standalone.py` - Main runner ✅
  - `optuna_optimizer_per_bot_standalone.py` - Optimizer class (HAS BUG ❌)
  - Service: `/etc/systemd/system/optuna-bot2-standalone.service` ✅

### 3. Service Configuration
- ✅ Service file installed and enabled
- ✅ Continuous running (Restart=always, RestartSec=21600)
- ✅ Resource limits set (30% CPU, 19GB RAM)
- ✅ Logging configured

---

## ❌ CURRENT BUG

### Bug Summary
**IndentationError in optimizer file prevents service from starting**

**Error:**
```
File "/home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py", line 636-665
IndentationError: unexpected indent / invalid syntax
```

**Service Status:**
```
Active: activating (auto-restart) (Result: exit-code)
Process: code=exited, status=1/FAILURE
```

### Root Cause
Attempted to add OHLCV-based optimization methods to the optimizer class, but the code insertion created cascading indentation errors in the exception handling block.

---

## 🔍 TECHNICAL DETAILS

### What OHLCV Optimization Does
**Critical for accurate parameter optimization:**
- Loads actual OHLCV candle data from BOT2's data directory
- Simulates trades over 30 minutes (6 candles @ 5m timeframe)
- Checks if stop-loss or take-profit is hit on each candle's high/low
- More accurate than simple entry/exit price comparison
- **User says: "it is the main logic effect"** - this is essential

### Expected OHLCV Methods
Two methods should be added to `OptunaParameterOptimizer` class:

1. `_load_ohlcv_data(pair, timeframe='5m')` - Loads candles from disk
2. `_simulate_trade_on_candles(...)` - Walks through candles to find SL/TP hits

### File Structure Issue
**Problem area in `save_results_to_influxdb` method:**

```python
# Line 635 (correct)
        from influxdb_client import Point
        
# Line 636 (SHOULD BE HERE but has wrong indentation)
        try:
            write_api = self.influx.client.write_api(write_options=SYNCHRONOUS)
            points = []
            # ... rest of try block
            
# Line 664-667 (except block exists but try: is malformed)
        except Exception as e:
            logger.error(...)
```

**Current state:**
- Lines 636-663 have incorrect indentation (12 spaces when should be 8, or vice versa)
- The `try:` statement is either missing or has wrong indentation
- The `except:` block at line 664 has no matching `try:` → SyntaxError

---

## 🔧 WHAT HAS BEEN TRIED

### Attempt 1: Remove Malformed Lines
```python
# Removed lines 636-640 (duplicate imports)
# Result: Still IndentationError at new line 636
```

### Attempt 2: Fix Indentation (12→8 spaces)
```python
# Reduced indentation from 12 to 8 spaces for lines 636-660
# Result: except Exception as e: - SyntaxError (no matching try)
```

### Attempt 3: Add Missing try: Block
```python
# Inserted 'try:' after line 635
# Result: IndentationError - expected indented block after try
```

### Attempt 4: Re-indent Code Inside try Block
```python
# Added 4 spaces to lines after try: (8→12 spaces)
# Result: Still SyntaxError at except block
```

### Attempt 5: Use Clean Base File
```python
# Tried to read optuna_optimizer_per_bot.py
# Result: "Can't find optimize_pair_parameters method" - wrong file structure
```

---

## 📁 FILE LOCATIONS

### On BOT3 (192.168.3.33)
**Clean working files:**
- `user_data/services/optuna_optimizer_per_bot.py` - May not have the right methods
- `user_data/services/run_optuna_per_bot_standalone.py` - Working ✅

### On BOT2 (192.168.3.72)  
**Broken file:**
- `user_data/services/optuna_optimizer_per_bot_standalone.py` - HAS INDENTATION BUG ❌
- Last modified: 2025-11-28 10:33
- Size: 26KB
- Status: IndentationError at line 636

**Working files:**
- `user_data/services/run_optuna_per_bot_standalone.py` - OK ✅
- Service: `/etc/systemd/system/optuna-bot2-standalone.service` - OK ✅

---

## 🎯 RECOMMENDED FIX FOR NEXT SESSION

### Approach: Fresh Start with Careful Insertion

**Step 1: Get Clean Base**
```bash
# Find or create a working optimizer without OHLCV
# Either use optuna_optimizer_per_bot.py or download from BOT1
```

**Step 2: Add OHLCV Methods Correctly**
```python
# Add as CLASS METHODS (not inside another method!)
# Insert before the optimize_pair_parameters method
# Use proper indentation: 4 spaces for class methods

class OptunaParameterOptimizer:
    
    # ... existing methods ...
    
    def _load_ohlcv_data(self, pair: str, timeframe: str = '5m'):
        """Load OHLCV data from BOT2's data directory."""
        try:
            from pathlib import Path
            from freqtrade.data.history import load_pair_history
            
            datadir = Path('/home/ubuntu/freqtrade/user_data/data/binance')
            candles = load_pair_history(
                pair=pair,
                timeframe=timeframe,
                datadir=datadir
            )
            logger.info(f"   ✅ Loaded {len(candles)} candles for {pair}")
            return candles
        except Exception as e:
            logger.warning(f"   ⚠️  Could not load OHLCV for {pair}: {e}")
            return None
    
    def _simulate_trade_on_candles(self, entry_time, entry_price, candles_df, 
                                    stop_loss, take_profit, is_short, candle_count=6):
        """Simulate trade outcome using actual candle data."""
        try:
            import pandas as pd
            
            # Find entry candle
            entry_idx = candles_df[candles_df['date'] >= entry_time].index
            if len(entry_idx) == 0:
                return {'profit': 0, 'exit_reason': 'no_data'}
            
            entry_idx = entry_idx[0]
            next_candles = candles_df.iloc[entry_idx:entry_idx + candle_count]
            
            # Calculate SL/TP prices
            if not is_short:
                sl_price = entry_price * (1 + stop_loss)
                tp_price = entry_price * (1 + take_profit)
            else:
                sl_price = entry_price * (1 - stop_loss)
                tp_price = entry_price * (1 - take_profit)
            
            # Walk through candles
            for idx, row in next_candles.iterrows():
                if not is_short:
                    if row['low'] <= sl_price:
                        return {'profit': stop_loss, 'exit_reason': 'stop_loss'}
                    if row['high'] >= tp_price:
                        return {'profit': take_profit, 'exit_reason': 'take_profit'}
                else:
                    if row['high'] >= sl_price:
                        return {'profit': stop_loss, 'exit_reason': 'stop_loss'}
                    if row['low'] <= tp_price:
                        return {'profit': take_profit, 'exit_reason': 'take_profit'}
            
            # Neither hit - use final close
            final_price = next_candles.iloc[-1]['close']
            profit = (final_price - entry_price) / entry_price if not is_short else (entry_price - final_price) / entry_price
            
            return {'profit': profit, 'exit_reason': 'timeout'}
        
        except Exception as e:
            logger.warning(f"   ⚠️  Simulation error: {e}")
            return {'profit': 0, 'exit_reason': 'error'}
    
    def optimize_pair_parameters(...):
        # Existing method
```

**Step 3: Modify _backtest_parameters to Use OHLCV**
```python
def _backtest_parameters(self, pair, trades, stop_loss, take_profit):
    # Load candles once per pair
    candles_df = self._load_ohlcv_data(pair, timeframe='5m')
    
    if candles_df is not None and len(candles_df) > 0:
        # Use OHLCV simulation
        total_profit = 0
        for trade in trades:
            result = self._simulate_trade_on_candles(
                entry_time=trade['open_time'],
                entry_price=trade['open_rate'],
                candles_df=candles_df,
                stop_loss=stop_loss,
                take_profit=take_profit,
                is_short=trade.get('is_short', False),
                candle_count=6
            )
            total_profit += result['profit']
        return total_profit / len(trades) if trades else 0
    else:
        # Fallback to simple backtest
        # ... existing logic
```

**Step 4: Test Syntax Before Deployment**
```bash
python3 -m py_compile /tmp/fixed_optimizer.py
```

**Step 5: Deploy and Verify**
```bash
scp /tmp/fixed_optimizer.py ubuntu@192.168.3.72:/home/ubuntu/freqtrade/user_data/services/optuna_optimizer_per_bot_standalone.py

ssh ubuntu@192.168.3.72 "sudo systemctl restart optuna-bot2-standalone && systemctl status optuna-bot2-standalone --no-pager"
```

---

## ⚠️ CRITICAL NOTES FOR NEXT SESSION

1. **DO NOT insert OHLCV code inside save_results_to_influxdb method**
   - This was the source of all indentation errors
   - OHLCV methods must be separate class methods

2. **Verify indentation with Python's parser BEFORE deployment**
   - Use `py_compile` to catch syntax errors locally
   - Don't deploy and test iteratively - fix locally first

3. **The exception handling in save_results_to_influxdb is fragile**
   - Don't modify this method
   - Add OHLCV methods elsewhere in the class

4. **OHLCV is essential - user explicitly stated**
   - "it is the main logic effect"
   - Cannot skip this - must fix the indentation

5. **Method to find in base file:**
   - Look for `class OptunaParameterOptimizer`
   - Find `def _backtest_parameters` - this is where OHLCV should be called
   - Add `_load_ohlcv_data` and `_simulate_trade_on_candles` BEFORE this method

---

## 📊 EXPECTED BEHAVIOR AFTER FIX

**Service starts successfully:**
```
● optuna-bot2-standalone.service - running
   Active: active (running)
```

**Logs show OHLCV usage:**
```
🚀 BOT2 Optuna Parameter Optimization
✅ InfluxDB connected
📊 Optimizing AERO/USDT:USDT (1/163)
   ✅ Loaded 50000 candles for AERO/USDT:USDT  ← OHLCV WORKING!
   💡 Trial 1: SL=-1.5%, TP=8.0%
      Walking 6 candles (30 min)...
   ✅ Best: SL=-1.0%, TP=11.36%
💾 Results saved to InfluxDB
```

**BOT2 behavior:**
- Stops using `roi_fallback`
- Uses `custom_stop` and `custom_profit`
- Parameters optimized with real candle data
- Re-optimizes every 6 hours automatically

---

## 🔑 KEY FILES TO EXAMINE

1. **OHLCV_OPTIMIZATION_METHODS.txt** (create this with the methods above)
2. **optuna_optimizer_per_bot_standalone.py** (on BOT2 - broken)
3. **optuna_optimizer_per_bot.py** (on BOT3 - may be clean base)

---

## 📝 SESSION SUMMARY

**Time spent:** ~2 hours
**Main blocker:** Python indentation errors from OHLCV code insertion
**Status:** Service deployed but not running due to syntax errors
**Next action:** Fix indentation by starting fresh with clean base file
**Priority:** HIGH - OHLCV is essential per user requirement

---

**Created:** 2025-11-28 10:47
**For next session:** Start by reading this document, then implement the recommended fix above.
