# 10 Feeder Service Conversion - Complete Plan
**Based on Actual Running Process Investigation**

Date: 2025-11-23
Credentials: freqai / NewHopes@168

---

## INVESTIGATION SUMMARY

### Running Feeders: 5 out of 10 (All running for 11+ days!)

**LONG Feeders Running:**
1. ✅ Feeder 74 (MOMENTUM) - Running since Jan 11
2. ✅ Feeder 79 (VOLATILITY) - Running since Jan 11

**SHORT Feeders Running:**
3. ✅ Feeder 120 (ProducerStrategy) - Running since Jan 11
4. ✅ Feeder 121 (MEANREV) - Running since Jan 11
5. ✅ Feeder 124 (MTF) - Running since Jan 11

**Stopped Feeders: 5 out of 10**
- ❌ Feeder 73, 75, 108 (LONG)
- ❌ Feeder 122, 123 (SHORT)

---

## EXACT CONFIGURATIONS FROM RUNNING PROCESSES

### Feeder 74 (192.168.3.73) - LONG MOMENTUM
```bash
PID: 75827
Uptime: Since Jan 11, 2018 (11+ days)
CPU Hours: 45,628 hours
Memory: 12.6 GB

Exact Command:
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ArtiSkyTrader_Feeder_LONG_74_MOMENTUM \
  --config config_freqai_LightGBM_LONG_74.json \
  --freqaimodel LightGBMClassifier_Optuna \
  --logfile=dryrun.log

Working Directory: /home/freqai/freqtrade
Python: /home/freqai/freqtrade/.env/bin/python3.10
Terminal: pts/0 (manual process, no service!)
```

### Feeder 79 (192.168.3.79) - LONG VOLATILITY
```bash
PID: 36437
Uptime: Since Jan 11, 2018
CPU Hours: 43,901 hours
Memory: 12.8 GB

Exact Command:
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ArtiSkyTrader_Feeder_LONG_79_VOLATILITY \
  --config config_freqai_LightGBM_LONG_79.json \
  --freqaimodel LightGBMClassifier_Optuna \
  --logfile=dryrun.log

Working Directory: /home/freqai/freqtrade
Python: /home/freqai/freqtrade/.env/bin/python3.10
Terminal: pts/0 (manual process, no service!)
```

### Feeder 120 (192.168.3.120) - SHORT Producer
```bash
PID: 125658
Uptime: Since Jan 11, 2019
CPU Hours: 58,200 hours (highest!)
Memory: 49.1 GB (very high!)

Exact Command:
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ProducerStrategy \
  --config config_freqai_LightGBM.json \
  --freqaimodel LightGBMClassifier_SHORT \
  --logfile=dryrun.log

Working Directory: /home/freqai/freqtrade
Python: /home/freqai/freqtrade/.env/bin/python3.10
Terminal: pts/0 (manual process, no service!)

Note: Using generic ProducerStrategy (not specialized)
```

### Feeder 121 (192.168.3.121) - SHORT MEANREV (Mean Reversion)
```bash
PID: 1026
Uptime: Since Jan 11, 2017
CPU Hours: 45,111 hours
Memory: 10.8 GB

Exact Command:
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ArtiSkyTrader_Feeder_SHORT_121_MEANREV \
  --config config_freqai_LightGBM.json \
  --freqaimodel LightGBMClassifier_Optuna \
  --logfile=dryrun.log

Working Directory: /home/freqai/freqtrade
Python: /home/freqai/freqtrade/.env/bin/python3.10
Terminal: pts/0 (manual process, no service!)
```

### Feeder 124 (192.168.3.124) - SHORT MTF (Multi-Timeframe)
```bash
PID: 23622
Uptime: Since Jan 11, 2018
CPU Hours: 43,893 hours
Memory: 13.5 GB

Exact Command:
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ArtiSkyTrader_Feeder_SHORT_124_MTF \
  --config config_feeder_short_124.json \
  --freqaimodel LightGBMClassifier_SHORT \
  --logfile=dryrun.log

Working Directory: /home/freqai/freqtrade
Python: /home/freqai/freqtrade/.env/bin/python3.10
Terminal: pts/0 (manual process, no service!)
```

---

## FEEDER SPECIALIZATIONS DISCOVERED

### LONG Feeders (3 running):
```
Feeder 74: MOMENTUM-based signals
  - Strategy: ArtiSkyTrader_Feeder_LONG_74_MOMENTUM
  - Focus: Momentum indicators
  - Model: LightGBMClassifier_Optuna

Feeder 79: VOLATILITY-based signals
  - Strategy: ArtiSkyTrader_Feeder_LONG_79_VOLATILITY
  - Focus: Volatility patterns
  - Model: LightGBMClassifier_Optuna

Feeders 73, 75, 108: NOT RUNNING
  - Need to determine their specializations
  - Likely: TREND, REVERSAL, BREAKOUT or similar
```

### SHORT Feeders (3 running):
```
Feeder 120: Generic Producer
  - Strategy: ProducerStrategy (generic)
  - Model: LightGBMClassifier_SHORT
  - Note: NOT specialized like others

Feeder 121: MEAN REVERSION-based signals
  - Strategy: ArtiSkyTrader_Feeder_SHORT_121_MEANREV
  - Focus: Mean reversion patterns
  - Model: LightGBMClassifier_Optuna

Feeder 124: MULTI-TIMEFRAME-based signals
  - Strategy: ArtiSkyTrader_Feeder_SHORT_124_MTF
  - Focus: Multiple timeframe analysis
  - Model: LightGBMClassifier_SHORT

Feeders 122, 123: NOT RUNNING
  - Need to determine their specializations
```

---

## SYSTEMD SERVICE TEMPLATES

### Template for Running Feeders (Example: Feeder 74):
```ini
[Unit]
Description=FreqTrade Feeder 74 (LONG-MOMENTUM) - LightGBM Signal Generator
After=network.target
Documentation=CORRECT_10_FEEDER_ARCHITECTURE.md

[Service]
Type=simple
User=freqai
Group=freqai
WorkingDirectory=/home/freqai/freqtrade
Environment="PATH=/home/freqai/freqtrade/.env/bin:/usr/local/bin:/usr/bin:/bin"

ExecStart=/home/freqai/freqtrade/.env/bin/freqtrade trade \
    --dry-run \
    --strategy ArtiSkyTrader_Feeder_LONG_74_MOMENTUM \
    --config config_freqai_LightGBM_LONG_74.json \
    --freqaimodel LightGBMClassifier_Optuna \
    --logfile feeder74.log

Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

# Resource limits (based on current usage)
MemoryMax=16G
CPUQuota=200%

[Install]
WantedBy=multi-user.target
```

---

## ALL 10 FEEDER SERVICE DEFINITIONS

### LONG Feeders:

**1. Feeder 73 (192.168.3.73) - TBD**
```
Status: Stopped
Service Name: freqtrade-feeder73.service
Config: Need to determine
Strategy: Need to determine
Model: TBD
Action: Investigate configs, create service, start
```

**2. Feeder 74 (192.168.3.74) - MOMENTUM** ✅
```
Status: Running (11+ days)
Service Name: freqtrade-feeder74.service
Config: config_freqai_LightGBM_LONG_74.json
Strategy: ArtiSkyTrader_Feeder_LONG_74_MOMENTUM
Model: LightGBMClassifier_Optuna
Action: Create service, migrate from manual process
```

**3. Feeder 75 (192.168.3.75) - TBD**
```
Status: Stopped
Service Name: freqtrade-feeder75.service
Config: Need to determine
Strategy: Need to determine
Model: TBD
Action: Investigate configs, create service, start
```

**4. Feeder 79 (192.168.3.79) - VOLATILITY** ✅
```
Status: Running (11+ days)
Service Name: freqtrade-feeder79.service
Config: config_freqai_LightGBM_LONG_79.json
Strategy: ArtiSkyTrader_Feeder_LONG_79_VOLATILITY
Model: LightGBMClassifier_Optuna
Action: Create service, migrate from manual process
```

**5. Feeder 108 (192.168.3.108) - TBD**
```
Status: Stopped
Service Name: freqtrade-feeder108.service
Config: Need to determine
Strategy: Need to determine
Model: TBD
Action: Investigate configs, create service, start
```

### SHORT Feeders:

**6. Feeder 120 (192.168.3.120) - PRODUCER** ✅
```
Status: Running (11+ days)
Service Name: freqtrade-feeder120.service
Config: config_freqai_LightGBM.json
Strategy: ProducerStrategy
Model: LightGBMClassifier_SHORT
Memory: 49 GB (monitor this - very high!)
Action: Create service, migrate from manual process
```

**7. Feeder 121 (192.168.3.121) - MEANREV** ✅
```
Status: Running (11+ days)
Service Name: freqtrade-feeder121.service
Config: config_freqai_LightGBM.json
Strategy: ArtiSkyTrader_Feeder_SHORT_121_MEANREV
Model: LightGBMClassifier_Optuna
Action: Create service, migrate from manual process
```

**8. Feeder 122 (192.168.3.122) - TBD**
```
Status: Stopped
Service Name: freqtrade-feeder122.service
Config: Need to determine
Strategy: Need to determine
Model: TBD
Action: Investigate configs, create service, start
```

**9. Feeder 123 (192.168.3.123) - TBD**
```
Status: Stopped
Service Name: freqtrade-feeder123.service
Config: Need to determine
Strategy: Need to determine
Model: TBD
Action: Investigate configs, create service, start
```

**10. Feeder 124 (192.168.3.124) - MTF** ✅
```
Status: Running (11+ days)
Service Name: freqtrade-feeder124.service
Config: config_feeder_short_124.json
Strategy: ArtiSkyTrader_Feeder_SHORT_124_MTF
Model: LightGBMClassifier_SHORT
Action: Create service, migrate from manual process
```

---

## CRITICAL RISKS IDENTIFIED

### Current Setup Vulnerabilities:

**1. No Service Management:**
```
All 5 running feeders are manual processes
- Started in terminal sessions (pts/0)
- Will die if terminal closes
- Will die on server reboot
- No auto-restart on crash
```

**2. Long Runtimes Without Protection:**
```
Running since: Jan 11 (11+ days!)
No systemd supervision
High memory usage (10-49 GB per feeder)
Total: ~98 GB across 5 feeders
```

**3. Missing Feeders:**
```
5 feeders NOT running
System operating at 50% capacity
Missing signal diversity
Ensemble voting degraded (3/5 vs 5/5 possible)
```

---

## MIGRATION STRATEGY (FOR NEXT SESSION)

### Phase 1: Create Services for Running Feeders (Priority!)

**For feeders 74, 79, 120, 121, 124:**
1. Create systemd service file with EXACT current command
2. Deploy to `/etc/systemd/system/`
3. Test service start (dry-run)
4. Gracefully stop manual process
5. Start systemd service
6. Verify operation continues
7. Enable for boot-start

**Why Priority:**
These have been running 11+ days, can't afford to lose them!

### Phase 2: Investigate Stopped Feeders

**For feeders 73, 75, 108, 122, 123:**
1. Determine intended strategy (check logs, configs)
2. Create appropriate service files
3. Deploy and enable
4. Start services
5. Monitor for errors

### Phase 3: System-Wide Verification

1. All 10 services running
2. All enabled for boot-start
3. Aggregator receiving from all 10
4. BOT1/BOT2 receiving ensemble signals
5. Signal quality improved (10/10 vs 5/10)

---

## FEEDER SPECIALIZATION PATTERNS

### LONG Feeders (Pattern Discovered):
```
Feeder 74: MOMENTUM (confirmed)
Feeder 79: VOLATILITY (confirmed)
Feeder 73: Likely TREND or BREAKOUT
Feeder 75: Likely REVERSAL or VOLUME
Feeder 108: Likely PATTERN or MULTI-INDICATOR
```

### SHORT Feeders (Pattern Discovered):
```
Feeder 120: PRODUCER (generic, oldest implementation)
Feeder 121: MEANREV (mean reversion)
Feeder 124: MTF (multi-timeframe)
Feeder 122: Likely MOMENTUM or VOLATILITY
Feeder 123: Likely TREND or BREAKOUT
```

---

## DEPLOYMENT COMMANDS (FOR NEXT SESSION)

### Example: Create Service for Feeder 74

**1. Create service file locally:**
```bash
cat > freqtrade-feeder74.service << 'EOF'
[Unit]
Description=FreqTrade Feeder 74 (LONG-MOMENTUM)
After=network.target

[Service]
Type=simple
User=freqai
WorkingDirectory=/home/freqai/freqtrade
ExecStart=/home/freqai/freqtrade/.env/bin/freqtrade trade --dry-run --strategy ArtiSkyTrader_Feeder_LONG_74_MOMENTUM --config config_freqai_LightGBM_LONG_74.json --freqaimodel LightGBMClassifier_Optuna --logfile feeder74.log
Restart=always
RestartSec=10
MemoryMax=16G

[Install]
WantedBy=multi-user.target
EOF
```

**2. Deploy to feeder:**
```bash
sshpass -p 'NewHopes@168' scp freqtrade-feeder74.service \
  freqai@192.168.3.74:/tmp/

sshpass -p 'NewHopes@168' ssh freqai@192.168.3.74 \
  "echo 'NewHopes@168' | sudo -S mv /tmp/freqtrade-feeder74.service /etc/systemd/system/ && \
   sudo systemctl daemon-reload"
```

**3. Enable service:**
```bash
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.74 \
  "echo 'NewHopes@168' | sudo -S systemctl enable freqtrade-feeder74.service"
```

**4. Migrate (carefully!):**
```bash
# Test service start first (will fail if already running)
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.74 \
  "echo 'NewHopes@168' | sudo -S systemctl start freqtrade-feeder74.service"

# If successful, stop old process
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.74 \
  "kill -TERM 75827"  # Graceful stop

# Service auto-restarts, continuing operation
```

---

## MEMORY USAGE CONCERN

### Feeder 120 (192.168.3.120):
```
⚠️ WARNING: Using 49 GB memory!

This is 4x higher than other feeders (10-13 GB)

Possible Causes:
- Memory leak
- Model size too large  
- Data caching issue
- FreqAI model accumulation

Action Needed:
- Investigate FreqAI model files
- Check for memory leak
- May need restart to clear
- Monitor after service migration
```

---

## FILES NEEDED FOR EACH FEEDER

### Service Definition Files (10 total):
```
freqtrade-feeder73.service
freqtrade-feeder74.service  ← Has exact config from process
freqtrade-feeder75.service
freqtrade-feeder79.service  ← Has exact config from process
freqtrade-feeder108.service
freqtrade-feeder120.service ← Has exact config from process
freqtrade-feeder121.service ← Has exact config from process
freqtrade-feeder122.service
freqtrade-feeder123.service
freqtrade-feeder124.service ← Has exact config from process
```

---

## QUICK REFERENCE

### Running Feeder Details:

| Feeder | IP | Strategy | Config | Model | Memory |
|--------|-----|----------|--------|-------|--------|
| 74 | 192.168.3.74 | LONG_74_MOMENTUM | config_freqai_LightGBM_LONG_74.json | LightGBMClassifier_Optuna | 12.6 GB |
| 79 | 192.168.3.79 | LONG_79_VOLATILITY | config_freqai_LightGBM_LONG_79.json | LightGBMClassifier_Optuna | 12.8 GB |
| 120 | 192.168.3.120 | ProducerStrategy | config_freqai_LightGBM.json | LightGBMClassifier_SHORT | 49.1 GB ⚠️ |
| 121 | 192.168.3.121 | SHORT_121_MEANREV | config_freqai_LightGBM.json | LightGBMClassifier_Optuna | 10.8 GB |
| 124 | 192.168.3.124 | SHORT_124_MTF | config_feeder_short_124.json | LightGBMClassifier_SHORT | 13.5 GB |

### Stopped Feeder Configs Available:

| Feeder | IP | Config Files Found |
|--------|-----|-------------------|
| 73 | 192.168.3.73 | config_freqai.json, config_freqai_PyTorch.json |
| 75 | 192.168.3.75 | config_freqai.json, config_freqai_PyTorch.json |
| 108 | 192.168.3.108 | config_freqai.json, config.json |
| 122 | 192.168.3.122 | config_freqai_LightGBM.json, config_freqai_123.json, etc. |
| 123 | 192.168.3.123 | config_freqai_LightGBM.json, config_freqai_123.json, etc. |

---

## RECOMMENDATIONS FOR NEXT SESSION

### Critical Actions:

**1. URGENT: Protect Running Feeders**
```
Create systemd services for:
- Feeder 74 (LONG-MOMENTUM)
- Feeder 79 (LONG-VOLATILITY)
- Feeder 120 (SHORT-PRODUCER) - Also address 49GB memory!
- Feeder 121 (SHORT-MEANREV)
- Feeder 124 (SHORT-MTF)

Benefit: Protection from crashes, auto-restart, boot-start
Risk if not done: Any terminal disconnect kills 11+ days of uptime!
```

**2. HIGH: Investigate feeder 120 Memory**
```
49 GB is abnormally high (4x normal)
May indicate:
- Memory leak
- Model bloat
- Need investigation before service creation
```

**3. MEDIUM: Start Stopped Feeders**
```
Determine configs for:
- Feeders 73, 75, 108 (LONG)
- Feeders 122, 123 (SHORT)

Start as services
Bring system to 100% capacity (10/10 feeders)
```

---

## ESTIMATED TIME AND EFFORT

### Next Session Requirements:

**Time: 45-60 minutes**
- Create 5 service files (running): 15 min
- Deploy and test: 15 min
- Investigate stopped feeders: 15 min
- Create remaining 5 services: 10 min
- Final verification: 10 min

**Complexity: Medium**
- Straightforward for running feeders (copy configs)
- Some investigation needed for stopped feeders
- Careful migration to preserve 11+ day uptime

**Risk Level: Low-Medium**
- Running feeders: Low risk (just formalizing)  
- Stopped feeders: Medium (need correct configs)
- Can rollback to manual if needed

---

## SUCCESS CRITERIA

After service conversion complete:

✅ All 10 feeders have systemd services
✅ All 10 feeders running and sending signals
✅ All services enabled for boot-start
✅ Aggregator receiving from 10/10 feeders
✅ BOT1 getting 5/5 LONG ensemble (not 2/5 or 3/5)
✅ BOT2 getting 5/5 SHORT ensemble (not 2/5 or 3/5)
✅ No downtime during migration
✅ Memory issue on feeder 120 addressed

---

**END OF INVESTIGATION - READY FOR NEXT SESSION**
