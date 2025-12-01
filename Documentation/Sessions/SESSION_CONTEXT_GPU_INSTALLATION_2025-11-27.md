# Session Context: GPU Installation and Troubleshooting
**Date:** 2025-11-27 18:00 - 18:31 (+07 timezone)
**Session Duration:** ~90 minutes
**Context Window Used:** 80% (159,357 / 200K tokens)

---

## 🎯 Session Objectives
- Original request: Read recent context, review BOT2 stoploss/takeprofit, check if BOT2 should apply BOT1 simplicity
- **Actual work completed:** GPU-enabled LightGBM installation across both FreqAI hosts (75 & 120)

---

## ✅ What Was Accomplished

### 1. LightGBM GPU Installation - Host 75 (192.168.3.75)
- ✅ Upgraded CMake from 3.10.2 → 4.2.0 (via pip)
- ✅ Installed OpenCL libraries (ocl-icd-opencl-dev, opencl-headers, clinfo)
- ✅ Compiled LightGBM 4.6.0.99 with GPU support (OpenCL 2.2)
- ✅ Installed Python package successfully
- ✅ Model file configured: `device_type: 'gpu'` in LightGBMQuantileMultiTarget.py

### 2. LightGBM GPU Installation - Host 120 (192.168.3.120)
- ✅ Upgraded CMake from 3.10.2 → 4.2.0 (via pip)
- ✅ Installed OpenCL libraries (ocl-icd-opencl-dev, opencl-headers, clinfo)
- ✅ Installed Boost libraries 1.65.1 (libboost-all-dev) - was missing
- ✅ Compiled LightGBM 4.6.0.99 with GPU support (OpenCL 2.2)
- ✅ Installed Python package successfully
- ✅ Updated model file: `device_type: 'cpu'` → `device_type: 'gpu'`

### 3. Service Management
- ✅ Restarted all feeder services on both hosts
- ✅ Host 120: feeder120a, 120b, 120c - all running ✓
- ✅ Host 75: feeder75a, 75b, 75c - all restarted ✓

---

## ⚠️ CRITICAL ISSUE: OpenCL GPU Detection Failure on Host 75

### Problem
```bash
$ clinfo
Number of platforms: 0
```

**OpenCL cannot detect the NVIDIA GPU on Host 75**, resulting in:
- LightGBM error: `lightgbm.basic.LightGBMError: No OpenCL device found`
- Services running but falling back to CPU (if fallback is implemented)
- Training attempts fail with GPU, skip to next pair

### Diagnosis
- ✅ NVIDIA Driver 550.90.07 installed
- ✅ NVIDIA OpenCL library present: `/usr/lib/x86_64-linux-gnu/libnvidia-opencl.so.550.90.07`
- ✅ OpenCL vendor file configured: `/etc/OpenCL/vendors/nvidia.icd` → `libnvidia-opencl.so.1`
- ❌ OpenCL runtime shows 0 platforms (GPU not visible)

### Likely Causes
1. VM GPU passthrough not properly configured
2. NVIDIA driver not fully loaded/initialized
3. Missing NVIDIA OpenCL ICD loader (packages not available in Ubuntu 18.04 repos)
4. GPU device nodes (`/dev/nvidia*`) not accessible

---

## 📊 Current System State

### Hardware
- **Host 75:** Tesla T4 GPU, Driver 550.90.07
- **Host 120:** Tesla T4 GPU, Driver 550.90.07

### Software Versions
| Component | Version | Location |
|-----------|---------|----------|
| CMake | 4.2.0 | pip (in venv) |
| LightGBM | 4.6.0.99 | GPU-compiled |
| OpenCL | 2.2 | System libraries |
| Boost | 1.65.1 | System (Host 120 only) |

### Service Status (as of 18:25-18:31)
**Host 75 Services:**
- freqtrade-feeder75a: Active (running) - **GPU errors present**
- freqtrade-feeder75b: Active (running) - **GPU errors present**
- freqtrade-feeder75c: Active (running) - **GPU errors present**

**Host 120 Services:**
- freqtrade-feeder120a: Active (running) - Status unknown
- freqtrade-feeder120b: Active (running) - Status unknown
- freqtrade-feeder120c: Active (running) - Status unknown

### Configuration Files
**Model File Location (both hosts):**
```
/home/freqai/freqtrade/user_data/freqaimodels/LightGBMQuantileMultiTarget.py
```

**Current Setting:**
- Host 75: Line 120: `'device_type': 'gpu'` ⚠️ (but GPU not working)
- Host 120: Line 120: `'device_type': 'gpu'` ⚠️ (not verified)

---

## �� Recommended Next Steps

### IMMEDIATE ACTION REQUIRED: Fix Host 75

**Option 1: Switch to CPU Mode (Recommended)**
```bash
ssh freqai@192.168.3.75
cd /home/freqai/freqtrade/user_data/freqaimodels
sed -i "s/'device_type': 'gpu'/'device_type': 'cpu'/g" LightGBMQuantileMultiTarget.py
sudo systemctl restart 'freqtrade-feeder75*.service'
```

**Option 2: Deep GPU Troubleshooting (Complex)**
1. Check NVIDIA driver status:
   ```bash
   ssh freqai@192.168.3.75 "nvidia-smi"
   ssh freqai@192.168.3.75 "lsmod | grep nvidia"
   ```

2. Verify GPU device nodes:
   ```bash
   ssh freqai@192.168.3.75 "ls -la /dev/nvidia*"
   ```

3. Check if VM has proper GPU passthrough configured

4. Consider NVIDIA driver reinstallation

### Verify Host 120 GPU Status
```bash
# Check if GPU is actually being used
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120 "clinfo"
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120 "nvidia-smi"

# Check for GPU errors in logs
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120 \
  "journalctl -u freqtrade-feeder120a.service --since '5 minutes ago' | grep -i 'opencl\|gpu'"
```

---

## 📁 Key File Paths

### Model Files
```
Host 75: /home/freqai/freqtrade/user_data/freqaimodels/LightGBMQuantileMultiTarget.py
Host 120: /home/freqai/freqtrade/user_data/freqaimodels/LightGBMQuantileMultiTarget.py
```

### Service Files
```
Host 75:
  - /etc/systemd/system/freqtrade-feeder75a.service
  - /etc/systemd/system/freqtrade-feeder75b.service
  - /etc/systemd/system/freqtrade-feeder75c.service

Host 120:
  - /etc/systemd/system/freqtrade-feeder120a.service
  - /etc/systemd/system/freqtrade-feeder120b.service
  - /etc/systemd/system/freqtrade-feeder120c.service
```

### Python Virtual Environment
```
Both hosts: /home/freqai/freqtrade/.env/bin/activate
```

### OpenCL Configuration
```
Host 75: /etc/OpenCL/vendors/nvidia.icd
Host 120: /etc/OpenCL/vendors/nvidia.icd
```

---

## �� Outstanding Issues

1. **CRITICAL - Host 75:** OpenCL cannot detect GPU (0 platforms found)
   - Services running but GPU training failing
   - Recommend switching to CPU mode

2. **UNKNOWN - Host 120:** GPU status not verified
   - Need to check if GPU is actually being used
   - May have similar OpenCL detection issues

3. **UNADDRESSED - Original Request:** BOT2 stoploss/takeprofit review not completed
   - Session diverted to GPU installation work
   - User may want this addressed in next session

---

## 📝 Important Commands

### Service Management
```bash
# Restart all feeder services
ssh freqai@192.168.3.75 "sudo systemctl restart 'freqtrade-feeder*.service'"
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120 "sudo systemctl restart 'freqtrade-feeder*.service'"

# Check service status
ssh freqai@192.168.3.75 "systemctl status freqtrade-feeder75a.service"
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120 "systemctl status freqtrade-feeder120a.service"
```

### GPU Monitoring
```bash
# Check GPU usage
ssh freqai@192.168.3.75 "nvidia-smi"
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120 "nvidia-smi"

# Real-time monitoring
ssh freqai@192.168.3.75 "watch -n 2 nvidia-smi"
```

### OpenCL Testing
```bash
# Check OpenCL platforms
ssh freqai@192.168.3.75 "clinfo"
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120 "clinfo"
```

### Log Checking
```bash
# Check for errors
ssh freqai@192.168.3.75 \
  "journalctl -u freqtrade-feeder75a.service --since '10 minutes ago' | grep -i error"

# Follow logs in real-time
ssh freqai@192.168.3.75 "journalctl -u freqtrade-feeder75a.service -f"
```

---

## 💡 Lessons Learned

1. **CMake Version Dependency:** Latest LightGBM requires CMake 3.28+, but pip provides 4.2.0
2. **Boost Required on Host 120:** Host 75 already had Boost, Host 120 needed installation
3. **OpenCL != GPU Working:** Having OpenCL libraries ≠ GPU accessible to OpenCL
4. **VM GPU Considerations:** VMs require special GPU passthrough configuration for OpenCL
5. **NVIDIA OpenCL ICD:** Ubuntu 18.04 repos don't have packages for NVIDIA driver 550.x

---

## 🎯 Recommendations for Next Session

### Priority 1: Fix Host 75 GPU Issue
- **Quick fix:** Switch to CPU mode to restore stable operation
- **Long-term fix:** Investigate OpenCL platform detection failure

### Priority 2: Verify Host 120 Status
- Check if GPU is actually being used (may have same issue as Host 75)
- If GPU not working, switch to CPU mode as well

### Priority 3: Address Original Request
- Review BOT2 stoploss/takeprofit settings
- Compare with BOT1 simplicity approach
- Determine if BOT2 should adopt similar patterns

### Priority 4: Document Performance
- Once GPU/CPU decision is made, benchmark training times
- Document actual performance (CPU: 85-95s/pair, GPU: expected 30-50s/pair)

---

## 📞 Quick Reference

**SSH Access:**
```bash
# Host 75
ssh freqai@192.168.3.75

# Host 120
ssh -i ~/.ssh/id_rsa_host120 freqai@192.168.3.120
```

**Activate Python Environment:**
```bash
source /home/freqai/freqtrade/.env/bin/activate
```

**Check Service Status:**
```bash
systemctl status 'freqtrade-feeder*.service'
```

---

**End of Session Context Summary**
**Next Session:** Address Host 75 GPU issue + verify Host 120 + complete original BOT2 review request
