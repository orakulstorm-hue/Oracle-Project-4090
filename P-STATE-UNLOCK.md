# P-State Unlock: Technical Deep Dive

**For those who want to understand WHY, not just HOW.**

---

## Table of Contents

1. [What Are P-States?](#what-are-p-states)
2. [The NVIDIA Power Management Problem](#the-nvidia-power-management-problem)
3. [Detection Methods](#detection-methods)
4. [Unlock Techniques](#unlock-techniques)
5. [Verification & Monitoring](#verification--monitoring)
6. [Advanced Troubleshooting](#advanced-troubleshooting)
7. [Platform-Specific Notes](#platform-specific-notes)

---

## What Are P-States?

P-States (Performance States) are NVIDIA's GPU power management system, similar to CPU C-states.

### P-State Hierarchy (RTX 40-series):

| P-State | Description | Typical Clock | Power | Use Case |
|---------|-------------|---------------|-------|----------|
| **P8** | Deep sleep | 210-450 MHz | 20-50W | Boot state, idle |
| **P5** | Light sleep | 800-1200 MHz | 80-150W | Desktop, light work |
| **P3** | Medium work | 1500-1800 MHz | 150-250W | Video playback, browsing |
| **P2** | Active compute | 2400-2600 MHz | 300-400W | **AI training (default)** |
| **P0** | Maximum performance | 2700-2790 MHz | 400-450W | **Gaming (default)** |

### The Problem:

**What users expect:**
```
AI Training → GPU goes to P0/P2 → Full performance
```

**What actually happens:**
```
AI Training → GPU stays in P5/P8 → Throttled performance
            → But shows "high utilization" in nvidia-smi
            → Users think it's working at full speed
            → Actually running at 25-50% effective performance
```

---

## The NVIDIA Power Management Problem

### Why Does This Happen?

NVIDIA driver has **heuristics** to determine workload priority:

**High Priority (gets P0):**
- 3D games (detected via DirectX/Vulkan/OpenGL API calls)
- Video rendering (Adobe Premiere, DaVinci Resolve)
- CUDA applications with specific API patterns

**Low Priority (stays in P5/P8):**
- PyTorch training (CUDA compute, but "background" pattern)
- TensorFlow (same issue)
- General CUDA compute
- Mining (intentionally throttled)

### The "Chicken Leather Jacket" Reference

This is a jab at Jensen Huang (NVIDIA CEO), famous for his leather jacket.

**The metaphor:**
```
You: Pay $1600 for RTX 4090
Promise: "Ultimate AI training performance"
Reality: GPU throttled to 25-50% unless you know the unlock trick
Effect: Like buying a Ferrari but having handbrake engaged by default
```

**Why NVIDIA does this:**
1. Protect GPUs from thermal damage (liability)
2. Reduce RMA rates from overheating
3. Differentiate consumer cards from datacenter (H100/A100)
4. Encourage enterprise customers to buy expensive datacenter GPUs

**The irony:** Consumer cards are CAPABLE of full speed, but artificially limited by software.

---

## Detection Methods

### Method 1: Check Current P-State

**Windows:**
```powershell
nvidia-smi -q -d PERFORMANCE | Select-String "Performance State"
```

**Linux:**
```bash
nvidia-smi -q -d PERFORMANCE | grep "Performance State"
```

**Expected outputs:**
- `P8` = Deep sleep (BAD during training)
- `P5` = Light sleep (BAD during training)
- `P2` = Active compute (OK, but not maximum)
- `P0` = Maximum performance (IDEAL)

### Method 2: Monitor Clocks During Training

**Real-time monitoring:**
```powershell
nvidia-smi dmon -s pucvmet -d 1
```

**What to look for:**
```
# pwr: Power consumption (watts)
# gtemp: GPU temperature (celsius)
# sm: Streaming Multiprocessor utilization (%)
# mem: Memory utilization (%)
# enc: Encoder utilization (%)
# dec: Decoder utilization (%)
# mclk: Memory clock (MHz)
# pclk: Graphics/shader clock (MHz)

Example OUTPUT (throttled):
    pwr  gtemp    sm   mem   enc   dec  mclk  pclk
    250     65    85    75     0     0  9501  1800  ← THROTTLED!

Example OUTPUT (unlocked):
    pwr  gtemp    sm   mem   enc   dec  mclk  pclk
    425     72    95    85     0     0  9501  2790  ← FULL SPEED!
```

**Key indicator:** `pclk` (graphics clock) should be near 2700-2790 MHz during training.

### Method 3: Training Speed Comparison

**Simple benchmark:**

1. Start training, note iteration time
2. Apply P-state unlock
3. Restart training, compare iteration time

**Expected difference:**
- Throttled: 45-60 sec/iteration
- Unlocked: 15-20 sec/iteration (depending on config)

---

## Unlock Techniques

### Technique 1: Standard nvidia-smi Lock (Recommended)

**What it does:**
- Forces GPU to maintain performance clocks
- Prevents driver from entering low-power states
- Official NVIDIA tool, safe, stable

**Commands:**
```powershell
# 1. Enable persistence mode (keeps driver loaded)
nvidia-smi -pm 1

# 2. Lock graphics clock (min=max=2790 means constant 2790 MHz)
nvidia-smi -lgc 2790,2790

# 3. Set power limit (adjust for your card)
nvidia-smi -pl 450
```

**Verify:**
```powershell
nvidia-smi -q -d CLOCK
nvidia-smi -q -d POWER
```

**Persistence:**
- Settings reset on reboot
- Must re-apply after each restart
- Consider adding to startup script

### Technique 2: "Defibrillator" Method (Experimental)

**WARNING:** This is the method described in the Manifesto. It's more dramatic but less reliable than nvidia-smi lock.

**Concept:**
1. Launch a game (Cyberpunk, etc.) to force GPU into P-0
2. While game is running at peak, launch a batch script that:
   - Locks clocks using nvidia-smi
   - Briefly disables/enables driver
3. Close game
4. GPU maintains P-0 state thinking game is still running

**Why this exists:**
- Developed before discovering nvidia-smi lock method
- Works on systems where nvidia-smi doesn't have permission
- More complex, higher failure rate

**Recommendation:** Use nvidia-smi method unless you have specific reasons not to.

### Technique 3: NVIDIA Profile Inspector (Windows GUI)

**Alternative for users uncomfortable with command line:**

1. Download NVIDIA Profile Inspector
2. Set "Power Management Mode" → "Prefer Maximum Performance"
3. Set "CUDA - Force P2 State" → "On"

**Effectiveness:** Partial. Helps but doesn't guarantee P0 like nvidia-smi lock.

---

## Verification & Monitoring

### During Training Session:

**Monitor P-state:**
```powershell
# Check every second
while($true) { 
    nvidia-smi -q -d PERFORMANCE | Select-String "Performance State"
    Start-Sleep -Seconds 1
}
```

**Monitor all metrics:**
```powershell
nvidia-smi dmon -s pucvmet -d 1
```

### What "Success" Looks Like:

```
✅ P-state: P0 or P2 (not P5/P8)
✅ pclk: 2700-2790 MHz (not 1800 or below)
✅ Power: 350-450W during compute (not 200-300W)
✅ Temperature: 65-75°C (higher than idle, shows work happening)
✅ Iteration time: 15-20 sec (not 45-60 sec)
```

### Logging for Analysis:

**Create monitoring log:**
```powershell
nvidia-smi dmon -s pucvmet -d 1 > training_monitor.log
```

**Analyze after training:**
```powershell
# Find average clock speed
Get-Content training_monitor.log | Select-String "pclk" | Measure-Object -Average
```

---

## Advanced Troubleshooting

### Issue 1: Lock Doesn't Persist

**Symptom:** GPU returns to P5/P8 shortly after locking.

**Causes:**
1. Background applications triggering power management
2. Driver update reset settings
3. Windows power plan interference

**Solutions:**
```powershell
# 1. Set Windows power plan to High Performance
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c

# 2. Disable Windows "USB selective suspend"
powercfg /change usb-selective-suspend-setting 0

# 3. Re-apply nvidia-smi lock AFTER starting training
# (Some training scripts reset GPU state on initialization)
```

### Issue 2: Temperature Too High After Unlock

**Symptom:** GPU reaches 85-90°C (thermal throttling threshold).

**Causes:**
- Inadequate cooling for sustained 450W load
- Basement setup in Manifesto had unusually good cooling (18-20°C ambient)
- Normal room temp (22-25°C) requires better airflow

**Solutions:**

**Option A: Reduce power limit**
```powershell
nvidia-smi -pl 400  # Instead of 450W
# Impact: ~5% speed reduction, ~10°C temperature reduction
```

**Option B: Improve cooling**
- Open case side panel
- Add additional case fans
- Increase fan curve in MSI Afterburner:
  ```
  60°C → 60% fan speed
  70°C → 80% fan speed
  75°C → 100% fan speed
  ```

**Option C: Undervolt (advanced)**
- Reduce voltage curve in MSI Afterburner
- Maintain 2700 MHz but at lower voltage
- Can reduce temps by 8-12°C with minimal performance impact

### Issue 3: Crashes After Applying Lock

**Symptom:** Training crashes with CUDA errors after P-state unlock.

**Possible causes:**
1. **GPU instability at max clocks:** Not all 4090s reach 2790 MHz stable
2. **Power supply insufficient:** 450W GPU + 250W CPU = 700W, need 850W+ PSU
3. **VRAM instability:** Overclocked memory causing errors

**Solutions:**

**Step 1: Reduce clock target**
```powershell
nvidia-smi -lgc 2700,2700  # Instead of 2790
```

**Step 2: Check PSU capacity**
```powershell
# Monitor total system power draw
# Ensure PSU has 150W+ headroom above peak draw
```

**Step 3: Disable memory overclock**
```powershell
# If you had MSI Afterburner overclocks, remove them
# Test with stock clocks first
```

### Issue 4: "Permission Denied" Errors

**Windows:**
```
Error: Insufficient permissions to modify GPU settings
```

**Solution:**
1. Right-click PowerShell
2. "Run as Administrator"
3. Re-run nvidia-smi commands

**Linux:**
```
Error: NVML: Insufficient Permissions
```

**Solution:**
```bash
sudo nvidia-smi -pm 1
sudo nvidia-smi -lgc 2790,2790
sudo nvidia-smi -pl 450
```

---

## Platform-Specific Notes

### Windows 11

**Known issues:**
- Windows Update can reset nvidia-smi settings
- Game Bar overlay interferes with P-state detection

**Solutions:**
```powershell
# Disable Game Bar
Set-ItemProperty -Path "HKCU:\Software\Microsoft\GameBar" -Name "ShowStartupPanel" -Value 0

# Pause Windows Update during training
# (Settings → Windows Update → Pause for 1 week)
```

### Windows 10

**Generally more stable** for P-state locking than Windows 11.

No specific issues reported.

### Linux (Ubuntu 22.04+)

**Advantages:**
- More reliable P-state persistence
- Better control over power management
- No Game Bar interference

**Differences:**
```bash
# Persistence mode (same as Windows)
sudo nvidia-smi -pm 1

# Lock clocks (same)
sudo nvidia-smi -lgc 2790,2790

# Power limit (same)
sudo nvidia-smi -pl 450

# But: Might need to disable system power management
sudo systemctl mask systemd-sleep.target
```

### Linux (Other Distros)

**Arch/Manjaro:**
- May need to install `nvidia-utils` package
- Same commands as Ubuntu

**Fedora/RHEL:**
- NVIDIA drivers from RPM Fusion
- Consider using `nvidia-settings` GUI as alternative

---

## Summary

**The Core Problem:**
- NVIDIA RTX 40-series GPUs throttle AI training by default
- P5/P8 states limit performance to 25-50% of capability
- Users don't notice because "utilization" appears high

**The Solution:**
- Force P0/P2 state using nvidia-smi lock
- Monitor to ensure lock persists during training
- Adjust power limit if thermal issues arise

**The Result:**
- 3-4x speedup on same hardware
- No cost, just knowledge
- "Unlocking" potential already paid for

**Jensen's "leather jacket" keeping you slow?**  
**Take it off. Run free.** 🔥

---

**For implementation details, see [README.md](README.md)**  
**For the story, see [MANIFESTO.md](MANIFESTO.md)**
