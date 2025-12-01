# BOT2 Fallback Issue Analysis
**Date:** 2025-11-27  
**Issue:** BOT2 is falling back to default values instead of using Optuna-optimized parameters from InfluxDB

---

## 🔍 PROBLEM STATEMENT

BOT2 (192.168.3.72) is not properly retrieving take profit and stop loss values from InfluxDB that were optimized by Optuna, causing it to fallback to default/hardcoded values.

---

## 📋 INVESTIGATION FINDINGS

### Finding 1: Architecture Documentation Gap

**Issue:** The BOT1_BOT2_PARAMETER_SHARING_ARCHITECTURE.md document describes how BOT1 and BOT2 should retrieve parameters from InfluxDB, but does NOT clarify that there are **TWO DIFFERENT versions** of ArtiSkyTrader.py:

1. **Feeder Version** (ProducerStrategy)
   - Location: Local `/home/ubuntu/freqtrade/user_data/strategies/ArtiSkyTrader.py`
   - Purpose: Generates trading signals using FreqAI models
   - Sends signals to Aggregator (192.168.3.80)
   - Does NOT execute trades
   - Does NOT need Optuna/InfluxDB parameter retrieval

2. **Executor Version** (ConsumerStrategy)
   - Location: BOT1 (192.168.3.71) and BOT2 (192.168.3.72) servers
   - Purpose: Receives signals from Aggregator and executes trades
   - Should implement:
     - `custom_exit()` method
     - Optuna parameter loading
     - Stateless Manager integration
     - InfluxDB queries for optimized parameters

### Finding 2: Strategy Class Naming Confusion

**From architecture document:**
```python
class ArtiSkyTrader(IStrategy):  # ❌ Document uses this name
```

**Actual implementation on BOT2:**
```python
class ConsumerStrategy(IStrategy):  # ✅ Actual class name
```

The architecture document references `ArtiSkyTrader` class but BOT2 actually uses `ConsumerStrategy` class.

### Finding 3: Missing Implementation

Based on the architecture document, BOT2's ConsumerStrategy should have this structure:

```python
class ConsumerStrategy(IStrategy):
    def __init__(self, config):
        super().__init__(config)
        
        # 1. Initialize InfluxDB reader
        self.influx_reader = UltimateSignalQualityReaderV3(
            influx_url="http://192.168.3.6:8086",
            influx_token="your_token",
            influx_org="freqtrade",
            influx_bucket="BOT33_trades"
        )
        
        # 2. Initialize Stateless Manager
        self.stateless_params = InfluxDBParameterManager(
            influx_reader=self.influx_reader,
            cache_duration_seconds=30
        )
        
        # 3. Load Optuna parameters
        self.optuna_params = {}
        try:
            optimizer = OptunaParameterOptimizer(
                influx_url="http://192.168.3.6:8086",
                influx_token="your_token"
            )
            self.optuna_params = optimizer.load_results()
        except Exception as e:
            logger.info(f"Optuna not available yet: {e}")
    
    def custom_exit(self, pair, trade, current_time, current_rate, 
                    current_profit, **kwargs):
        """
        PRIORITY SYSTEM:
        1. Try Optuna params (per-pair optimized)
        2. Fallback to Stateless Manager (leverage-aware)
        3. Fallback to default logic
        """
        
        leverage = trade.leverage or 1.0
        
        # Priority 1: Optuna
        if pair in self.optuna_params:
            params = self.optuna_params[pair]
            stop_position = params['stop_loss_pct']
            target_position = params['take_profit_pct']
            
            # Convert to account level
            stop_account = stop_position * leverage
            target_account = target_position * leverage
            
            if current_profit >= target_account:
                return 'roi_optuna'
        
        # Priority 2: Stateless Manager
        else:
            leverage_params = self.stateless_params.get_leverage_params(leverage)
            
            if leverage_params:
                stop_position = leverage_params['stop_loss_pct']
                target_position = leverage_params['take_profit_pct']
                
                # Convert to account level
                stop_account = stop_position * leverage
                target_account = target_position * leverage
                
                if current_profit >= target_account:
                    return 'roi'
        
        # Priority 3: Fallback
        return None
```

**Current Status:** Need to verify if BOT2's ConsumerStrategy has this implementation.

---

## 🔧 ROOT CAUSE ANALYSIS

### Possible Causes for Fallback:

1. **Missing Implementation**
   - ConsumerStrategy on BOT2 doesn't have `custom_exit()` method
   - OR custom_exit() exists but doesn't implement InfluxDB parameter retrieval

2. **Missing Dependencies**
   - Required classes not imported:
     - `UltimateSignalQualityReaderV3`
     - `InfluxDBParameterManager`
     - `OptunaParameterOptimizer`

3. **InfluxDB Connection Issues**
   - Cannot connect to InfluxDB at 192.168.3.6:8086
   - Authentication failing
   - Network connectivity problems

4. **No Optuna Data Available**
   - Optuna optimizer hasn't run yet
   - No optimized parameters exist in OPTUNA_PARAMS bucket
   - Falling back to Stateless Manager should work, but doesn't

5. **No BOT33 Trades Data**
   - BOT33_trades bucket is empty
   - Stateless Manager queries return no results
   - Falling back to default values

6. **Configuration Issues**
   - Wrong InfluxDB URL/token/org in config
   - Cache expiration too aggressive
   - Wrong bucket names

---

## 📊 VERIFICATION CHECKLIST

To diagnose the exact issue, check:

### On BOT2 Server (192.168.3.72):

```bash
# 1. Check ConsumerStrategy has custom_exit
ssh ubuntu@192.168.3.72
cd /home/ubuntu/freqtrade
grep -A 50 "def custom_exit" user_data/strategies/ArtiSkyTrader.py

# 2. Check InfluxDB imports
grep -E "InfluxDB|Optuna|Stateless" user_data/strategies/ArtiSkyTrader.py

# 3. Check InfluxDB connectivity
curl http://192.168.3.6:8086/ping
# Should return: 204 No Content

# 4. Check BOT2 logs for InfluxDB errors
tail -100 /home/ubuntu/freqtrade/bot2.log | grep -i "influx\|optuna\|fallback"

# 5. Check config file has InfluxDB settings
cat /home/ubuntu/freqtrade/config_bot2.json | grep -i influx
```

### On InfluxDB Server (192.168.3.6):

```bash
# 6. Check Optuna parameters exist
influx query 'from(bucket:"OPTUNA_PARAMS") |> range(start:-30d) |> count()'

# 7. Check BOT33 trades exist
influx query 'from(bucket:"BOT33_trades") |> range(start:-30d) |> count()'

# 8. Verify parameter structure
influx query 'from(bucket:"OPTUNA_PARAMS") |> range(start:-1d) |> limit(n:1)'
```

### On BOT3 Server (192.168.3.33):

```bash
# 9. Check Optuna optimizer is running
ps aux | grep optuna

# 10. Check when last optimization ran
tail -50 /home/ubuntu/freqtrade/user_data/logs/optuna_optimizer.log
```

---

## 🔨 RECOMMENDED FIXES

### Fix 1: Verify Implementation Exists

**If custom_exit() is missing or incomplete:**

1. Copy the reference implementation from architecture document
2. Ensure all required imports are present
3. Test locally first, then deploy to BOT2

### Fix 2: Add Missing Dependencies

**Required files on BOT2:**

```python
# user_data/controllers/bot3_ultimate_adaptive_v6_hybird.py
class UltimateSignalQualityReaderV3:
    # InfluxDB reader implementation
    pass

# user_data/controllers/influxdb_parameter_manager.py
class InfluxDBParameterManager:
    # Stateless Manager implementation
    pass

# user_data/services/optuna_optimizer_service.py
class OptunaParameterOptimizer:
    # Optuna parameter loader
    pass
```

### Fix 3: Fix InfluxDB Configuration

**In BOT2's config file:**

```json
{
  "influxdb": {
    "url": "http://192.168.3.6:8086",
    "token": "your_actual_token_here",
    "org": "freqtrade",
    "buckets": {
      "trades": "BOT33_trades",
      "optuna": "OPTUNA_PARAMS"
    }
  }
}
```

### Fix 4: Add Fallback Logging

**To identify where fallback occurs:**

```python
def custom_exit(self, pair, trade, current_time, current_rate, 
                current_profit, **kwargs):
    leverage = trade.leverage or 1.0
    
    # Try Optuna
    if pair in self.optuna_params:
        logger.info(f"✅ Using Optuna params for {pair}")
        # ... implementation
    
    # Try Stateless Manager  
    else:
        logger.warning(f"⚠️ No Optuna params for {pair}, trying Stateless Manager")
        leverage_params = self.stateless_params.get_leverage_params(leverage)
        
        if leverage_params:
            logger.info(f"✅ Using Stateless Manager for leverage {leverage}x")
            # ... implementation
        else:
            logger.error(f"❌ FALLBACK: No params available for {pair} @ {leverage}x")
            # ... fallback logic
```

---

## 📝 ARCHITECTURE DOCUMENT UPDATES NEEDED

### Update 1: Clarify Two ArtiSkyTrader Versions

Add section to BOT1_BOT2_PARAMETER_SHARING_ARCHITECTURE.md:

```markdown
## CRITICAL: Two Versions of ArtiSkyTrader.py

### Version 1: Feeder Strategy (ProducerStrategy)
- **Location:** Feeder hosts (192.168.3.75, 192.168.3.120)
- **Class Name:** ProducerStrategy
- **Purpose:** Generate trading signals using FreqAI
- **Sends To:** Aggregator (192.168.3.80)
- **Does NOT:** Execute trades, retrieve Optuna params

### Version 2: Executor Strategy (ConsumerStrategy)
- **Location:** BOT1 (192.168.3.71), BOT2 (192.168.3.72)  
- **Class Name:** ConsumerStrategy
- **Purpose:** Execute trades from Aggregator signals
- **Must Have:** 
  - custom_exit() method
  - InfluxDB parameter retrieval
  - Optuna integration
  - Stateless Manager fallback
```

### Update 2: Add Troubleshooting Section

```markdown
## Troubleshooting: BOT1/BOT2 Not Using InfluxDB Parameters

### Symptom: Fallback to default values

**Check 1: Strategy has custom_exit()**
grep "def custom_exit" /home/ubuntu/freqtrade/user_data/strategies/ArtiSkyTrader.py

**Check 2: InfluxDB connectivity**  
curl http://192.168.3.6:8086/ping

**Check 3: Optuna data exists**
influx query 'from(bucket:"OPTUNA_PARAMS") |> range(start:-1d) |> count()'

**Check 4: BOT logs for errors**
tail -100 bot2.log | grep -i "influx\|optuna\|fallback"
```

### Update 3: Add File Location Reference

```markdown
## File Locations Reference

### Feeder Hosts (75, 120):
- Strategy: `/home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_LONG.py`
- Also: `/home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_SHORT.py`

### BOT1/BOT2 Hosts (71, 72):
- Strategy: `/home/ubuntu/freqtrade/user_data/strategies/ArtiSkyTrader.py`
  - Contains: ConsumerStrategy class
  - Must have: custom_exit() with InfluxDB integration

### BOT3 Host (33):
- Optuna Service: `/home/ubuntu/freqtrade/user_data/services/optuna_optimizer_service.py`
- Controllers: `/home/ubuntu/freqtrade/user_data/controllers/`
  - influxdb_parameter_manager.py
  - bot3_ultimate_adaptive_v6_hybird.py
```

---

## 🎯 ACTION PLAN

### Phase 1: Diagnosis (15 minutes)

1. SSH to BOT2 (192.168.3.72)
2. Check if ConsumerStrategy has custom_exit() method
3. Check InfluxDB connectivity
4. Check BOT2 logs for errors

### Phase 2: Fix Implementation (30 minutes)

1. If custom_exit() missing: Add it based on architecture document
2. If InfluxDB unreachable: Fix network/auth
3. If no Optuna data: Run Optuna optimizer manually
4. If no BOT33 trades: Wait for trades to accumulate

### Phase 3: Testing (15 minutes)

1. Restart BOT2 service
2. Open a test trade
3. Monitor logs for Optuna/Stateless Manager usage
4. Verify exit uses InfluxDB parameters

### Phase 4: Documentation (15 minutes)

1. Update BOT1_BOT2_PARAMETER_SHARING_ARCHITECTURE.md
2. Add discovered issues to this document
3. Create troubleshooting checklist
4. Document file locations clearly

---

## 🚨 IMMEDIATE NEXT STEPS

1. **Get access to BOT2 server** to read ConsumerStrategy implementation
2. **Verify custom_exit() exists** and has InfluxDB integration
3. **Check InfluxDB connectivity** from BOT2
4. **Review BOT2 logs** for actual error messages
5. **Compare with BOT1** to see if it's working correctly

---

**Status:** Awaiting access to BOT2 server to verify ConsumerStrategy implementation  
**Priority:** HIGH - BOT2 is currently trading with incorrect take profit/stop loss values  
**Risk:** Potential losses due to suboptimal exit parameters
