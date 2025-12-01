# Feeder Optuna Integration - Deployment Guide
**Date:** November 27, 2025  
**Status:** Ready for Testing

---

## 📋 Overview

This guide documents the implementation of Optuna-based per-pair thresholds for feeder signal generation strategies.

### What Was Implemented

**NEW Strategies Created:**
- `ProducerStrategyDualQuantile_LONG_Optuna.py` - LONG feeders with Optuna thresholds
- `ProducerStrategyDualQuantile_SHORT_Optuna.py` - SHORT feeders with Optuna thresholds

**How It Works:**
1. Feeders load `take_profit_pct` and `stop_loss_pct` from existing `OPTUNA_PARAMS` InfluxDB bucket
2. These values (already optimized by consumer trade performance) become signal thresholds
3. Creates a feedback loop: Consumer optimization → Feeder thresholds → Better signals → Better consumer performance

---

## 🔑 Key Implementation Details

### LONG Feeder Thresholds

```python
# Load from Optuna
take_profit = abs(record.values.get('take_profit_pct', 0.01))  # e.g., 0.02 (2%)
stop_loss = abs(record.values.get('stop_loss_pct', 0.07))      # e.g., 0.05 (5%)

# Apply to signal generation (LONG logic)
upside_threshold = current_close * (1 + take_profit)      # Price must go UP by 2%
downside_threshold = current_low * (1 - stop_loss)        # Protect from 5% drawdown

# Send signal when
predicted_close >= upside_threshold AND predicted_min_low >= downside_threshold
```

### SHORT Feeder Thresholds (INVERSE)

```python
# Load from Optuna (same values as LONG)
take_profit = abs(record.values.get('take_profit_pct', 0.01))  # e.g., 0.02 (2%)
stop_loss = abs(record.values.get('stop_loss_pct', 0.07))      # e.g., 0.05 (5%)

# Apply to signal generation (SHORT logic - INVERSE)
downside_threshold = current_close * (1 - take_profit)    # Price must go DOWN by 2%
upside_threshold = current_high * (1 + stop_loss)         # Protect from 5% upward movement

# Send signal when (INVERSE comparisons)
predicted_close <= downside_threshold AND predicted_max_high <= upside_threshold
```

### Fallback Values

If Optuna unavailable:
- **Profit target:** 1% (fallback from static implementation)
- **Risk protection:** 7% (reasonable default based on market volatility)

---

## 🎯 Mathematical Comparison

### Example: BTC/USDT with Optuna params: take_profit=2%, stop_loss=5%

**Current price:** $100,000

#### LONG Feeder (expects price UP):
```
Upside threshold   = $100,000 × (1 + 0.02) = $102,000
Downside threshold = $100,000 × (1 - 0.05) = $95,000

Signal sent if:
  - Predicted close (30m) >= $102,000 (2% profit)
  - Predicted min low >= $95,000 (max 5% drawdown)
```

#### SHORT Feeder (expects price DOWN):
```
Downside threshold = $100,000 × (1 - 0.02) = $98,000
Upside threshold   = $100,000 × (1 + 0.05) = $105,000

Signal sent if:
  - Predicted close (30m) <= $98,000 (2% drop)
  - Predicted max high <= $105,000 (max 5% upward spike)
```

---

## 📊 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    OPTUNA OPTIMIZATION                          │
│                                                                 │
│  BOT1/BOT2 Trade History → Optuna → InfluxDB OPTUNA_PARAMS    │
│  Fields: take_profit_pct, stop_loss_pct, win_rate, total_trades│
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓ Load every 30 minutes
┌─────────────────────────────────────────────────────────────────┐
│                    FEEDER SIGNAL GENERATION                     │
│                                                                 │
│  Host 75 LONG Feeders                  Host 120 SHORT Feeders  │
│  ┌──────────────────────┐             ┌──────────────────────┐ │
│  │ Load Optuna params   │             │ Load Optuna params   │ │
│  │ take_profit → upside │             │ take_profit → down   │ │
│  │ stop_loss → downside │             │ stop_loss → upside   │ │
│  │                      │             │                      │ │
│  │ LONG LOGIC:          │             │ SHORT LOGIC:         │ │
│  │ up = close*(1+tp)    │             │ down = close*(1-tp)  │ │
│  │ down = low*(1-sl)    │             │ up = high*(1+sl)     │ │
│  │                      │             │                      │ │
│  │ Signal if:           │             │ Signal if:           │ │
│  │ pred >= up_thresh    │             │ pred <= down_thresh  │ │
│  │ pred >= down_thresh  │             │ pred <= up_thresh    │ │
│  └──────────────────────┘             └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ↓ Signals
┌─────────────────────────────────────────────────────────────────┐
│                    CONSUMER TRADING                             │
│                                                                 │
│  BOT1 (LONG) / BOT2 (SHORT)                                    │
│  Uses same Optuna params for exit logic                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🚀 Deployment Instructions

### Phase 1: Test on Single LONG Feeder (feeder75a)

```bash
# SSH to Host 75
ssh freqai@192.168.3.75
# Password: NewHopes@168

# Backup current strategy
cd /home/freqai/freqtrade/user_data/strategies
cp ProducerStrategyDualQuantile_LONG.py ProducerStrategyDualQuantile_LONG_backup_$(date +%Y%m%d_%H%M%S).py

# Copy new Optuna-enabled strategy
scp ubuntu@192.168.3.77:/home/ubuntu/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_LONG_Optuna.py /home/freqai/freqtrade/user_data/strategies/

# Rename to replace current strategy
mv ProducerStrategyDualQuantile_LONG_Optuna.py ProducerStrategyDualQuantile_LONG.py

# Restart feeder75a service
sudo systemctl restart freqtrade-feeder75a

# Monitor logs
sudo journalctl -u freqtrade-feeder75a -f | grep -E "OPTUNA|LONG SIGNAL|Loaded Optuna"
```

**Expected output:**
```
✅ LONG FEEDER WITH OPTUNA THRESHOLDS INITIALIZING...
🎯 Optuna thresholds loaded: 163 pairs
   Optuna: ACTIVE
```

### Phase 2: Test on Single SHORT Feeder (feeder120a)

```bash
# SSH to Host 120
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120

# Backup current strategy
cd /home/freqai/freqtrade/user_data/strategies
cp ProducerStrategyDualQuantile_SHORT.py ProducerStrategyDualQuantile_SHORT_backup_$(date +%Y%m%d_%H%M%S).py

# Copy new Optuna-enabled strategy
scp ubuntu@192.168.3.77:/home/ubuntu/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_SHORT_Optuna.py /home/freqai/freqtrade/user_data/strategies/

# Rename to replace current strategy
mv ProducerStrategyDualQuantile_SHORT_Optuna.py ProducerStrategyDualQuantile_SHORT.py

# Restart feeder120a service
sudo systemctl restart freqtrade-feeder120a

# Monitor logs
sudo journalctl -u freqtrade-feeder120a -f | grep -E "OPTUNA|SHORT SIGNAL|Loaded Optuna"
```

**Expected output:**
```
✅ SHORT FEEDER WITH OPTUNA THRESHOLDS INITIALIZING...
🎯 Optuna thresholds loaded: 163 pairs
   Optuna: ACTIVE
```

### Phase 3: Validation (24-48 hours)

Monitor for:

1. **Optuna connection:**
   ```bash
   # Check for successful Optuna loading
   sudo journalctl -u freqtrade-feeder75a --since "1 hour ago" | grep "Loaded Optuna params"
   sudo journalctl -u freqtrade-feeder120a --since "1 hour ago" | grep "Loaded Optuna params"
   ```

2. **Signal generation:**
   ```bash
   # Count signals with Optuna vs fallback
   sudo journalctl -u freqtrade-feeder75a --since "1 hour ago" | grep -c "SIGNAL SENT.*optuna"
   sudo journalctl -u freqtrade-feeder75a --since "1 hour ago" | grep -c "SIGNAL SENT.*fallback"
   ```

3. **Signal quality:**
   ```bash
   # Check signal confidence scores
   sudo journalctl -u freqtrade-feeder75a --since "1 hour ago" | grep "Confidence:" | tail -20
   ```

4. **Consumer performance:**
   - Monitor BOT1/BOT2 win rate
   - Check if Optuna-sourced signals perform better

### Phase 4: Full Deployment (if validation successful)

**LONG feeders (Host 75):**
```bash
# Deploy to feeder75b, feeder75c
for service in freqtrade-feeder75b freqtrade-feeder75c; do
    sudo systemctl restart $service
    echo "Restarted $service"
    sleep 10
done

# Verify all running
sudo systemctl status freqtrade-feeder75a freqtrade-feeder75b freqtrade-feeder75c | grep "active (running)"
```

**SHORT feeders (Host 120):**
```bash
# Deploy to feeder120b, feeder120c
for service in freqtrade-feeder120b freqtrade-feeder120c; do
    sudo systemctl restart $service
    echo "Restarted $service"
    sleep 10
done

# Verify all running
sudo systemctl status freqtrade-feeder120a freqtrade-feeder120b freqtrade-feeder120c | grep "active (running)"
```

---

## 📈 Monitoring & Metrics

### Key Metrics to Track

1. **Optuna Parameter Distribution:**
   ```bash
   # Query InfluxDB for parameter ranges
   influx query '
   from(bucket: "OPTUNA_PARAMS")
   |> range(start: -7d)
   |> filter(fn: (r) => r._measurement == "parameters")
   |> filter(fn: (r) => r._field == "take_profit_pct" or r._field == "stop_loss_pct")
   |> group(columns: ["_field"])
   |> aggregateWindow(every: 1d, fn: mean)
   '
   ```

2. **Signal Source Distribution:**
   ```bash
   # Count Optuna vs fallback signals (last 24h)
   echo "LONG feeders:"
   sudo journalctl -u freqtrade-feeder75a -u freqtrade-feeder75b -u freqtrade-feeder75c --since "24 hours ago" | grep "SIGNAL SENT" | grep -c "optuna"
   sudo journalctl -u freqtrade-feeder75a -u freqtrade-feeder75b -u freqtrade-feeder75c --since "24 hours ago" | grep "SIGNAL SENT" | grep -c "fallback"
   
   echo "SHORT feeders:"
   sudo journalctl -u freqtrade-feeder120a -u freqtrade-feeder120b -u freqtrade-feeder120c --since "24 hours ago" | grep "SIGNAL SENT" | grep -c "optuna"
   sudo journalctl -u freqtrade-feeder120a -u freqtrade-feeder120b -u freqtrade-feeder120c --since "24 hours ago" | grep "SIGNAL SENT" | grep -c "fallback"
   ```

3. **Per-Pair Threshold Analysis:**
   ```bash
   # Example: Check BTC/USDT thresholds
   sudo journalctl -u freqtrade-feeder75a --since "1 hour ago" | grep "BTC/USDT" | grep "THRESHOLDS" -A 3
   ```

4. **Consumer Win Rate Comparison:**
   - Before Optuna feeders: Track baseline win rate
   - After Optuna feeders: Compare improvement

---

## 🔧 Troubleshooting

### Issue: "InfluxDB not ready, using fallback thresholds"

**Cause:** InfluxDB connection failure

**Fix:**
```bash
# Check InfluxDB status
curl -I http://192.168.3.6:8086/health

# Verify token and org in strategy file
grep -E "INFLUX_TOKEN|INFLUX_ORG" /home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_LONG.py

# Check network connectivity
ping 192.168.3.6
```

### Issue: "Loaded Optuna params for 0 pairs"

**Cause:** No data in OPTUNA_PARAMS bucket or query issue

**Fix:**
```bash
# Query InfluxDB directly
influx query '
from(bucket: "OPTUNA_PARAMS")
|> range(start: -90d)
|> filter(fn: (r) => r._measurement == "parameters")
|> limit(n: 5)
'

# If empty, run Optuna optimization first
# (Should already have data from BOT2 optimization)
```

### Issue: All signals use "fallback" source

**Cause:** Optuna params not matching pair names

**Fix:**
```bash
# Check pair format in InfluxDB
influx query '
from(bucket: "OPTUNA_PARAMS")
|> range(start: -7d)
|> filter(fn: (r) => r._measurement == "parameters")
|> keep(columns: ["pair"])
|> distinct(column: "pair")
|> limit(n: 10)
'

# Compare with feeder whitelist
sudo journalctl -u freqtrade-feeder75a | grep "whitelist" | tail -1
```

### Issue: HIGH percentage of filtered signals

**Cause:** Optuna thresholds too strict for some pairs

**Expected:** This is normal! Optuna-learned thresholds should be MORE selective than static 1%

**Action:**
- Monitor signal quality (do sent signals have higher win rate?)
- If too few signals (< 5 per day total), may need to adjust fallback values

---

## 📊 Expected Behavior

### Signal Volume Changes

**Before (static 1% threshold):**
- LONG feeders: ~50-100 signals/day
- SHORT feeders: ~50-100 signals/day

**After (Optuna thresholds):**
- Expected: 20-50% reduction in signal volume
- But: Higher quality signals (better win rate)

### Threshold Distribution

**Typical Optuna values** (based on BOT2 optimization):
- Take profit: 1-3% (some pairs higher for volatile assets)
- Stop loss: 5-10% (larger for high-volatility pairs)

**Pairs with tighter thresholds** (1-2%):
- Stable coins, low volatility
- High win rate historically

**Pairs with looser thresholds** (3-5%):
- High volatility (BTC, ETH in volatile markets)
- Lower historical win rate, need larger moves

---

## ✅ Success Criteria

### Deployment is successful if:

1. **✅ Optuna connection:** > 95% of signals use "optuna" source (not "fallback")
2. **✅ Signal quality:** Win rate improves by 5-10% within 7 days
3. **✅ Per-pair adaptation:** Different pairs use different thresholds appropriately
4. **✅ No errors:** No InfluxDB connection errors in logs
5. **✅ Fallback works:** If InfluxDB down, feeders continue with fallback values

### Rollback if:

1. **❌ Optuna connection fails:** > 50% signals use "fallback" after 24 hours
2. **❌ Win rate degrades:** Consumer win rate drops by > 5%
3. **❌ Signal volume drops too much:** < 10 signals/day total across all feeders
4. **❌ Errors in logs:** Persistent InfluxDB errors, crashes, or exceptions

**Rollback procedure:**
```bash
# LONG feeders
ssh freqai@192.168.3.75
cd /home/freqai/freqtrade/user_data/strategies
cp ProducerStrategyDualQuantile_LONG_backup_*.py ProducerStrategyDualQuantile_LONG.py
sudo systemctl restart freqtrade-feeder75a freqtrade-feeder75b freqtrade-feeder75c

# SHORT feeders  
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120
cd /home/freqai/freqtrade/user_data/strategies
cp ProducerStrategyDualQuantile_SHORT_backup_*.py ProducerStrategyDualQuantile_SHORT.py
sudo systemctl restart freqtrade-feeder120a freqtrade-feeder120b freqtrade-feeder120c
```

---

## 🎓 Key Insights

### Why This Works

1. **Unified optimization:** Same Optuna params used for both signal generation and trade exits
2. **Feedback loop:** Better signals → better trades → better Optuna params → better signals
3. **Per-pair adaptation:** High-volatility pairs get appropriate thresholds automatically
4. **Graceful degradation:** Fallback ensures system continues if Optuna unavailable

### Comparison to Previous Assessment

**Previous recommendation:** Don't implement Optuna for feeders (complexity > benefit)

**Current implementation:** Reuses existing Optuna params (low complexity, high benefit)

**Key difference:**
- Previous concern: Need separate optimization system for feeders
- Current solution: Reuse consumer optimization → simpler, creates feedback loop

This approach addresses the main concern (complexity) while keeping the benefits (per-pair optimization).

---

## 📝 Files Created

1. **LONG Strategy:** `user_data/strategies/ProducerStrategyDualQuantile_LONG_Optuna.py`
2. **SHORT Strategy:** `user_data/strategies/ProducerStrategyDualQuantile_SHORT_Optuna.py`
3. **This Guide:** `FEEDER_OPTUNA_DEPLOYMENT_GUIDE.md`

---

## 🔗 References

- **BOT2 Optuna Implementation:** `BOT2_ArtiSkyTrader_Optuna_Fixed.py`
- **Previous Assessment:** `FEEDER_OPTUNA_INTEGRATION_ASSESSMENT.md`
- **Session Context:** `SESSION_CONTEXT_BOT2_OPTUNA_2025-11-27.md`
- **InfluxDB Bucket:** `OPTUNA_PARAMS` on 192.168.3.6:8086

---

**Deployment Status:** Ready for Phase 1 testing  
**Next Step:** Deploy to feeder75a and feeder120a for 24-48h validation  
**Contact:** User for approval to proceed
