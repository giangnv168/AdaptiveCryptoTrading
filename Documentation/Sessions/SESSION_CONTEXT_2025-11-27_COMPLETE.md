# Session Context - November 27, 2025
**Status:** GPU Installation COMPLETE ✅ | Optuna Integration PLANNED ⏳

---

## Session Summary

This session accomplished two major objectives:

1. ✅ **COMPLETED:** Fixed GPU detection and installation issues on Host 75
2. ⏳ **IN PROGRESS:** Identified and planned Optuna parameter integration for feeders

---

## 1. GPU Installation - COMPLETE ✅

### Problem
- **Host 75:** NVIDIA Xid 119 GSP Timeout errors
- **Symptoms:** 
  - nvidia-smi showing "ERR!" for all GPU metrics
  - OpenCL showing 0 platforms (GPU not detectable)
  - LightGBM unable to use GPU: "No OpenCL device found"

### Root Cause
**Insufficient PCIe MMIO address space in ESXi VM configuration**

- Host 120 (1 GPU, working): `pciHole.dynStart = 3072`
- Host 75 (2 GPUs, failing): `pciHole.dynStart = 2787` ❌
- **Solution:** Increased Host 75 to `pciHole.dynStart = 3584` ✅

### Solution Applied

**ESXi Configuration Change:**
```bash
# ESXi Host: 192.168.3.39 (root/Tuong@168)
# VM: Producer75 (ID: 110)
# File: /vmfs/volumes/Samsung860/Producer75/Producer75.vmx

# Changed:
pciHole.dynStart = "2787"  # Before (insufficient)
pciHole.dynStart = "3584"  # After (adequate for 2 GPUs)

# Backup created: Producer75.vmx.backup
```

**Model Configuration:**
```bash
# File: /home/freqai/freqtrade/user_data/freqaimodels/LightGBMQuantileMultiTarget.py
# Line 120: Changed verbose from -1 to 2 for GPU verification

'device_type': 'gpu',
'gpu_platform_id': 0,
'gpu_device_id': 0,
'verbose': 2,  # Shows "[LightGBM] [Info] This is the GPU trainer!!"
```

### Results

**Host 75 GPU Status - WORKING ✅**
```
GPU 0: Tesla T4, 50°C, 26W, 287MiB used, 8% utilization
GPU 1: Tesla T4, 38°C, 10W, 3MiB used, 0% utilization
OpenCL: 1 platform detected with 2 Tesla T4 devices
Python Processes: 2 processes using GPU (142MiB each)
```

**Host 120 GPU Status - WORKING ✅**
```
GPU 0: Tesla T4, 73°C, 32W, 287MiB used, 6% utilization
Python Processes: 2 processes using GPU (142MiB each)
```

### Documentation Created
- **File:** `GPU_INSTALLATION_TROUBLESHOOTING_GUIDE_2025-11-27.md` (700+ lines)
- **Contents:** Complete troubleshooting guide, ESXi configuration, monitoring procedures

---

## 2. Optuna Integration Gap - IDENTIFIED ⏳

### Problem Discovered

**Feeder strategies are NOT using Optuna-optimized thresholds from InfluxDB!**

**Current Implementation (INCORRECT):**
```python
# File: ProducerStrategyDualQuantile_LONG.py (lines 77-78)
PROFIT_TARGET_PCT = 0.01      # 1% - STATIC, NOT from Optuna!
RISK_PROTECTION_PCT = 0.0     # 0% - STATIC, NOT from Optuna!

# These static values are used for ALL pairs:
upside_threshold = current_close * (1 + self.PROFIT_TARGET_PCT)
downside_threshold = current_low * (1 + self.RISK_PROTECTION_PCT)
```

**Impact:**
- All pairs use same 1% profit threshold
- All pairs use same 0% drawdown threshold
- Optuna optimization results in InfluxDB are **IGNORED**
- No per-pair optimization happening on feeders

**What Should Happen:**
- Load per-pair optimized thresholds from InfluxDB OPTUNA_PARAMS bucket
- Use Optuna values (Priority 1) or fallback to defaults (Priority 2)
- Same architecture that BOT3 uses successfully

### Solution Designed

**Add to Both LONG and SHORT Feeder Strategies:**

```python
# 1. Add imports after existing imports
from influxdb_client import InfluxDBClient
import time

# 2. Add __init__ method (after std_dev_multiplier_sell line)
def __init__(self, config: dict) -> None:
    """Initialize strategy with Optuna parameter loading from InfluxDB"""
    super().__init__(config)
    
    # Connect to InfluxDB
    try:
        self.influx_client = InfluxDBClient(
            url="http://192.168.3.6:8086",
            token="freqtradeInfluxToken2024",
            org="freqtrade"
        )
        self.optuna_params = {}
        self.last_params_load = 0
        self._load_optuna_params()
        logger.info(f"🎯 Optuna integration active: {len(self.optuna_params)} pairs loaded")
    except Exception as e:
        logger.error(f"❌ Optuna initialization failed: {e}")
        self.optuna_params = {}

# 3. Add parameter loading method
def _load_optuna_params(self) -> None:
    """Load per-pair optimized thresholds from InfluxDB OPTUNA_PARAMS bucket"""
    try:
        query = '''
        from(bucket: "OPTUNA_PARAMS")
        |> range(start: -90d)
        |> filter(fn: (r) => r._measurement == "parameters")
        |> group(columns: ["pair"])
        |> last()
        '''
        
        result = self.influx_client.query_api().query(query)
        
        for table in result:
            for record in table.records:
                pair = record.values.get('pair')
                if pair:
                    self.optuna_params[pair] = {
                        'profit_target_pct': abs(record.values.get('take_profit_pct', 0.01)),
                        'risk_protection_pct': abs(record.values.get('stop_loss_pct', 0.0))
                    }
        
        self.last_params_load = time.time()
        
    except Exception as e:
        logger.error(f"❌ Failed to load Optuna params: {e}")

# 4. Add threshold getter method
def _get_thresholds_for_pair(self, pair: str) -> tuple:
    """Get profit/risk thresholds for a specific pair (Optuna or defaults)"""
    
    # Reload params every 30 minutes
    if hasattr(self, 'last_params_load') and time.time() - self.last_params_load > 1800:
        self._load_optuna_params()
    
    # Priority 1: Optuna per-pair params
    if hasattr(self, 'optuna_params') and pair in self.optuna_params:
        params = self.optuna_params[pair]
        return (params['profit_target_pct'], params['risk_protection_pct'])
    
    # Priority 2: Defaults
    return (self.PROFIT_TARGET_PCT, self.RISK_PROTECTION_PCT)

# 5. Modify populate_entry_trend to use dynamic thresholds
# CHANGE FROM:
upside_threshold = current_close * (1 + self.PROFIT_TARGET_PCT)
downside_threshold = current_low * (1 + self.RISK_PROTECTION_PCT)

# CHANGE TO:
# Get dynamic thresholds from Optuna (per-pair) or defaults
profit_target, risk_protection = self._get_thresholds_for_pair(pair)
upside_threshold = current_close * (1 + profit_target)
downside_threshold = current_low * (1 + risk_protection)
```

### Files to Modify

**Host 75 (192.168.3.75) - LONG Strategy:**
```
/home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_LONG.py
Services: feeder75a, feeder75b, feeder75c
Credentials: freqai / NewHopes@168
```

**Host 120 (192.168.3.120) - SHORT Strategy:**
```
/home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_SHORT.py
Services: feeder120a, feeder120b, feeder120c
Credentials: freqai / (SSH key: ~/.ssh/id_rsa_host120)
```

### Deployment Steps (NEXT SESSION)

1. **Backup Current Strategies**
   ```bash
   ssh freqai@192.168.3.75 "cp /home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_LONG.py{,.backup}"
   ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120 "cp /home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_SHORT.py{,.backup}"
   ```

2. **Modify Strategies** (add code from Solution Designed section above)

3. **Restart Services**
   ```bash
   # Host 75
   ssh freqai@192.168.3.75 "echo 'NewHopes@168' | sudo -S systemctl restart 'freqtrade-feeder75*.service'"
   
   # Host 120
   ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120 "sudo systemctl restart 'freqtrade-feeder120*.service'"
   ```

4. **Verify Optuna Loading**
   ```bash
   # Check logs for Optuna loading messages
   ssh freqai@192.168.3.75 "journalctl -u freqtrade-feeder75a.service -f | grep -i optuna"
   
   # Should see: "🎯 Optuna integration active: XX pairs loaded"
   ```

5. **Monitor Signal Generation**
   ```bash
   # Watch for dynamic threshold usage in logs
   ssh freqai@192.168.3.75 "journalctl -u freqtrade-feeder75a.service -f | grep -i 'Using Optuna thresholds'"
   ```

### Expected Impact

**Before (Current State):**
- BTC/USDT: 1.0% profit target (static)
- ETH/USDT: 1.0% profit target (static)
- ADA/USDT: 1.0% profit target (static)
- **All pairs treated the same**

**After (With Optuna):**
- BTC/USDT: 2.0% profit target (from Optuna)
- ETH/USDT: 1.8% profit target (from Optuna)
- ADA/USDT: 2.5% profit target (from Optuna)
- **Each pair individually optimized**

---

## System Architecture Reference

### Current Infrastructure

**Hosts:**
- **192.168.3.75** (Host 75) - 2× Tesla T4 GPUs - Feeds LONG signals
- **192.168.3.120** (Host 120) - 1× Tesla T4 GPU - Feeds SHORT signals
- **192.168.3.6** - InfluxDB (central database)
- **192.168.3.71** - BOT1 (LONG consumer)
- **192.168.3.72** - BOT2 (SHORT consumer)
- **192.168.3.33** - BOT3 (Executor with Optuna optimization)

**Data Flow:**
```
Optuna Optimizer (BOT3)
  ↓ writes optimized parameters
InfluxDB OPTUNA_PARAMS bucket
  ↓ should be read by (NOT IMPLEMENTED YET!)
Feeder Strategies (75a/b/c, 120a/b/c)
  ↓ sends signals
Aggregator (192.168.3.80)
  ↓ distributes
BOT1 (LONG) / BOT2 (SHORT)
```

### Access Credentials

**Host 75:**
- SSH: `freqai@192.168.3.75` / `NewHopes@168`
- Sudo: `echo 'NewHopes@168' | sudo -S <command>`

**Host 120:**
- SSH: `ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120`

**ESXi Hosts:**
- Host 75 ESXi: `root@192.168.3.39` / `Tuong@168`
- Host 120 ESXi: `root@192.168.3.37` / `Tuong@168`

**InfluxDB:**
- URL: `http://192.168.3.6:8086`
- Token: `freqtradeInfluxToken2024`
- Org: `freqtrade`

---

## Key Lessons & Best Practices

### From GPU Installation

1. **VM GPU Passthrough Requires Adequate MMIO Space**
   - Formula: ~1500 MB per Tesla T4 GPU minimum
   - More GPUs = More VRAM = More MMIO space needed

2. **Always Test Configuration Changes Independently**
   - Verified GPU hardware first (nvidia-smi)
   - Then OpenCL detection (clinfo)
   - Then LightGBM manual test
   - Finally integrated with FreqAI

3. **Enable Verbose Logging During Troubleshooting**
   - LightGBM verbose=2 shows GPU initialization messages
   - Critical for confirming actual device usage

### From Optuna Integration Analysis

1. **Don't Assume - Verify Implementation**
   - User correctly asked me to review actual code
   - Found gap between documentation and implementation
   - Infrastructure exists but wasn't being used by feeders

2. **Session Context Documentation is Critical**
   - Complex multi-session work requires detailed documentation
   - Include: problem, solution, code changes, deployment steps
   - From now on: Always create session context files

---

## Next Session Priorities

### IMMEDIATE (Priority 1)
1. **Complete Optuna Integration for Feeders**
   - Modify ProducerStrategyDualQuantile_LONG.py (Host 75)
   - Modify ProducerStrategyDualQuantile_SHORT.py (Host 120)
   - Deploy to all 6 services
   - Verify Optuna parameters are loaded and used

### HIGH (Priority 2)
2. **Validate Optuna Integration**
   - Monitor logs for Optuna loading confirmation
   - Verify per-pair thresholds are being used
   - Compare signal generation before/after
   - Document any issues

### MEDIUM (Priority 3)
3. **Consider Additional Entry Quality Improvements**
   - Add LightGBM model performance metrics (R², MAE, RMSE)
   - Implement prediction confidence scoring
   - Track recent prediction accuracy vs actual outcomes

---

## Technical Notes

### InfluxDB Optuna Parameters Schema
```
Bucket: OPTUNA_PARAMS
Measurement: parameters
Tags:
  - pair (e.g., "BTC/USDT:USDT")
Fields:
  - take_profit_pct (float, position-level percentage)
  - stop_loss_pct (float, position-level percentage, negative)
  - win_rate (float, 0-1)
  - profit_factor (float)
  - total_trades (int)
```

### LightGBM GPU Configuration
```python
lgb_params = {
    'objective': 'quantile',
    'alpha': alpha,
    'verbose': 2,  # -1 for production, 2 for debugging
    'device_type': 'gpu',  # 'cpu' to disable
    'gpu_platform_id': 0,  # OpenCL platform ID
    'gpu_device_id': 0,    # GPU device (0 or 1 for Host 75)
}
```

---

## Document History

| Date | Session | Key Accomplishments |
|------|---------|-------------------|
| 2025-11-27 | This Session | GPU installation complete, Optuna gap identified, implementation planned |

---

**END OF SESSION CONTEXT**

**Next session: Implement Optuna parameter loading for all feeders following the plan above.**
