# BOT2 Credentials and Access

**Last Updated:** 2025-12-01

---

## 🔐 BOT2 Server Access

**Host:** 192.168.3.72  
**Username:** ubuntu  
**Password:** ArtiSky@7

**SSH Command:**
```bash
ssh ubuntu@192.168.3.72
# Password: ArtiSky@7
```

**Or with sshpass:**
```bash
sshpass -p 'ArtiSky@7' ssh ubuntu@192.168.3.72
```

---

## 🤖 BOT2 Services

### Main Trading Bot
**Status Check:**
```bash
ssh ubuntu@192.168.3.72 "ps aux | grep freqtrade | grep ConsumerStrategy"
```

**Log File:**
```bash
ssh ubuntu@192.168.3.72 "tail -f /home/ubuntu/freqtrade/bot2.log"
```

### Optuna Optimization
**Check Running Optuna:**
```bash
ssh ubuntu@192.168.3.72 "ps aux | grep optuna"
```

**Optuna Log:**
```bash
ssh ubuntu@192.168.3.72 "tail -f /home/ubuntu/freqtrade/user_data/logs/optuna_continuous_BOT2.log"
```

**Optuna Continuous Script:**
```bash
/home/ubuntu/freqtrade/optuna_continuous_bot2_with_signal.sh
```

---

## 📁 Important Directories

**Freqtrade Installation:**
```
/home/ubuntu/freqtrade/
```

**Configuration:**
```
/home/ubuntu/freqtrade/config_freqai.json
```

**Strategy:**
```
ConsumerStrategy
```

**Logs:**
```
/home/ubuntu/freqtrade/bot2.log
/home/ubuntu/freqtrade/user_data/logs/optuna_continuous_BOT2.log
```

---

## 🔄 Optuna Parameters

**Stored in InfluxDB:**
- Bucket: `OPTUNA_BOT2`
- Measurement: `optimized_parameters`
- Fields: `take_profit`, `cutloss`, `volatility_multiplier`, `win_rate`, `profit_factor`

**Optimization Command:**
```bash
ssh ubuntu@192.168.3.72 "cd /home/ubuntu/freqtrade && python3 user_data/services/run_optuna_per_bot_standalone.py --bot BOT2 --days-back 90 --n-trials 100"
```

---

## 📊 Monitoring

**Check Optimization Progress:**
```bash
ssh ubuntu@192.168.3.72 "tail -100 /home/ubuntu/freqtrade/user_data/logs/optuna_continuous_BOT2.log"
```

**Check InfluxDB for New Parameters:**
```bash
# From any host with InfluxDB access
python3 << 'PYEOF'
from influxdb_client import InfluxDBClient
client = InfluxDBClient(
    url='http://192.168.3.6:8086',
    token='uvAgH7X2WJxE_RGmZC4wUEuNLsZX2e9Uyhxg1lB9euW4I2qV6RjUZmQlzSTOJPoHbO2tLvarifULUCgvArVEXA==',
    org='ArtiSky'
)
query = '''
from(bucket: "OPTUNA_BOT2")
|> range(start: -1h)
|> filter(fn: (r) => r._measurement == "optimized_parameters")
|> last()
'''
result = client.query_api().query(query)
for table in result:
    for record in table.records:
        print(f"{record.values}")
PYEOF
```

---

## ⚙️ Common Tasks

### Restart BOT2
```bash
ssh ubuntu@192.168.3.72 "pkill -f 'freqtrade trade.*ConsumerStrategy'"
ssh ubuntu@192.168.3.72 "cd /home/ubuntu/freqtrade && nohup .venv/bin/freqtrade trade --config config_freqai.json --strategy ConsumerStrategy --logfile bot2.log &"
```

### Restart Optuna Continuous
```bash
ssh ubuntu@192.168.3.72 "pkill -f optuna_continuous_bot2_with_signal.sh"
ssh ubuntu@192.168.3.72 "cd /home/ubuntu/freqtrade && nohup ./optuna_continuous_bot2_with_signal.sh &"
```

### Manual Optuna Run
```bash
ssh ubuntu@192.168.3.72 "cd /home/ubuntu/freqtrade && python3 user_data/services/run_optuna_per_bot_standalone.py --bot BOT2 --days-back 90 --n-trials 100"
```

---

## 🔗 Related Documentation

- [[OPTUNA_COMPLETE_INTEGRATION_SUMMARY]]
- [[BOT2_OPTUNA_COMPLETE_FIX_SESSION_2025-11-29]]
- [[OPTUNA_PER_BOT_USAGE_GUIDE]]

---

**Security Note:** These credentials are for internal use only. Store securely and do not commit to public repositories.
