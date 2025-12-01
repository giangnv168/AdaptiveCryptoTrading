# Session Context: Docker-Based Multi-IP Feeder Solution
**Date:** 2025-11-24  
**Host:** 192.168.3.120

## Problem
Binance rate limiting the 163-pair feeder - need to split across 3 IPs on Host 120.

## Current Status
- **Network Setup Complete:**
  - 192.168.3.120 (ens160) - Working ✅
  - 192.168.3.220 (ens192) - Working ✅ (Router NAT configured)
  - 192.168.3.221 (ens224) - Needs Router NAT 1:1 ❌

- **Custom Exchange Class Approach - FAILED:**
  - Created `BinanceLocalAddr` class to bind to specific local_addr
  - Issue: Exchange name case-sensitivity caused feeders to crash
  - Reverted configs back to "binance"

## Recommended Solution: Docker with Macvlan Network

Docker macvlan allows each container to have its own IP address directly on the physical network.

### Architecture
```
Host 120 (192.168.3.120)
├── Docker macvlan network (192.168.3.0/24)
│   ├── feeder120a container → IP: 192.168.3.120
│   ├── feeder120b container → IP: 192.168.3.220
│   └── feeder120c container → IP: 192.168.3.221
```

### Implementation Steps

#### 1. Install Docker on Host 120
```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker freqai
```

#### 2. Create Macvlan Network
```bash
# Create macvlan network using the parent interface
docker network create -d macvlan \
  --subnet=192.168.3.0/24 \
  --gateway=192.168.3.1 \
  --ip-range=192.168.3.120/30 \
  -o parent=ens160 \
  freqtrade_macvlan
```

#### 3. Docker Compose for Multi-IP Feeders
```yaml
version: '3.8'

services:
  feeder120a:
    image: freqtradeorg/freqtrade:stable
    container_name: feeder120a
    volumes:
      - ./user_data:/freqtrade/user_data
      - ./config_feeder_120a.json:/freqtrade/config.json
    command: trade --dry-run --strategy ProducerStrategy --config /freqtrade/config.json --logfile /freqtrade/user_data/logs/feeder120a.log
    networks:
      freqtrade_macvlan:
        ipv4_address: 192.168.3.120
    restart: unless-stopped

  feeder120b:
    image: freqtradeorg/freqtrade:stable
    container_name: feeder120b
    volumes:
      - ./user_data:/freqtrade/user_data
      - ./config_feeder_120b.json:/freqtrade/config.json
    command: trade --dry-run --strategy ProducerStrategy --config /freqtrade/config.json --logfile /freqtrade/user_data/logs/feeder120b.log
    networks:
      freqtrade_macvlan:
        ipv4_address: 192.168.3.220
    restart: unless-stopped

  feeder120c:
    image: freqtradeorg/freqtrade:stable
    container_name: feeder120c
    volumes:
      - ./user_data:/freqtrade/user_data
      - ./config_feeder_120c.json:/freqtrade/config.json
    command: trade --dry-run --strategy ProducerStrategy --config /freqtrade/config.json --logfile /freqtrade/user_data/logs/feeder120c.log
    networks:
      freqtrade_macvlan:
        ipv4_address: 192.168.3.221
    restart: unless-stopped

networks:
  freqtrade_macvlan:
    external: true
```

### Benefits of Docker Approach
1. **Clean IP isolation** - Each container gets its own IP on the LAN
2. **No code modifications** - Standard freqtrade image works
3. **Easy management** - docker-compose up/down
4. **Portable** - Same setup works on any host

### Prerequisites
1. Router NAT 1:1 for 192.168.3.221 (currently missing)
2. Docker installed on Host 120
3. Remove `local_addr` from configs (not needed with macvlan)

## Alternative: iptables SNAT (Simpler, No Docker)

If Docker is not preferred, use iptables SNAT with cgroups:

```bash
# Mark packets from specific PIDs
iptables -t mangle -A OUTPUT -m owner --pid-owner <PID> -j MARK --set-mark 1

# SNAT marked packets to specific source IP
iptables -t nat -A POSTROUTING -m mark --mark 1 -j SNAT --to-source 192.168.3.220
```

This requires scripts to track PIDs and update iptables rules when feeders restart.

## Files Created This Session
- `deploy_multi_ip_fix.sh` - Custom exchange class deployment (not working)
- `freqtrade/exchange/binance_local_addr.py` (on Host 120) - Can be removed

## Next Steps
1. **User to configure router NAT 1:1 for 192.168.3.221**
2. **Choose approach:**
   - Docker macvlan (recommended)
   - iptables SNAT (simpler but more fragile)
3. Implement chosen solution
4. Test all 3 feeders use different IPs
