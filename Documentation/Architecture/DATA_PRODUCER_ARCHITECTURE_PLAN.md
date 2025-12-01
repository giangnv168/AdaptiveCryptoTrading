# Data Producer Architecture Plan

## Overview

Set up centralized Data Producers that download all OHLCV data from Binance, then broadcast to FreqAI feeders as consumers.

## Current Architecture

```
Host 75  ─────→ Binance API (110 pairs)
Host 120 ─────→ Binance API (110 pairs)
Host 121 ─────→ Binance API (110 pairs)
Host 124 ─────→ Binance API (110 pairs)
Total: 4 × 110 = 440 pair downloads hitting Binance
```

**Problems:**
- High API rate limiting risk
- 403 Forbidden errors (IP blocking)
- Data inconsistency between feeders
- Wasted bandwidth (same data downloaded 4 times)

## Proposed Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DATA PRODUCER LAYER                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌───────────────────┐    ┌───────────────────┐           │
│   │  Host 73          │    │  Host 108         │           │
│   │  DATA PRODUCER 1  │    │  DATA PRODUCER 2  │           │
│   │  55 pairs         │    │  55 pairs         │           │
│   │  Port 8080        │    │  Port 8080        │           │
│   │  NAT IPs 1-6      │    │  NAT IPs 7-12     │           │
│   └─────────┬─────────┘    └─────────┬─────────┘           │
│             │ WebSocket              │ WebSocket            │
│             │                        │                      │
└─────────────┼────────────────────────┼──────────────────────┘
              │                        │
              ▼                        ▼
┌─────────────────────────────────────────────────────────────┐
│                   FREQAI CONSUMER LAYER                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│   │ Host 75  │ │ Host 120 │ │ Host 121 │ │ Host 124 │      │
│   │ FreqAI   │ │ FreqAI   │ │ FreqAI   │ │ FreqAI   │      │
│   │ Consumer │ │ Consumer │ │ Consumer │ │ Consumer │      │
│   │ 110 pairs│ │ 110 pairs│ │ 110 pairs│ │ 110 pairs│      │
│   └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Benefits

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Binance API Calls | 4 hosts × 110 | 2 hosts × 55 | **75% reduction** |
| Rate Limit Risk | High | Low | Significant |
| Data Consistency | Variable | Identical | Perfect sync |
| IP Usage | 1 IP per host | 12 NAT IPs rotating | Better distribution |

## Implementation Steps

### Phase 1: Set Up Data Producers (Host 73 & 108)

#### 1.1 Producer 1 Config (Host 73) - Pairs 1-55
```json
{
    "strategy": "StrategyNone",
    "exchange": {
        "name": "binance",
        "pair_whitelist": [/* pairs 1-55 */],
        "use_ws": true
    },
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "verbosity": "error",
        "enable_openapi": false,
        "jwt_secret_key": "producer73_secret_key_change_me",
        "ws_token": "producer73_ws_token_change_me"
    },
    "internals": {
        "process_only_new_candles": true
    }
}
```

#### 1.2 Producer 2 Config (Host 108) - Pairs 56-110
```json
{
    "strategy": "StrategyNone",
    "exchange": {
        "name": "binance",
        "pair_whitelist": [/* pairs 56-110 */],
        "use_ws": true
    },
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "verbosity": "error",
        "enable_openapi": false,
        "jwt_secret_key": "producer108_secret_key_change_me",
        "ws_token": "producer108_ws_token_change_me"
    }
}
```

### Phase 2: Configure FreqAI Feeders as Consumers

#### 2.1 Consumer Config Addition (All FreqAI hosts)
```json
{
    "external_message_consumer": {
        "enabled": true,
        "producers": [
            {
                "name": "producer_73",
                "host": "192.168.3.73",
                "port": 8080,
                "ws_token": "producer73_ws_token_change_me"
            },
            {
                "name": "producer_108",
                "host": "192.168.3.108",
                "port": 8080,
                "ws_token": "producer108_ws_token_change_me"
            }
        ],
        "wait_timeout": 30,
        "initial_candle_sync": true,
        "pair_whitelist": [".*"]
    }
}
```

### Phase 3: NAT IP Rotation (Optional Enhancement)

For even better distribution, configure producers to use different outbound IPs:

```bash
# On Producer hosts, add to systemd service or startup script:
# Use specific NAT IP for outbound connections
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source <NAT_IP>
```

## Deployment Commands

### Deploy Producer on Host 73
```bash
# 1. Create producer config
sshpass -p 'NewHopes@168' ssh freqai@192.168.3.73 "
cd /home/freqai/freqtrade && 
cat > config_producer_73.json << 'EOF'
{...producer config...}
EOF"

# 2. Create systemd service
# 3. Enable and start service
```

### Update Consumers on Hosts 75, 120, 121, 124
```bash
# Add external_message_consumer to existing configs
# Restart services
```

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Producer failure | 2 redundant producers |
| Network latency | Local network (< 1ms) |
| Data sync issues | `initial_candle_sync: true` |
| Consumer can't connect | Fallback to direct Binance API |

## Timeline

| Phase | Duration | Tasks |
|-------|----------|-------|
| Phase 1 | 30 min | Set up producers on hosts 73/108 |
| Phase 2 | 30 min | Update consumer configs on hosts 75/120/121/124 |
| Phase 3 | 15 min | Testing and validation |
| Phase 4 | Optional | NAT IP rotation setup |

## Ready to Implement?

This plan reduces Binance API load by 75% and eliminates rate limiting issues.

Shall I proceed with implementation?

---
Created: 2025-11-23
Status: PLANNING
