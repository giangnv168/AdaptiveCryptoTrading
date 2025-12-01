# BOT1 OPTUNA - Fix Strategy File Bucket Query (2025-11-29)

## 🎯 TASK: Fix BOT1 Optuna Integration

Based on BOT2 investigation, BOT1 likely has the same issue:
- Strategy file querying old OPTUNA_PARAMS bucket instead of OPTUNA_BOT1

## 📋 BACKGROUND - BOT2 Issue & Fix

### What We Found on BOT2

1. **Symptom:** BOT2 showing "roi_fallback" exit reasons despite Optuna data existing
2. **Root Cause:** Strategy file (ArtiSkyTrader.py) querying wrong bucket
   - Was querying: `OPTUNA_PARAMS` (old, empty bucket)
   - Should query: `OPTUNA_BOT2` (correct bucket with data)
3. **Fix:** Updated ArtiSkyTrader.py line 121 to query OPTUNA_BOT2

### BOT1 Likely Has Same Issue

BOT1 probably:
- Uses a strategy file (need to identify which one)
- That strategy queries OPTUNA_PARAMS instead of OPTUNA_BOT1
- Result: Fallback parameters instead of optimized ones

## 🔍 INVESTIGATION STEPS

### Step 1: Identify BOT1's Actual Strategy File

```bash
# SSH to BOT1
ssh ubuntu@192.168.3.7  # BOT1 server

# Check BOT1 logs to see which strategy it's using
sudo journalctl -u freqtrade -n 100 | grep -E "Strategy|INFO" | head -20

# Look for strategy class name in logs (e.g., "ArtiSkyTrader", "ConsumerStrategy", etc.)
```

### Step 2: List Strategy Files on BOT1

```bash
# List all strategy files
ls -la user_data/strategies/*.py

# Look for files matching the strategy name from logs
```

### Step 3: Check What Bucket the Strategy Queries

```bash
# Check for OPTUNA references in the strategy file
# Replace STRATEGY_FILE.py with actual filename
grep -n 'OPTUNA' user_data/strategies/STRATEGY_FILE.py

# Look for line like:
# from(bucket: "OPTUNA_PARAMS")  # ← This is WRONG for BOT1!
```

### Step 4: Verify OPTUNA_BOT1 Bucket Has Data

```bash
# Check if OPTUNA_BOT1 bucket exists and has data
cd /home/ubuntu/freqtrade
.venv/bin/python3 -c "
from influxdb_client import InfluxDBClient

query = '''
from(bucket: \"OPTUNA_BOT1\")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == \"optimized_parameters\")
  |> group(columns: [\"pair\"])
  |> count()
'''

with InfluxDBClient(url='http://192.168.3.6:8086', 
                    token='uvAgH7X2WJxE_RGmZC4wUEuNLsZX2e9Uyhxg1lB9euW4I2qV6RjUZmQlzSTOJPoHbO2tLvarifULUCgvArVEXA==', 
                    org='ArtiSky') as client:
    result = client.query_api().query(query)
    pairs = set()
    for table in result:
        for record in table.records:
            pair = record.values.get('pair')
            if pair:
                pairs.add(pair)
    
    print(f'OPTUNA_BOT1 bucket contains {len(pairs)} pairs:')
    for p in sorted(pairs):
        print(f'  ✅ {p}')
"
```

## ✅ THE FIX

### Update Strategy File to Query OPTUNA_BOT1

```bash
# SSH to BOT1
ssh ubuntu@192.168.3.7

cd /home/ubuntu/freqtrade

# Update the strategy file (replace STRATEGY_FILE.py with actual filename)
sed -i 's/from(bucket: "OPTUNA_PARAMS")/from(bucket: "OPTUNA_BOT1")/' user_data/strategies/STRATEGY_FILE.py

# Verify the change
grep -n 'OPTUNA_BOT1' user_data/strategies/STRATEGY_FILE.py

# Should show:
# 121:            from(bucket: "OPTUNA_BOT1")  ✅
```

### Restart BOT1

```bash
sudo systemctl restart freqtrade

# Wait a few seconds for startup
sleep 5

# Check logs for Optuna initialization
sudo journalctl -u freqtrade -n 50 | grep -E "Optuna|pairs loaded"

# Expected output:
# ✅ BOT1 WITH OPTUNA INITIALIZING...
# 🎯 Optuna integration active: 40+ pairs loaded
```

## 🎯 EXPECTED RESULTS

### Before Fix
```
Exit Reason: roi_fallback
Exit Reason: stop_loss_fallback
Exit Reason: target_fallback
```

### After Fix
```
Exit Reason: roi_optuna
Exit Reason: stop_loss_optuna
Exit Reason: target_optuna
```

## 📊 VERIFICATION

### Check BOT1 Trades After Restart

```bash
# Monitor BOT1 logs for new trades
sudo journalctl -u freqtrade -f

# Look for exit reasons - should say "optuna" not "fallback"
```

### Verify Specific Pair Parameters

```bash
# Check a specific pair's parameters in OPTUNA_BOT1
cd /home/ubuntu/freqtrade
.venv/bin/python3 -c "
from influxdb_client import InfluxDBClient

# Replace with actual trading pair
pair = 'BTC/USDT:USDT'

query = f'''
from(bucket: \"OPTUNA_BOT1\")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == \"optimized_parameters\")
  |> filter(fn: (r) => r.pair == \"{pair}\")
  |> last()
'''

with InfluxDBClient(url='http://192.168.3.6:8086', 
                    token='uvAgH7X2WJxE_RGmZC4wUEuNLsZX2e9Uyhxg1lB9euW4I2qV6RjUZmQlzSTOJPoHbO2tLvarifULUCgvArVEXA==', 
                    org='ArtiSky') as client:
    result = client.query_api().query(query)
    
    if result and len(result) > 0:
        print(f'✅ {pair} found in OPTUNA_BOT1:')
        for table in result:
            for record in table.records:
                field = record.get_field()
                value = record.get_value()
                if field in ['stop_loss', 'take_profit', 'win_rate', 'trades_count']:
                    print(f'  {field}: {value}')
    else:
        print(f'❌ {pair} NOT found in OPTUNA_BOT1')
"
```

## 🎓 KEY POINTS

1. **Strategy File Mismatch:** Always verify which strategy file the bot actually uses
2. **Check Logs:** Bot logs show the actual strategy class name
3. **Bucket Names:** BOT1 → OPTUNA_BOT1, BOT2 → OPTUNA_BOT2, BOT3 → OPTUNA_BOT3
4. **No Errors:** Wrong bucket query doesn't throw errors, just silently uses fallback

## 📝 CHECKLIST

- [ ] SSH to BOT1 (192.168.3.7)
- [ ] Check logs to identify actual strategy file
- [ ] List strategy files and find the correct one
- [ ] Check what bucket it queries (should be OPTUNA_PARAMS - WRONG)
- [ ] Verify OPTUNA_BOT1 bucket has data
- [ ] Update strategy file: OPTUNA_PARAMS → OPTUNA_BOT1
- [ ] Verify update applied correctly
- [ ] Restart BOT1
- [ ] Check startup logs (should load 40+ pairs)
- [ ] Monitor trades (exit reasons should say "optuna" not "fallback")
- [ ] ✅ Complete!

## 🚀 QUICK FIX SCRIPT

```bash
#!/bin/bash
# Quick fix for BOT1 Optuna bucket issue

echo "🔍 Checking BOT1 strategy file..."

# SSH to BOT1 and run fix
ssh ubuntu@192.168.3.7 << 'ENDSSH'
cd /home/ubuntu/freqtrade

# Find strategy files
echo "Available strategy files:"
ls -la user_data/strategies/*.py | grep -v '__pycache__'

echo ""
echo "Checking for OPTUNA references..."
for file in user_data/strategies/*.py; do
    if grep -q "OPTUNA_PARAMS" "$file"; then
        echo "❌ Found OPTUNA_PARAMS in: $file"
        echo "   Updating to OPTUNA_BOT1..."
        sed -i 's/from(bucket: "OPTUNA_PARAMS")/from(bucket: "OPTUNA_BOT1")/' "$file"
        echo "   ✅ Updated!"
    fi
done

echo ""
echo "Verifying changes..."
grep -r "OPTUNA_BOT1" user_data/strategies/*.py

echo ""
echo "✅ Fix complete! Now restart BOT1:"
echo "   sudo systemctl restart freqtrade"

ENDSSH
```

## 📅 RELATED ISSUES

- **BOT2 Fix:** BOT2_OPTUNA_WRONG_STRATEGY_FILE_FIX_2025-11-29.md
- **Optimizer Fix:** BOT2_OPTUNA_INFLUXDB_TIME_BUG_COMPLETE_2025-11-28.md
- **Schema Fix:** BOT2_OPTUNA_SCHEMA_COLLISION_FIX_COMPLETE.md

---

**Date:** 2025-11-29  
**Priority:** HIGH - BOT1 likely using fallback parameters  
**Estimated Time:** 10 minutes  
**Status:** Ready to execute
