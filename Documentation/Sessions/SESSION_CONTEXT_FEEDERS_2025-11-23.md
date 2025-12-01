# 10 Feeder Investigation & Service Conversion - Session Context
Date: 2025-11-23
Status: Investigation Complete, Ready for Service Deployment

---

## QUICK REFERENCE

### Credentials:
```
Feeders (All 10): freqai / NewHopes@168
BOT1: ubuntu / ArtiSky@7 (192.168.3.71)
BOT2: ubuntu / ArtiSky@7 (192.168.3.72)
BOT3: ubuntu / ArtiSky@7 (192.168.3.33)
```

### 10 Feeder IPs:
```
LONG Feeders (→ BOT1):
- 192.168.3.73  (Feeder #1 - Stopped)
- 192.168.3.74  (Feeder #2 - Running 11+ days - MOMENTUM) ✅
- 192.168.3.75  (Feeder #3 - Stopped)
- 192.168.3.79  (Feeder #4 - Running 11+ days - VOLATILITY) ✅
- 192.168.3.108 (Feeder #5 - Stopped)

SHORT Feeders (→ BOT2):
- 192.168.3.120 (Feeder #6 - Running 11+ days - PRODUCER) ✅
- 192.168.3.121 (Feeder #7 - Running 11+ days - MEANREV) ✅
- 192.168.3.122 (Feeder #8 - Stopped)
- 192.168.3.123 (Feeder #9 - Stopped)
- 192.168.3.124 (Feeder #10 - Running 11+ days - MTF) ✅
```

### Key Servers:
```
Aggregator: 192.168.3.80 (Ensemble voting & signal aggregation)
InfluxDB: 192.168.3.6 (Shared learning database)
BOT1: 192.168.3.71 (LONG executor)
BOT2: 192.168.3.72 (SHORT executor)
BOT3: 192.168.3.33 (Meta-learner)
```

---

## SESSION SUMMARY

### What Was Accomplished:

**1. BOT1 & BOT2 Status Verified** ✅
- Both running stable (10+ hours before restart)
- Using 3-tier intelligent exit system
- Position→account conversion working
- Verified cross-server parameter sharing via InfluxDB

**2. Issues Fixed** ✅
- ZEC blacklist applied (BOT3 restarted 09:40)
- Optuna parameters reloaded (BOT1 09:45, BOT2 09:46)
- Mirror service resumed (PID 659498)
- All services operational

**3. Architecture Documented** ✅
- Parameter sharing via InfluxDB explained
- Position→account leverage conversion verified
- Service restart procedures documented
- 10-feeder architecture with Aggregator mapped

**4. 10 Feeder Investigation** ✅
- All 10 feeders accessed
- Running processes analyzed
- Exact configurations extracted
- Service conversion plan created

---

## 10 FEEDER CURRENT STATE

### Running Feeders: 5 out of 10

**All running as MANUAL PROCESSES for 11+ DAYS!**

| Feeder | IP | Strategy | Config | Memory | Status |
|--------|-----|----------|--------|--------|--------|
| 74 | .74 | LONG_74_MOMENTUM | config_freqai_LightGBM_LONG_74.json | 12.6 GB | ✅ Running |
| 79 | .79 | LONG_79_VOLATILITY | config_freqai_LightGBM_LONG_79.json | 12.8 GB | ✅ Running |
| 120 | .120 | ProducerStrategy | config_freqai_LightGBM.json | 49 GB ⚠️ | ✅ Running |
| 121 | .121 | SHORT_121_MEANREV | config_freqai_LightGBM.json | 10.8 GB | ✅ Running |
| 124 | .124 | SHORT_124_MTF | config_feeder_short_124.json | 13.5 GB | ✅ Running |

### Stopped Feeders: 5 out of 10

| Feeder | IP | Config Found | Action Needed |
|--------|-----|--------------|---------------|
| 73 | .73 | config_freqai.json | Determine strategy, create service, start |
| 75 | .75 | config_freqai.json | Determine strategy, create service, start |
| 108 | .108 | config_freqai.json | Determine strategy, create service, start |
| 122 | .122 | config_freqai_LightGBM.json | Determine strategy, create service, start |
| 123 | .123 | config_freqai_LightGBM.json | Determine strategy, create service, start |

---

## EXACT PROCESS COMMANDS (FROM RUNNING FEEDERS)

### Feeder 74:
```bash
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ArtiSkyTrader_Feeder_LONG_74_MOMENTUM \
  --config config_freqai_LightGBM_LONG_74.json \
  --freqaimodel LightGBMClassifier_Optuna \
  --logfile=dryrun.log
```

### Feeder 79:
```bash
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ArtiSkyTrader_Feeder_LONG_79_VOLATILITY \
  --config config_freqai_LightGBM_LONG_79.json \
  --freqaimodel LightGBMClassifier_Optuna \
  --logfile=dryrun.log
```

### Feeder 120:
```bash
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ProducerStrategy \
  --config config_freqai_LightGBM.json \
  --freqaimodel LightGBMClassifier_SHORT \
  --logfile=dryrun.log
```

### Feeder 121:
```bash
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ArtiSkyTrader_Feeder_SHORT_121_MEANREV \
  --config config_freqai_LightGBM.json \
  --freqaimodel LightGBMClassifier_Optuna \
  --logfile=dryrun.log
```

### Feeder 124:
```bash
/home/freqai/freqtrade/.env/bin/freqtrade trade \
  --dry-run \
  --strategy ArtiSkyTrader_Feeder_SHORT_124_MTF \
  --config config_feeder_short_124.json \
  --freqaimodel LightGBMClassifier_SHORT \
  --logfile=dryrun.log
```

**Common Pattern:**
- Working Directory: `/home/freqai/freqtrade`
- Python: `/home/freqai/freqtrade/.env/bin/python3.10`
- Logfile: `dryrun.log` (same for all)
- Mode: `--dry-run`

---

## SYSTEMD SERVICE TEMPLATE

### Generic Template (Adapt for Each Feeder):

```ini
[Unit]
Description=FreqTrade Feeder {NUMBER} ({DIRECTION}-{SPECIALIZATION})
After=network.target

[Service]
Type=simple
User=freqai
Group=freqai
WorkingDirectory=/home/freqai/freqtrade
Environment="PATH=/home/freqai/freqtrade/.env/bin:/usr/bin"

ExecStart=/home/freqai/freqtrade/.env/bin/freqtrade trade \
    --dry-run \
    --strategy {STRATEGY_NAME} \
    --config {CONFIG_FILE} \
    --freqaimodel {MODEL_NAME} \
    --logfile feeder{NUMBER}.log

Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
MemoryMax=16G

[Install]
WantedBy=multi-user.target
```

### Service File Names:
```
freqtrade-feeder73.service
freqtrade-feeder74.service
freqtrade-feeder75.service
freqtrade-feeder79.service
freqtrade-feeder108.service
freqtrade-feeder120.service
freqtrade-feeder121.service
freqtrade-feeder122.service
freqtrade-feeder123.service
freqtrade-feeder124.service
```

---

## MIGRATION PROCEDURE (FOR NEXT SESSION)

### Phase 1: Create Services for Running Feeders (PRIORITY!)

**For each running feeder (74, 79, 120, 121, 124):**

```bash
# 1. Create service file
cat > freqtrade-feeder{XX}.service << 'EOF'
[Unit]
Description=FreqTrade Feeder {XX}
...
EOF

# 2. Deploy to feeder
sshpass -p 'NewHopes@168' scp freqtrade-feeder{XX}.service \
  freqai@192.168.3.{XX}:/tmp/

# 3. Install
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.{XX} \
  "echo 'NewHopes@168' | sudo -S mv /tmp/freqtrade-feeder{XX}.service /etc/systemd/system/ && \
   sudo systemctl daemon-reload && \
   sudo systemctl enable freqtrade-feeder{XX}.service"

# 4. Test (optional - will fail if already running)
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.{XX} \
  "echo 'NewHopes@168' | sudo -S systemctl start freqtrade-feeder{XX}.service"

# 5. If test successful, migrate:
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.{XX} \
  "kill -TERM {PID}"  # Stop old process gracefully

# Service will auto-start and continue operation
```

### Phase 2: Start Stopped Feeders

**For stopped feeders (73, 75, 108, 122, 123):**

```bash
# 1. Investigate to find correct config/strategy
# 2. Create service file
# 3. Deploy and enable
# 4. Start service
# 5. Monitor for errors
```

---

## CRITICAL ISSUES IDENTIFIED

### 🚨 Issue #1: No Systemd Services
```
Impact: ALL 5 running feeders vulnerable
Risk: Terminal disconnect = process death = 11+ days lost
Action: URGENT - Create services ASAP
Priority: HIGHEST
```

### 🔴 Issue #2: Feeder 120 Memory
```
Current: 49.1 GB (4x normal!)
Normal: 10-13 GB
Impact: Potential memory leak or model bloat
Action: Investigate before service creation
Priority: HIGH
```

### ⚠️ Issue #3: 50% Feeder Capacity
```
Running: 5/10 feeders (50%)
Impact: Degraded ensemble quality
Missing: TREND, REVERSAL, BREAKOUT specializations?
Action: Start stopped feeders
Priority: MEDIUM
```

---

## DOCUMENTS CREATED THIS SESSION

### Technical Documentation:
1. **BOT1_BOT2_PARAMETER_SHARING_ARCHITECTURE.md**
   - Cross-server parameter flow via InfluxDB
   - Network topology

2. **POSITION_TO_ACCOUNT_CONVERSION_EXPLAINED.md**
   - Leverage multiplication: Account = Position × Leverage
   - 7% risk cap implementation
   - Real trade examples

3. **CONFIG_CHANGE_RESTART_REMINDER.md**
   - Why service restarts are required
   - Step-by-step procedures
   - Learned from ZEC and Optuna issues

4. **CORRECT_10_FEEDER_ARCHITECTURE.md**
   - Complete 10-feeder network map
   - Aggregator role (192.168.3.80)
   - REST API signal flow

5. **FEEDER_SERVICE_CONVERSION_PLAN.md**
   - Exact running process configurations
   - Systemd service templates
   - Migration procedures
   - Risk assessment

### Investigation Tools:
6. **check_all_feeders.sh** - Reusable investigation script
7. **feeder_investigation_results.md** - Investigation findings
8. **feeder_investigation_output.txt** - Raw output

---

## NEXT SESSION OBJECTIVES

### Primary Goal:
**Convert all 10 feeders to systemd services**

### Tasks:

**URGENT (Phase 1):**
1. Create systemd services for 5 running feeders
2. Deploy services to `/etc/systemd/system/`
3. Enable for boot-start
4. Migrate from manual to service (carefully!)
5. Verify no downtime

**HIGH (Phase 2):**
6. Investigate feeder 120 memory (49 GB!)
7. Determine why so high
8. Address before service creation

**MEDIUM (Phase 3):**
9. Investigate configs for stopped feeders (73, 75, 108, 122, 123)
10. Determine correct strategies
11. Create and start their services
12. Bring system to 100% capacity (10/10)

### Success Criteria:
- ✅ All 10 feeders have systemd services
- ✅ All 10 feeders running
- ✅ All enabled for boot-start
- ✅ No downtime during migration
- ✅ Memory issue addressed
- ✅ Aggregator receiving 10/10 signals

---

## SYSTEM ARCHITECTURE (CORRECTED)

### Signal Flow:
```
10 Feeders (LightGBM ML)
    ↓ REST API POST /forcebuy
Aggregator (192.168.3.80)
    ↓ Ensemble voting (3/5 minimum)
    ↓ REST API POST /forcebuy with ensemble tags
BOT1 (LONG) & BOT2 (SHORT)
    ↓ Execute with Optuna/Stateless exits
    ↓ Log to InfluxDB
BOT3 (Meta-learner)
    ↓ Analyze quality, optimize parameters
    ↓ Write to InfluxDB OPTUNA_PARAMS
All Bots read parameters from InfluxDB
```

### NOT Using:
```
❌ Producer/Consumer WebSockets
❌ Direct feeder→BOT1/BOT2 connection
✅ Using: Aggregator→BOT1/BOT2 via REST
```

---

## RUNNING FEEDER DETAILS

### Feeder 74 (LONG-MOMENTUM):
```
IP: 192.168.3.74
PID: 75827
Strategy: ArtiSkyTrader_Feeder_LONG_74_MOMENTUM
Config: config_freqai_LightGBM_LONG_74.json
Model: LightGBMClassifier_Optuna
Memory: 12.6 GB
CPU Hours: 45,628 (11+ days)
Started: Jan 11, 2018
Status: Running in pts/0 terminal ⚠️ NO SERVICE
```

### Feeder 79 (LONG-VOLATILITY):
```
IP: 192.168.3.79
PID: 36437
Strategy: ArtiSkyTrader_Feeder_LONG_79_VOLATILITY
Config: config_freqai_LightGBM_LONG_79.json
Model: LightGBMClassifier_Optuna
Memory: 12.8 GB
CPU Hours: 43,901
Started: Jan 11, 2018
Status: Running in pts/0 terminal ⚠️ NO SERVICE
```

### Feeder 120 (SHORT-PRODUCER):
```
IP: 192.168.3.120
PID: 125658
Strategy: ProducerStrategy (generic)
Config: config_freqai_LightGBM.json
Model: LightGBMClassifier_SHORT
Memory: 49.1 GB ⚠️ VERY HIGH!
CPU Hours: 58,200 (highest!)
Started: Jan 11, 2019
Status: Running in pts/0 terminal ⚠️ NO SERVICE
Issue: Memory 4x normal - needs investigation
```

### Feeder 121 (SHORT-MEANREV):
```
IP: 192.168.3.121
PID: 1026
Strategy: ArtiSkyTrader_Feeder_SHORT_121_MEANREV
Config: config_freqai_LightGBM.json
Model: LightGBMClassifier_Optuna
Memory: 10.8 GB
CPU Hours: 45,111
Started: Jan 11, 2017
Status: Running in pts/0 terminal ⚠️ NO SERVICE
```

### Feeder 124 (SHORT-MTF):
```
IP: 192.168.3.124
PID: 23622
Strategy: ArtiSkyTrader_Feeder_SHORT_124_MTF
Config: config_feeder_short_124.json
Model: LightGBMClassifier_SHORT
Memory: 13.5 GB
CPU Hours: 43,893
Started: Jan 11, 2018
Status: Running in pts/0 terminal ⚠️ NO SERVICE
```

---

## CRITICAL RISKS

### 🚨 All Running Feeders Vulnerable:
```
Current State:
- Manual processes in terminal (pts/0)
- No systemd protection
- 11+ days uptime (since Jan 11)
- Will die on: terminal disconnect, server reboot, crash

Risk Level: CRITICAL
Impact: Loss of 11+ days continuous operation
Solution: Convert to systemd services URGENTLY
```

### 🔴 Feeder 120 Memory Leak:
```
Memory: 49.1 GB (4x normal)
Other feeders: 10-13 GB average
Concern: Memory leak or model accumulation
Action: Investigate FreqAI model storage, cache
May need: Restart to clear, then service creation
```

### ⚠️ 50% System Capacity:
```
Active: 5/10 feeders
Missing: 5 feeder specializations
Impact: Lower ensemble quality, fewer signals
Example: "3/5 ensemble" instead of "5/5 ensemble"
Solution: Start stopped feeders with services
```

---

## WHY FEEDER 74 CONFIG WAS "DISABLED" IN BOT1

### Explanation (Now Clear):

**BOT1 config shows:**
```json
"external_message_consumer": {
    "enabled": false
}
```

**This is CORRECT because:**
- You're NOT using FreqTrade producer/consumer websockets
- You're using **Aggregator (192.168.3.80)** architecture
- Feeders → Aggregator → BOT1/BOT2 (via REST forcebuy)
- Much cleaner than websocket mesh

**The entry tags prove this:**
```
"long ensemble 4/5 conf=0.85 from=74"
     ↑
Created by Aggregator, not BOT1!
```

---

## SESSION ACTIONS TAKEN

### Services Restarted:
```
09:40 - BOT3 restarted (applied ZEC blacklist)
09:45 - BOT1 restarted (loaded Optuna parameters including HBAR)
09:46 - BOT2 restarted (loaded Optuna parameters including HBAR)
10:00 - Mirror service resumed (PID 659498)
```

### Issues Resolved:
```
✅ ZEC blacklist now active (no new ZEC entries)
✅ HBAR now uses Optuna (+7.71% target vs +4.49% fallback)
✅ All 37+ Optuna pairs loaded on BOT1/BOT2
✅ Position→account conversion verified
✅ Parameter sharing understood and documented
✅ 10-feeder architecture mapped
```

---

## CURRENT SYSTEM STATUS

### All BOT Services: ✅ OPERATIONAL

**BOT1 (192.168.3.71):**
- Running since: 09:45:38
- PID: 87830
- Optuna: 37+ pairs
- Stateless: Active
- Direction: LONG

**BOT2 (192.168.3.72):**
- Running since: 09:45:59
- PID: 93283
- Optuna: 37+ pairs
- Stateless: Active
- Direction: SHORT

**BOT3 (192.168.3.33):**
- Running since: 09:40:35
- Trading: 59 pairs (ZEC blacklisted)
- Optuna: Active
- Direction: BOTH

**Optuna Service:**
- Status: Running 3+ hours
- Pairs: 37+ optimized
- Continuous optimization active

**Mirror Service:**
- PID: 659498
- State: S (running)
- Targets: .38, .66, .68

**InfluxDB (192.168.3.6):**
- BOT1_trades: 31,788
- BOT2_trades: 13,497
- BOT33_trades: 3,642
- OPTUNA_PARAMS: 37+ pairs

### Feeder Status: ⚠️ NEEDS ATTENTION

**Running:** 5/10 (as manual processes, 11+ days)
**Systemd Services:** 0/10 ❌
**Next Priority:** Convert to services!

---

## FEEDER SPECIALIZATION STRATEGY

### Design Intent (Discovered):

**Diversity Through Specialization:**
- Each feeder focuses on different market aspect
- MOMENTUM: Catches trending moves
- VOLATILITY: Adapts to price swings
- MEANREV: Mean reversion opportunities
- MTF: Multi-timeframe confluence
- Others: Likely TREND, BREAKOUT, VOLUME, PATTERN, etc.

**Ensemble Benefit:**
```
5 different perspectives on same market
Vote aggregation filters noise
High confidence when multiple agree
Low confidence when opinions split
Better than single model approach
```

---

## QUICK START COMMANDS (NEXT SESSION)

### Check Feeder Status:
```bash
./check_all_feeders.sh
```

### Test Single Feeder (Example 74):
```bash
# SSH to feeder
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.74

# Check process
ps aux | grep freqtrade | grep -v grep
```

### Deploy Service (Example 74):
```bash
# Create service file (on local)
cat > feeder74.service << 'EOF'
[contents here]
EOF

# Deploy
sshpass -p 'NewHopes@168' scp feeder74.service freqai@192.168.3.74:/tmp/
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.74 \
  "echo 'NewHopes@168' | sudo -S mv /tmp/feeder74.service /etc/systemd/system/freqtrade-feeder74.service && \
   sudo systemctl daemon-reload && \
   sudo systemctl enable freqtrade-feeder74.service"
```

---

## LESSONS LEARNED THIS SESSION

### 1. Config Changes Require Restart:
```
ZEC blacklist: Added to config but not active
Optuna params: Generated but not loaded
Solution: ALWAYS restart services after config/data changes
```

### 2. Position vs Account Values:
```
InfluxDB stores: Position-level (e.g., -3.86%)
BOT1/BOT2 convert: Position × Leverage = Account
Formula working correctly in all bots
```

### 3. Architecture Complexity:
```
Initially thought: Producer/Consumer websockets
Actually using: Aggregator + REST API forcebuy
Much cleaner and more reliable
```

### 4. Long-Running Processes Need Protection:
```
5 feeders running 11+ days in terminals
One disconnect = everything lost
Systemd services = protection + management
```

---

## FILES FOR NEXT SESSION

### Investigation Tools:
- `check_all_feeders.sh` - Check all feeder status
- `feeder_investigation_output.txt` - Current status snapshot

### Service Templates:
- See FEEDER_SERVICE_CONVERSION_PLAN.md for templates
- Customize for each feeder

### Documentation:
- All documents in `/home/ubuntu/freqtrade/`
- Ready for next session reference

---

## NEXT SESSION CHECKLIST

**Pre-Session:**
- [ ] Review FEEDER_SERVICE_CONVERSION_PLAN.md
- [ ] Have credentials ready (freqai / NewHopes@168)
- [ ] Understand migration procedure

**During Session:**
- [ ] Create 5 service files (running feeders)
- [ ] Deploy to feeders
- [ ] Enable services
- [ ] Migrate from manual processes
- [ ] Investigate feeder 120 memory
- [ ] Find configs for stopped feeders
- [ ] Create 5 more service files
- [ ] Start all 10 services
- [ ] Verify complete system

**Post-Session:**
- [ ] All 10 feeders as services
- [ ] All enabled for boot-start
- [ ] All running and sending signals
  [ ] Documentation updated

---

**END OF SESSION CONTEXT - READY FOR FEEDER SERVICE DEPLOYMENT**
