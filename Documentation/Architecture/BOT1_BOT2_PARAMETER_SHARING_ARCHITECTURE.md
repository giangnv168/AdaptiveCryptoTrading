# BOT1 & BOT2 Parameter Sharing Architecture
**How BOT1/BOT2 Get Parameters from BOT3's Optuna & Stateless Manager**

Date: 2025-11-23
Status: Technical Deep Dive

---

## THE PROBLEM

BOT1, BOT2, and BOT3 run on **different physical servers**:
- **BOT1**: 192.168.3.71 (LONG signals)
- **BOT2**: 192.168.3.72 (SHORT signals)
- **BOT3**: 192.168.3.33 (Executor)

**Question**: How do BOT1 and BOT2 get optimized take-profit/cut-loss parameters from processes running on BOT3?

**Answer**: Through **InfluxDB as a central shared database**.

---

## THE SOLUTION: INFLUXDB AS DATA HUB

```
┌─────────────────────────────────────────────────────────────┐
│                    NETWORK TOPOLOGY                          │
└─────────────────────────────────────────────────────────────┘

    192.168.3.71          192.168.3.72          192.168.3.33
    ════════════          ════════════          ════════════
       BOT1                  BOT2                  BOT3
    (LONG signals)        (SHORT signals)        (Executor)
         │                     │                      │
         │                     │                      │
         └─────────────────────┼──────────────────────┘
                               │
                               │ ALL connect to
                               │
                               ▼
                        192.168.3.6
                        ════════════
                         INFLUXDB
                    (Central Database)
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
   BOT1_trades           BOT2_trades            BOT33_trades
   (32k signals)         (14k signals)          (3.6k trades)
        │                      │                      │
        │                      │                      │
        └──────────────────────┼──────────────────────┘
                               │
                        OPTUNA_PARAMS
                    (Optimized Parameters)
```

---

## DATA FLOW STEP-BY-STEP

### STEP 1: BOT3 Generates Parameters

**On Server 192.168.3.33 (BOT3)**

There are TWO processes running that generate parameters:

#### Process 1: Optuna Optimizer (Per-Pair)
```python
# File: /home/ubuntu/freqtrade/user_data/services/optuna_optimizer_service.py
# Running on: 192.168.3.33 (BOT3 server)

class OptunaOptimizer:
    def __init__(self):
        # Connect to InfluxDB at 192.168.3.6
        self.influx_client = InfluxDBClient(
            url="http://192.168.3.6:8086",
            token="your_token",
            org="freqtrade"
        )
        
    def optimize_pair(self, pair):
        # Runs simulation on historical candles
        # Tests different stop/target combinations
        # Example: For BTC/USDT
        
        best_params = {
            'pair': 'BTC/USDT',
            'stop_loss_pct': -1.50,      # Position level
            'take_profit_pct': 2.00,      # Position level
            'win_rate': 0.68,
            'profit_factor': 1.92,
            'total_trades': 892
        }
        
        # WRITES to InfluxDB OPTUNA_PARAMS bucket
        self.write_to_influxdb(best_params)
```

**What it writes to InfluxDB:**
```
Bucket: OPTUNA_PARAMS
Measurement: parameters
Tags: pair=BTC/USDT
Fields:
  - stop_loss_pct: -1.50
  - take_profit_pct: 2.00
  - win_rate: 0.68
  - profit_factor: 1.92
  - total_trades: 892
  - timestamp: 2025-11-23T02:15:00Z
```

#### Process 2: Stateless Manager (Leverage-Aware)
```python
# File: /home/ubuntu/freqtrade/user_data/controllers/influxdb_parameter_manager.py
# Running on: ALL servers (BOT1, BOT2, BOT3)

class InfluxDBParameterManager:
    def __init__(self):
        # Connect to InfluxDB at 192.168.3.6
        self.influx_client = InfluxDBClient(
            url="http://192.168.3.6:8086",
            token="your_token",
            org="freqtrade"
        )
        self.cache_duration = 30  # seconds
        
    def get_leverage_params(self, leverage):
        # READS from InfluxDB BOT33_trades bucket
        # Analyzes 3,642 actual trades
        # Groups by leverage bins
        
        # Example for 3.0x leverage:
        query = f'''
        from(bucket: "BOT33_trades")
        |> range(start: -30d)
        |> filter(fn: (r) => r.leverage >= 2.5 and r.leverage < 3.5)
        |> mean()
        '''
        
        results = self.query_influxdb(query)
        
        # Calculates optimal parameters
        params = {
            'leverage_bin': '2.5-3.5x',
            'stop_loss_pct': -1.48,      # Position level
            'take_profit_pct': 1.00,      # Position level
            'avg_win': 0.88,
            'avg_loss': -1.46,
            'win_rate': 0.61,
            'total_trades': 3548
        }
        
        # Cached locally for 30 seconds
        return params
```

**What it reads from InfluxDB:**
```
Bucket: BOT33_trades
Measurement: closed_trades
Filters: leverage between 2.5 and 3.5
Analyzes: 3,548 trades in this leverage range
Calculates: Average stop/target that worked historically
```

---

### STEP 2: BOT1 & BOT2 Read Parameters

**On Servers 192.168.3.71 (BOT1) and 192.168.3.72 (BOT2)**

Both BOT1 and BOT2 have the **same code** in their strategies that reads from InfluxDB:

```python
# File: /home/ubuntu/freqtrade/user_data/strategies/ArtiSkyTrader.py
# Running on: 192.168.3.71 (BOT1) and 192.168.3.72 (BOT2)

class ArtiSkyTrader(IStrategy):
    def __init__(self, config):
        super().__init__(config)
        
        # 1. Initialize connection to InfluxDB
        self.influx_reader = UltimateSignalQualityReaderV3(
            influx_url="http://192.168.3.6:8086",
            influx_token="your_token",
            influx_org="freqtrade",
            influx_bucket="BOT33_trades"
        )
        
        # 2. Initialize Stateless Manager
        self.stateless_params = InfluxDBParameterManager(
            influx_reader=self.influx_reader,
            cache_duration_seconds=30  # Refresh every 30s
        )
        
        # 3. Initialize Optuna loader
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
        This method runs on EVERY candle for EVERY open trade.
        It queries InfluxDB to get the latest parameters.
        """
        
        leverage = trade.leverage or 1.0
        
        # ═══════════════════════════════════════════════════════
        # TIER 1: Load Optuna Parameters (if available)
        # ═══════════════════════════════════════════════════════
        
        if pair in self.optuna_params:
            # This pair has Optuna-optimized parameters
            # These parameters are stored in InfluxDB OPTUNA_PARAMS bucket
            # on 192.168.3.6
            
            params = self.optuna_params[pair]
            stop_position = params['stop_loss_pct']      # e.g., -1.50%
            target_position = params['take_profit_pct']   # e.g., 2.00%
            
            # Convert to account level
            stop_account = stop_position * leverage       # -4.50% at 3x
            target_account = target_position * leverage   # 6.00% at 3x
            
            logger.info(f"🎯 OPTUNA params for {pair}:")
            logger.info(f"   Position: {target_position}%")
            logger.info(f"   Account: {target_account}% @ {leverage}x")
            logger.info(f"   From {params['total_trades']} trades")
            
            # Check if target hit
            if current_profit >= target_account:
                return 'roi_optuna'
        
        # ═══════════════════════════════════════════════════════
        # TIER 2: Load Stateless Manager Parameters
        # ═══════════════════════════════════════════════════════
        
        else:
            # No Optuna params yet, use Stateless Manager
            # This QUERIES InfluxDB BOT33_trades bucket in real-time
            # The query happens through the network to 192.168.3.6
            
            leverage_params = self.stateless_params.get_leverage_params(
                leverage=leverage
            )
            
            if leverage_params:
                stop_position = leverage_params['stop_loss_pct']
                target_position = leverage_params['take_profit_pct']
                
                # Convert to account level
                stop_account = stop_position * leverage
                target_account = target_position * leverage
                
                logger.info(f"💎 Leverage-aware target:")
                logger.info(f"   Position: {target_position}%")
                logger.info(f"   Account: {target_account}% @ {leverage}x")
                logger.info(f"   InfluxDB: {leverage_params['total_trades']} trades")
                
                # Check if target hit
                if current_profit >= target_account:
                    return 'roi'
        
        # ═══════════════════════════════════════════════════════
        # TIER 3: Fallback Logic
        # ═══════════════════════════════════════════════════════
        
        # If InfluxDB unavailable or no data
        # Use simple time-based logic
        return None
```

---

## THE MAGIC: HOW CROSS-SERVER COMMUNICATION WORKS

### Network Communication Flow

```
BOT1 Strategy Running on 192.168.3.71
════════════════════════════════════
Every 5 minutes (on new candle):
    │
    ├─ custom_exit() is called for each open trade
    │
    ├─ Needs parameters for BTC/USDT at 3.0x leverage
    │
    └─ Calls: self.stateless_params.get_leverage_params(3.0)
         │
         └─ This triggers a network call
                  │
                  ▼
         ┌──────────────────────────┐
         │   HTTP Request           │
         │   to 192.168.3.6:8086    │
         │   (InfluxDB server)      │
         └──────────────────────────┘
                  │
                  ▼
         InfluxDB Query:
         SELECT * FROM BOT33_trades
         WHERE leverage BETWEEN 2.5 AND 3.5
         AND timestamp > now() - 30d
                  │
                  ▼
         ┌──────────────────────────┐
         │   InfluxDB Response      │
         │   Returns: 3,548 trades  │
         │   Avg stop: -1.48%       │
         │   Avg target: +1.00%     │
         └──────────────────────────┘
                  │
                  ▼
         BOT1 receives parameters
         Cached for 30 seconds
         Used to exit the trade
```

### Actual Network Packets

```
BOT1 (192.168.3.71) → InfluxDB (192.168.3.6)
════════════════════════════════════════════

POST /api/v2/query HTTP/1.1
Host: 192.168.3.6:8086
Authorization: Token your_token
Content-Type: application/json

{
  "query": "from(bucket: 'BOT33_trades') 
            |> range(start: -30d) 
            |> filter(fn: (r) => r.leverage >= 2.5 and r.leverage < 3.5)
            |> filter(fn: (r) => r._measurement == 'closed_trades')
            |> group(columns: ['leverage_bin'])
            |> mean()",
  "org": "freqtrade"
}

Response:
{
  "results": [
    {
      "leverage_bin": "2.5-3.5x",
      "stop_loss_pct": -1.48,
      "take_profit_pct": 1.00,
      "win_rate": 0.61,
      "total_trades": 3548
    }
  ]
}
```

---

## CACHING MECHANISM

To avoid querying InfluxDB on every candle, there's a 30-second cache:

```python
class InfluxDBParameterManager:
    def __init__(self):
        self.cache = {}
        self.cache_timestamps = {}
        self.cache_duration = 30  # seconds
        
    def get_leverage_params(self, leverage):
        cache_key = f"leverage_{leverage}"
        
        # Check if cached data is still valid
        if cache_key in self.cache:
            age = time.time() - self.cache_timestamps[cache_key]
            if age < self.cache_duration:
                # Use cached data (no network call)
                return self.cache[cache_key]
        
        # Cache expired or missing - query InfluxDB
        params = self._query_influxdb(leverage)
        
        # Store in cache
        self.cache[cache_key] = params
        self.cache_timestamps[cache_key] = time.time()
        
        return params
```

**Timeline:**
```
00:00 - BOT1 queries InfluxDB → Gets params → Caches for 30s
00:05 - BOT1 needs params → Uses cache (no network call)
00:10 - BOT1 needs params → Uses cache (no network call)
00:15 - BOT1 needs params → Uses cache (no network call)
00:20 - BOT1 needs params → Uses cache (no network call)
00:25 - BOT1 needs params → Uses cache (no network call)
00:30 - Cache expires
00:30 - BOT1 queries InfluxDB → Gets fresh params → Caches for 30s
... cycle repeats ...
```

---

## TWO PARAMETER SOURCES EXPLAINED

### Source 1: Optuna Parameters (Per-Pair, Future)

**Where it's generated:**
- Server: 192.168.3.33 (BOT3)
- Process: `optuna_optimizer_service.py`
- Method: Candle simulation (backtesting approach)

**What it contains:**
```python
{
    'BTC/USDT': {
        'stop_loss_pct': -1.50,      # Optimized for BTC specifically
        'take_profit_pct': 2.00,      # Different from ETH!
        'win_rate': 0.68,
        'profit_factor': 1.92,
        'total_trades': 892,
        'validated': True
    },
    'ETH/USDT': {
        'stop_loss_pct': -1.20,      # Different from BTC!
        'take_profit_pct': 1.80,
        'win_rate': 0.71,
        'profit_factor': 2.15,
        'total_trades': 1024,
        'validated': True
    }
}
```

**How BOT1/BOT2 access it:**
```python
# During initialization (happens once)
optimizer = OptunaParameterOptimizer(
    influx_url="http://192.168.3.6:8086",  # Network call
    influx_token="your_token"
)
self.optuna_params = optimizer.load_results()  # Loads ALL pairs at once

# During trading (instant lookup, no network call)
if pair in self.optuna_params:
    params = self.optuna_params[pair]  # Dictionary lookup
```

### Source 2: Stateless Manager (Leverage-Aware, Active Now)

**Where it's generated:**
- Servers: ALL (192.168.3.71, 192.168.3.72, 192.168.3.33)
- Process: Running in each bot's strategy
- Method: Real-time query of historical trades

**What it contains:**
```python
{
    'leverage_bin': '2.5-3.5x',      # For 3.0x leverage
    'stop_loss_pct': -1.48,           # Based on 3,548 trades
    'take_profit_pct': 1.00,
    'avg_win': 0.88,
    'avg_loss': -1.46,
    'win_rate': 0.61,
    'total_trades': 3548
}
```

**How BOT1/BOT2 access it:**
```python
# During trading (network call every 30 seconds)
leverage_params = self.stateless_params.get_leverage_params(
    leverage=3.0  # Current trade's leverage
)
# This triggers:
# 1. Check cache (30s validity)
# 2. If expired: HTTP query to 192.168.3.6:8086
# 3. Parse response
# 4. Cache for next 30s
```

---

## PRIORITY SYSTEM

```python
def get_exit_parameters(self, pair, leverage):
    """
    BOT1 and BOT2 use this priority:
    """
    
    # Priority 1: Optuna (most specific)
    if pair in self.optuna_params:
        params = self.optuna_params[pair]
        source = "OPTUNA"
        return params, source
    
    # Priority 2: Stateless Manager (leverage-aware)
    elif leverage_params := self.stateless_params.get_leverage_params(leverage):
        params = leverage_params
        source = "STATELESS"
        return params, source
    
    # Priority 3: Fallback (simple logic)
    else:
        params = {
            'stop_loss_pct': -2.00,
            'take_profit_pct': 1.00
        }
        source = "FALLBACK"
        return params, source
```

---

## REAL EXAMPLE: BOT1 Exits a Trade

**Scenario**: BOT1 has an open BTC/USDT trade at 3.0x leverage

```
Time: 09:23:56
Server: 192.168.3.71 (BOT1)
═══════════════════════════════════════════════════════════

[1] New candle arrives for BTC/USDT
    │
    └─► custom_exit() called
         │
         ├─ trade.pair = 'BTC/USDT'
         ├─ trade.leverage = 3.0
         └─ current_profit = -2.10%
         
[2] Check Optuna parameters
    │
    ├─ if 'BTC/USDT' in self.optuna_params:
    │   │
    │   └─► NOT FOUND (Optuna still running)
    │
    └─► Skip to Stateless Manager
    
[3] Query Stateless Manager
    │
    └─► self.stateless_params.get_leverage_params(3.0)
         │
         ├─ Check cache for "leverage_3.0"
         │   └─► Cache hit! Age: 15 seconds (< 30s limit)
         │       No network call needed
         │
         └─► Return cached params:
              {
                'stop_loss_pct': -1.48,
                'take_profit_pct': 1.00,
                'leverage_bin': '2.5-3.5x'
              }

[4] Calculate account-level thresholds
    │
    ├─ stop_position = -1.48%
    ├─ stop_account = -1.48% × 3.0 = -4.44%
    │
    ├─ target_position = 1.00%
    └─ target_account = 1.00% × 3.0 = 3.00%

[5] Apply 7% risk cap
    │
    └─ max_account_risk = -7.00%
       current_stop = -4.44%
       │
       └─► -4.44% is within -7.00% → OK, use -4.44%
       
[6] Check if stop loss hit
    │
    ├─ current_profit = -2.10%
    ├─ stop_threshold = -4.44%
    │
    └─► -2.10% > -4.44% → Not hit yet, continue trade

[7] Log parameters used
    │
    └─► Logger output:
        "💎 Leverage-aware target:"
        "   Position: 1.00%"
        "   Account: 3.00% @ 3.0x"
        "   InfluxDB: 3548 trades, WR=61.0%"
```

---

## FILE LOCATIONS ON EACH SERVER

### BOT1 (192.168.3.71)
```
/home/ubuntu/freqtrade/
├── user_data/
│   ├── strategies/
│   │   └── ArtiSkyTrader.py          ← Reads from InfluxDB
│   │
│   └── controllers/
│       ├── bot3_ultimate_adaptive_v6_hybird.py  ← InfluxDB reader
│       └── influxdb_parameter_manager.py         ← Stateless Manager
│
└── config_freqai.json                 ← InfluxDB connection details
```

### BOT2 (192.168.3.72)
```
/home/ubuntu/freqtrade/
├── user_data/
│   ├── strategies/
│   │   └── ArtiSkyTrader.py          ← Same code as BOT1
│   │
│   └── controllers/
│       ├── bot3_ultimate_adaptive_v6_hybird.py  ← Same code as BOT1
│       └── influxdb_parameter_manager.py         ← Same code as BOT1
│
└── config_freqai.json                 ← Same InfluxDB connection
```

### BOT3 (192.168.3.33)
```
/home/ubuntu/freqtrade/
├── user_data/
│   ├── strategies/
│   │   └── BOT3MetaStrategyAdaptive.py  ← Also reads from InfluxDB
│   │
│   ├── controllers/
│   │   ├── bot3_ultimate_adaptive_v6_hybird.py  ← Writes trades to InfluxDB
│   │   └── influxdb_parameter_manager.py         ← Stateless Manager
│   │
│   └── services/
│       └── optuna_optimizer_service.py   ← Writes to OPTUNA_PARAMS bucket
│
└── config_freqai.json
```

---

## NETWORK REQUIREMENTS

For this to work, all servers must:

1. **Access InfluxDB on port 8086**
   ```bash
   # Test from BOT1
   curl http://192.168.3.6:8086/ping
   
   # Test from BOT2
   curl http://192.168.3.6:8086/ping
   ```

2. **Have InfluxDB credentials**
   ```python
   INFLUX_URL = "http://192.168.3.6:8086"
   INFLUX_TOKEN = "your_token_here"
   INFLUX_ORG = "freqtrade"
   ```

3. **Install InfluxDB client library**
   ```bash
   pip install influxdb-client
   ```

---

## TROUBLESHOOTING

### BOT1/BOT2 Not Getting Parameters

**Check 1: Network connectivity**
```bash
# From BOT1 server
ssh ubuntu@192.168.3.71
curl http://192.168.3.6:8086/ping

# Should return: 204 No Content
```

**Check 2: InfluxDB credentials**
```bash
# Check logs for authentication errors
tail -50 /home/ubuntu/freqtrade/bot1.log | grep -i "influx\|auth"
```

**Check 3: Data exists in InfluxDB**
```bash
# Query from BOT3
influx query 'from(bucket:"BOT33_trades") |> range(start:-1h) |> count()'

# Should return trade count
```

**Check 4: Code has InfluxDB URL**
```bash
# Check strategy file
grep -n "192.168.3.6" /home/ubuntu/freqtrade/user_data/strategies/ArtiSkyTrader.py
```

---

## BANDWIDTH USAGE

**Query frequency:**
- Optuna params: Once at startup (~1 KB)
- Stateless params: Every 30 seconds (~0.5 KB)
- Per bot: ~1 KB/minute = ~1.4 MB/day

**Total for 3 bots**: ~4.2 MB/day (negligible)

---

## SUMMARY

**How BOT1 and BOT2 get parameters from BOT3:**

1. **BOT3 processes** (Optuna, Stateless) write optimized parameters to **InfluxDB (192.168.3.6)**

2. **InfluxDB acts as a shared database** accessible from all servers

3. **BOT1 and BOT2** have the same code that **queries InfluxDB** over HTTP

4. **Caching** reduces network calls to once per 30 seconds

5. **Priority system**: Optuna (if available) → Stateless Manager → Fallback

6. All communication happens over **standard HTTP/InfluxDB protocol**

**It's like a shared Google Doc that updates every 30 seconds!**

- BOT3 writes the "doc" (InfluxDB)
- BOT1 and BOT2 read the "doc" (HTTP queries)
- Everyone sees the same data in real-time
- No direct server-to-server communication needed
- All through the InfluxDB hub at 192.168.3.6

---

---

## 🆕 NOVEMBER 27, 2025 UPDATE: ENSEMBLE AGGREGATOR ARCHITECTURE

### Discovered During Confidence Mismatch Investigation

**What We Found:**
- There's a **critical component** between Feeders and BOT1/BOT2 that wasn't documented: **The EnsembleEngine/Aggregator**
- Location: `192.168.3.80:/home/ubuntu/freqtrade/freqtrade-aggregator/freqtrade_signal_aggregator.py`
- This service acts as an **ensemble voting layer** before signals reach BOT1/BOT2

### Updated Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                   COMPLETE SIGNAL FLOW ARCHITECTURE                       │
└──────────────────────────────────────────────────────────────────────────┘

    192.168.3.75          192.168.3.80          192.168.3.71/72
    ════════════          ════════════          ════════════════
  Feeders (75a/b/c)      AGGREGATOR            BOT1/BOT2
  - FreqAI models        EnsembleEngine        (Executors)
  - LightGBM quantile    - Voting logic              │
  - Confidence calc      - 60s buffer                │
         │                     │                      │
         │ HTTP POST           │ HTTP POST            │
         │ forceenter()        │ forceenter()         │
         │ enter_tag           │ enter_tag            │
         │                     │                      │
         └────────────────────►│──────────────────────►
                               │                      │
                        ┌──────▼──────┐               │
                        │  Ensemble   │               │
                        │  Decision   │               │
                        │  Voting     │               │
                        └─────────────┘               │
                                                      │
                                               ┌──────▼──────┐
                                               │  Execute    │
                                               │  Trade      │
                                               │  Telegram   │
                                               └─────────────┘
```

### How the Aggregator Works

**Configuration** (from `freqtrade_signal_aggregator.py`):
```python
ENSEMBLE_WINDOW_SECONDS = 60   # Collect signals for 60 seconds
MIN_AGREEMENTS = 1             # Require 1/5 feeders to agree
MIN_CONFIDENCE = 0.70          # Minimum 70% average confidence
```

**Signal Processing Flow:**

1. **Signal Reception** (Lines 332-385):
   - Feeders send signals via FreqTrade REST API to port 8080
   - Aggregator parses enter_tag to extract feeder ID and confidence
   - Signals buffered for 60 seconds

2. **Ensemble Voting** (Lines 255-305):
   - Groups signals by pair and direction (LONG/SHORT)
   - Deduplicates by source (keeps latest from each feeder)
   - Checks if MIN_AGREEMENTS met (currently 1/5)
   - Calculates average confidence

3. **Decision Forwarding** (Lines 307-350):
   - Creates new enter_tag: `"ensemble {votes}/5 conf={avg_conf} from={sources}"`
   - Forwards to BOT1 (LONG) at 192.168.3.71 or BOT2 (SHORT) at 192.168.3.72
   - BOT1/BOT2 execute trade and send Telegram notification

**Example Signal Flow:**

```
T=0s    Feeder 75b: LightGBM calculates confidence=0.89
        Sends: "long from 75b conf=0.89 (DQ: profit=3.05%, safety=0.39%)"
        
T=0s    Aggregator: Receives signal, parses conf=0.89, buffers

T=60s   Aggregator: Checks buffer, finds 1 feeder agreed
        Creates: "long ensemble 1/5 conf=0.89 from=75b"
        Forwards to BOT1 at 192.168.3.71
        
T=60s   BOT1: Executes forceenter with ensemble enter_tag
        Sends Telegram with "conf=0.89"
```

### Network Communication

**Feeders → Aggregator:**
```python
# From: 192.168.3.75 (Feeder 75b)
# To: 192.168.3.80:8080 (Aggregator)

POST /api/v1/forceenter HTTP/1.1
Host: 192.168.3.80:8080
Authorization: Basic freqtrader:TuongLai7
Content-Type: application/json

{
  "pair": "BTC/USDT:USDT",
  "side": "long",
  "leverage": 3,
  "ordertype": "market",
  "enter_tag": "long from 75b conf=0.89 (DQ: profit=3.05%, safety=0.39%)"
}
```

**Aggregator → BOT1:**
```python
# From: 192.168.3.80 (Aggregator)
# To: 192.168.3.71:8080 (BOT1)

POST /api/v1/forceenter HTTP/1.1
Host: 192.168.3.71:8080
Authorization: Basic freqtrader:TuongLai7
Content-Type: application/json

{
  "pair": "BTC/USDT:USDT",
  "side": "long",
  "leverage": 3,
  "ordertype": "market",
  "enter_tag": "long ensemble 1/5 conf=0.89 from=75b"
}
```

### Lessons Learned from Confidence Mismatch Fix

**Issue:** Telegram showed `conf=0.75` instead of feeder's calculated `conf=0.89`

**Root Cause:** Aggregator hardcoded confidence to 0.75 because feeders didn't include it in enter_tag

**Fix Applied:**
1. Modified feeders to include confidence: `"long from 75b conf=0.89 (...)"`
2. Modified aggregator to parse confidence from tag

**Key Lessons:**

1. **Always Trace Complete Signal Flow**
   - Don't assume you know all components
   - Follow data through every hop
   - Document discovered components immediately

2. **Tag-Based Communication Has Limitations**
   - Requires string parsing (prone to errors)
   - Format mismatches cause silent failures
   - Consider structured metadata fields

3. **Default Values Can Hide Issues**
   - Aggregator defaulted to 0.75 silently
   - Add logging when using defaults
   - Alert when defaults used frequently

4. **Multi-Hop Systems Need End-to-End Testing**
   - Test complete flow: Feeder → Aggregator → BOT1 → Telegram
   - Verify data integrity at each hop
   - Use integration tests spanning services

5. **Documentation Is Critical**
   - This aggregator wasn't documented anywhere
   - Had to reverse-engineer its existence
   - Maintain up-to-date architecture diagrams

### Service Management

**Aggregator is NOT a systemd service!**
- Running as standalone Python process: `python3 freqtrade_signal_aggregator.py`
- Started with: `nohup python3 freqtrade_signal_aggregator.py > /dev/null 2>&1 &`
- Check status: `ps aux | grep freqtrade_signal_aggregator`
- Restart: Kill process and start new one

**Important:** Don't assume all services use systemctl!

### Related Documentation

- **Complete fix details:** `CONFIDENCE_MISMATCH_FIX_COMPLETE_2025-11-27.md`
- **Feeder strategies:** `/home/freqai/freqtrade/user_data/strategies/ProducerStrategyDualQuantile_LONG.py` at 192.168.3.75
- **Aggregator code:** `/home/ubuntu/freqtrade/freqtrade-aggregator/freqtrade_signal_aggregator.py` at 192.168.3.80

---

**End of Technical Deep Dive**
