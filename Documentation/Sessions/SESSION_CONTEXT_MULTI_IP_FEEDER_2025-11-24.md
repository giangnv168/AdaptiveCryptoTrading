# Session Context: Multi-IP Feeder Setup for Host 120
**Date:** 2025-11-24
**Status:** BLOCKED - Router configuration needed

## Summary
Setting up 3 feeders on Host 120 with 3 different public IPs to avoid Binance API rate limits.

## Current State
- **Host 120:** 3 NICs configured with IPs
- **Feeders 120a, 120b, 120c:** Running (but all using 192.168.3.120)
- **Issue:** New NICs cannot reach gateway - router needs configuration

### NIC Configuration
| NIC | IP | MAC Address | Status |
|-----|-----|-------------|--------|
| ens160 | 192.168.3.120 | (original) | ✅ Working |
| ens192 | 192.168.3.220 | 00:0c:29:bf:34:18 | ❌ Can't reach gateway |
| ens224 | 192.168.3.221 | 00:0c:29:bf:34:22 | ❌ Can't reach gateway |

## REQUIRED: Router Configuration

The new NICs have IPs assigned but **cannot reach the gateway (192.168.3.250)**. You need to configure your router/switch to:

### Option 1: Add Static ARP Entries (on router)
```
# Add ARP entries for the new MAC addresses
arp -s 192.168.3.220 00:0c:29:bf:34:18
arp -s 192.168.3.221 00:0c:29:bf:34:22
```

### Option 2: Configure NAT Rules (on router)
Map each internal IP to a different public IP:
```
192.168.3.120 → Public IP 1 (existing)
192.168.3.220 → Public IP 2 (new)
192.168.3.221 → Public IP 3 (new)
```

### After Router Configuration - Test Connectivity:
```bash
# SSH to Host 120 and test
ping -I ens192 8.8.8.8  # Should work after router config
ping -I ens224 8.8.8.8  # Should work after router config
```

## What Was Tried

### 1. Socket Binding Wrapper (`freqtrade_bind_ip.py`)
- Created a Python wrapper that patches `socket.create_connection` to bind outbound connections to specific IPs
- **Result:** Didn't work because CCXT uses aiohttp (async) which has its own connection handling

### 2. Network Namespaces
- Created network namespaces with veth pairs
- **Result:** Complex routing issues with Docker iptables rules

### 3. Current: Multiple NICs (IN PROGRESS)
- Added 2 NICs (ens192, ens224) ✅
- IPs assigned ✅
- Policy routing configured ✅
- **BLOCKED:** Router needs to be configured

## Current Configuration Done on Host 120

### Policy Routing (already configured)
```bash
# IP rules created
ip rule list | grep rt
# 32764: from 192.168.3.221 lookup rt221
# 32765: from 192.168.3.220 lookup rt220

# Routing tables in /etc/iproute2/rt_tables
# 121 rt220
# 122 rt221
```

### Configs Already Have Local Address
The feeder configs have `ccxt_async_config.local_addr` set:
- config_feeder_120a.json: `"local_addr": "192.168.3.120"`
- config_feeder_120b.json: `"local_addr": "192.168.3.220"`
- config_feeder_120c.json: `"local_addr": "192.168.3.221"`

## Next Steps After Router is Configured

1. **Verify connectivity from all NICs:**
   ```bash
   ping -c 1 -I 192.168.3.220 8.8.8.8
   ping -c 1 -I 192.168.3.221 8.8.8.8
   ```

2. **Restart feeders to pick up routing:**
   ```bash
   sudo systemctl restart freqtrade-feeder120a freqtrade-feeder120b freqtrade-feeder120c
   ```

3. **Verify connections use different IPs:**
   ```bash
   ss -tnp | grep -E '(python|freqtrade).*:443'
   ```

## Architecture Goal

```
Host 120 (3 NICs):
├── ens160 (192.168.3.120) → NAT → Public IP 1 → Feeder 120a
├── ens192 (192.168.3.220) → NAT → Public IP 2 → Feeder 120b
└── ens224 (192.168.3.221) → NAT → Public IP 3 → Feeder 120c
```

## Commands Reference

```bash
# Check NIC status and IPs
ssh freqai@192.168.3.120 "ip addr show | grep 'inet '"

# Test connectivity from each IP
ssh freqai@192.168.3.120 "ping -c 1 -I 192.168.3.220 8.8.8.8"
ssh freqai@192.168.3.120 "ping -c 1 -I 192.168.3.221 8.8.8.8"

# Start feeders
ssh freqai@192.168.3.120 "sudo systemctl start freqtrade-feeder120a freqtrade-feeder120b freqtrade-feeder120c"

# Check connections with source IPs
ssh freqai@192.168.3.120 "ss -tnp | grep -E ':443.*freqtrade'"
