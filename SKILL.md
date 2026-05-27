---
name: android-optimizer
description: >
  Optimize Android devices for battery life, speed, and storage.
  Trigger when user connects an Android device via ADB and asks to
  optimize, save battery, speed up, clean junk, disable bloatware,
  restrict background apps, or apply performance tweaks.
  Also trigger when user mentions "ADB optimization", "android battery
  saver", "disable bloatware", "安卓省电", "安卓优化", "流畅".
  Safe for non-root devices; offers root-only tweaks as optional.
---

# Android Optimizer

Safety-first ADB optimization for Android devices. Every operation must be revertible.

## Safety Principles

1. **Never disable critical packages**: `android`, `com.android.systemui`, `com.android.settings`, `com.google.android.gms`, `com.android.phone`, `com.android.launcher3` (or manufacturer launcher), `com.android.packageinstaller`, `com.android.providers.settings`, `com.android.vending`
2. **Prefer `pm disable-user` over `pm uninstall`**: disable-user is revertible (`pm enable`), uninstall is not without a full reset.
3. **Root is NOT required**: Non-root devices can still benefit from `pm disable-user`, `cmd appops`, safe `settings put` calls.
4. **Test one change at a time**: Batch is efficient, but isolate if user hits issues.
5. **Never modify `/system`**: Without root and full backup, modifying system partitions risks bricking.
6. **Backup before aggressive actions**: If user wants `pm uninstall` or root tweaks, warn them first.
7. **⚠️ NEVER delete packages.xml**: Deleting `/data/system/packages.xml` bricks the device (stuck at boot animation). The package manager cannot write to CE-encrypted storage without user unlock. If accidentally deleted:
   - Boot to recovery
   - Mount data partition
   - If `.bak` exists, restore it
   - If no backup, you must `adb root` and create a minimal packages.xml, or dirty flash the ROM
   - Full boot + user unlock may be needed for the package manager to write the CE storage

## Environment Setup

Before any optimization, ensure ADB is available and device is connected.

### 1. Install ADB

If `adb` not found, install platform tools for current OS:

```bash
# macOS
if ! command -v adb &>/dev/null; then
  if command -v brew &>/dev/null; then
    brew install --cask android-platform-tools
  else
    echo "Install Homebrew first: https://brew.sh"
    exit 1
  fi
fi

# Linux (Debian/Ubuntu)
if ! command -v adb &>/dev/null; then
  sudo apt install -y adb fastboot
fi

# Linux (Arch)
if ! command -v adb &>/dev/null; then
  sudo pacman -S android-tools
fi

# Windows (via winget)
if ! command -v adb &>/dev/null; then
  winget install Google.AndroidPlatformTools
fi
```

### 2. Verify Installation

```bash
adb --version
```

### 3. Enable ADB on Device

User must do this manually on the phone:

```
Settings → About Phone → Tap "Build Number" 7x → Developer options
Settings → Developer options → USB Debugging → ON
Settings → Developer options → Stay Awake → ON (optional)
Settings → Developer options → Disable ADB Authorization Timeout → ON
```

### 4. Connect Device

**USB connection:**

```bash
adb devices -l
# Expected: <serial> device <model>
# If unauthorized → check phone screen → allow USB debugging → check "Always allow"
```

**WiFi connection (no USB cable):**

```bash
# After USB connection is established, switch to TCP:
adb tcpip 5555
adb connect <device-ip>:5555
# Now disconnect USB — device stays connected on LAN
# Check: adb devices -l (should show <ip>:5555 device)
```

**WiFi pairing (Android 11+, no cable at all):**

```bash
# On phone: Developer options → Wireless debugging → enable → Pair device with pairing code
# This gives you IP:PORT and pairing code
adb pair <ip>:<pairing-port> <code>
adb connect <ip>:<connect-port>
```

### 5. Test Connection

```bash
adb shell echo "connected"
adb shell getprop ro.product.model
```

If any step fails, stop and ask user to fix the connection before proceeding.

## Quick Reference: What Works Where

| Operation | Root | Non-Root (ADB) | Notes |
|-----------|------|-----------------|-------|
| `pm disable-user` | ✅ | ✅ | Works on all devices (use `--user 0` flag) |
| `pm disable` (without --user) | ✅ | ❌ | Fails on Android 9+ with SecurityException |
| `cmd appops set` | ✅ | ✅ | Runtime permission control |
| `settings put global` | ✅ | ❌ | Blocked on Android 13+ (needs WRITE_SECURE_SETTINGS) |
| `settings put system` | ✅ | ❌ | Blocked on HyperOS 15+, Android 15+ |
| CPU governor tuning | ✅ | ❌ | Via sysfs, needs root/su |
| GPU governor | ✅ | ❌ | Via sysfs, needs root/su |
| Magisk module install | ✅ | ❌ | Requires Magisk root |
| `pm uninstall -k --user 0` | ✅ | ⚠️ | Works on some ROMs, risky without backup |
| `adb root` | ✅ | ❌ | Only works on userdebug/user builds (LineageOS, crDroid). Stock ROMs deny. |
| `cmd package reconcile` | ✅ | ❌ | Not available on Android 15+ |

## Workflow

### Step 1: Diagnostics

Run these to understand device state:

```bash
adb shell "getprop ro.build.display.id"   # ROM version
adb shell "getprop ro.build.version.sdk"  # SDK level
adb shell "dumpsys battery | grep -E 'level|temperature'"  # Battery
adb shell df -h /data | tail -1          # Storage
adb shell "settings get system screen_off_timeout"  # Screen timeout
adb shell "settings get global window_animation_scale"  # Animations
adb shell "settings get global low_power"  # Battery saver
adb shell "cmd deviceidle get deep"       # Doze mode
adb shell "pm list packages -3 | sort"    # Installed 3rd-party apps
```

Also check if rooted: `adb shell su -c "id" 2>&1 || echo "no root"`

### Step 2: System Settings

Non-root (try each, some may fail on newer ROMs):

```bash
# Screen - brightness auto, timeout 60s
adb shell settings put system screen_brightness_mode 1
adb shell settings put system screen_off_timeout 60000
adb shell settings put global screen_off_timeout 60000

# Animations - 0.5x for snappier feel
adb shell settings put global window_animation_scale 0.5
adb shell settings put global transition_animation_scale 0.5
adb shell settings put global animator_duration_scale 0.5

# Power saving
adb shell settings put global low_power 1
adb shell settings put global adaptive_battery_management_enabled 1
adb shell settings put global app_standby_enabled 1

# Radios - disable scanners
adb shell settings put global ble_scan_always_enabled 0
adb shell settings put global wifi_scan_always_available 0
adb shell settings put global nfc_on 0
adb shell settings put global auto_sync 0
adb shell settings put global mobile_data_always_on 0
```

Root-only (via `su -c`):

```bash
adb shell su -c "
  settings put global window_animation_scale 0.5
  settings put global transition_animation_scale 0.5
  settings put global animator_duration_scale 0.5
  settings put global low_power 1
  settings put global adaptive_battery_management_enabled 1
  settings put global ble_scan_always_enabled 0
  settings put global wifi_scan_always_available 0
  settings put global nfc_on 0
  settings put global auto_sync 0
  settings put global mobile_data_always_on 0
"
```

### Step 3: Disable Bloatware

Always use `pm disable-user --user 0`. This is safe and revertible.

**Universal safe-to-disable** (these have no system-wide impact):

```
com.miui.compass               # MIUI compass
com.miui.cleanmaster           # MIUI cleaner
com.miui.virtualsim            # MIUI virtual SIM
com.miui.huanji                # MIUI phone clone
com.miui.newhome               # MIUI new home
com.miui.newmidrive            # MIUI cloud drive
com.miui.voiceassistProxy      # MIUI voice assistant
com.miui.findmy                # MIUI find device
com.miui.video                 # MIUI video player
com.mfashiongallery.emag       # MIUI wallpaper carousel
com.xiaomi.shop                # Mi Store
com.xiaomi.vipaccount          # Mi VIP
com.xiaomi.mibrain.speech      # XiaoAi speech
com.miui.miservice             # MIUI service feedback
com.duokan.phone.remotecontroller  # IR remote
com.miui.notes                 # If user doesn't use notes
com.miui.weather2              # If user uses other weather app
com.miui.screenrecorder        # If not needed
com.miui.themestore            # Theme store
com.miui.securitymanager       # MIUI security - careful, may affect permissions
```

**ColorOS / OnePlus:**

```
com.heytap.market             # App market
com.heytap.cloud              # Cloud
com.heytap.pictorial          # Lock screen magazine
com.coloros.oshare            # OShare
com.coloros.floatassistant    # Floating assistant
com.coloros.gamespace         # Game space
com.coloros.smartdrive        # Smart driving
com.coloros.video             # Video
com.coloros.music             # Music
com.coloros.gallery           # Gallery (if replaced)
com.coloros.filemanager       # Files (if replaced)
com.coloros.browser           # Browser
com.coloros.yoli              # Yoli
com.heytap.gamecenter         # Game center
com.coloros.childrenspace     # Children space
com.coloros.healthservice     # Health
com.coloros.weather.service   # Weather
```

**Samsung One UI (safe ones):**

```
com.samsung.android.bixby.wakeup
com.samsung.android.bixby.agent
com.samsung.android.bixbyvision.framework
com.samsung.android.aremoji
com.samsung.android.ardrawing
com.samsung.android.arzone
com.samsung.android.game.gametools
com.samsung.android.game.gos
com.samsung.android.app.vrsetup
com.samsung.android.hmt.vrshell
com.facebook.appmanager
com.facebook.katana
com.facebook.orca
com.microsoft.skydrive
com.samsung.android.widgetapp.yahooedge
```

**After disabling, verify stability with:**

```bash
adb shell dumpsys package <package> | grep disabled
```

### Step 4: Restrict Background Apps

Target heavy apps that don't need to run in background (shopping, social, streaming, games):

```bash
# Syntax
cmd appops set <package> RUN_ANY_IN_BACKGROUND deny

# Common targets
cmd appops set com.taobao.taobao RUN_ANY_IN_BACKGROUND deny
cmd appops set com.xunmeng.pinduoduo RUN_ANY_IN_BACKGROUND deny
cmd appops set com.ss.android.ugc.aweme RUN_ANY_IN_BACKGROUND deny  # TikTok/Douyin
cmd appops set com.xingin.xhs RUN_ANY_IN_BACKGROUND deny             # Xiaohongshu
cmd appops set tv.danmaku.bili RUN_ANY_IN_BACKGROUND deny            # Bilibili
cmd appops set com.tencent.qqmusic RUN_ANY_IN_BACKGROUND deny
cmd appops set com.baidu.searchbox RUN_ANY_IN_BACKGROUND deny
cmd appops set com.eg.android.AlipayGphone RUN_ANY_IN_BACKGROUND deny
cmd appops set com.sankuai.meituan RUN_ANY_IN_BACKGROUND deny
cmd appops set com.zhihu.android RUN_ANY_IN_BACKGROUND deny
cmd appops set com.autonavi.minimap RUN_ANY_IN_BACKGROUND deny       # Amap
cmd appops set com.handsgo.jiakao.android RUN_ANY_IN_BACKGROUND deny
cmd appops set com.duolingo RUN_ANY_IN_BACKGROUND deny
cmd appops set cn.wps.moffice_eng RUN_ANY_IN_BACKGROUND deny

# To revert:
cmd appops set <package> RUN_ANY_IN_BACKGROUND allow
```

**DO NOT restrict** these (they need background for core function): `com.android.systemui`, launcher, keyboard (`com.baidu.input_mi`, etc.), `com.android.phone`, calendar sync adapters.

### Step 5: Storage Cleanup

```bash
# Find large files
adb shell find /sdcard -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# Find cache-heavy apps
adb shell du -sh /data/data/*/cache 2>/dev/null | sort -rh | head -10

# Clear caches safely
adb shell pm trim-caches 10G  # Ask system to free cache space

# Common junk targets
adb shell rm -rf /sdcard/Download/*.apk /sdcard/Android/data/*/cache/*
adb shell rm -rf /sdcard/tencent/*  # QQ/WeChat temp files (if safe)

# Check what can be removed
adb shell du -sh /sdcard/Download /sdcard/DCIM/.thumbnails /sdcard/Android/data/*/cache
```

### Step 6: Reading Device (E-ink) Optimizations

For e-ink devices (Hisense A5/A5 Pro/A7, Boox, etc.) used primarily for reading:

```bash
# No animations needed on e-ink
adb shell settings put global window_animation_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global animator_duration_scale 0

# Longer screen timeout for reading
adb shell settings put system screen_off_timeout 600000  # 10 min

# Reading comfort
adb shell settings put system font_scale 1.15
adb shell settings put system haptic_feedback_enabled 0
adb shell settings put system sound_effects_enabled 0
adb shell settings put system notification_light_pulse 0

# Radios not needed for reading
adb shell settings put secure location_providers_allowed -gps -network -wifi
adb shell settings put global ble_scan_always_enabled 0
adb shell settings put global wifi_scan_always_available 0
adb shell settings put global auto_sync 0
adb shell svc wifi disable 2>/dev/null
adb shell svc bluetooth disable 2>/dev/null

# Disable reading-unnecessary system apps
adb shell pm disable-user --user 0 com.hisense.hiphone.appstore 2>/dev/null
adb shell pm disable-user --user 0 com.hmct.music 2>/dev/null
adb shell pm disable-user --user 0 com.hmct.updater 2>/dev/null
adb shell pm disable-user --user 0 com.hmct.gamemodefloatwindow 2>/dev/null
adb shell pm disable-user --user 0 com.hisense.hiphone.payment 2>/dev/null
```

**Retain** these for reading quality: reading apps (微信读书, 掌阅, 多看), file managers, dictionary apps.

Note: On Hisense A5 Pro (Android 9), the shell is very limited — no `head`, `wc`, `sed`, `awk`, `cut`. Use basic shell commands only.

Warning: `pm disable` (without `--user 0`) fails on Android 9+ with `SecurityException: Shell cannot change component state`. Always use `pm disable-user --user 0`.

### Step 7: Root-Only Tweaks

Only execute these if device is rooted and user explicitly requests it.

**CPU Governor (conservative or schedutil are good for battery):**

```bash
adb shell su -c "
  for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo conservative > \$cpu
  done
  # Conservative tuning
  echo 70 > /sys/devices/system/cpu/cpufreq/conservative/up_threshold
  echo 30 > /sys/devices/system/cpu/cpufreq/conservative/down_threshold
  echo 8 > /sys/devices/system/cpu/cpufreq/conservative/freq_step
  echo 2 > /sys/devices/system/cpu/cpufreq/conservative/sampling_down_factor
"
```

**GPU Governor (powersave for battery, performance for speed):**

```bash
adb shell su -c "
  [ -f /sys/class/kgsl/kgsl-3d0/devfreq/governor ] && echo powersave > /sys/class/kgsl/kgsl-3d0/devfreq/governor
  [ -f /sys/class/kgsl/kgsl-3d0/min_clock_mhz ] && echo 100 > /sys/class/kgsl/kgsl-3d0/min_clock_mhz
"
```

**Persist via Magisk module:**

Create `service.sh` in Magisk module:

```bash
#!/system/bin/sh

# CPU governor + tuning
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
  echo conservative > "$cpu" 2>/dev/null
done

[ -f /sys/devices/system/cpu/cpufreq/conservative/up_threshold ] && echo 70 > /sys/devices/system/cpu/cpufreq/conservative/up_threshold
[ -f /sys/devices/system/cpu/cpufreq/conservative/down_threshold ] && echo 30 > /sys/devices/system/cpu/cpufreq/conservative/down_threshold
[ -f /sys/devices/system/cpu/cpufreq/conservative/freq_step ] && echo 8 > /sys/devices/system/cpu/cpufreq/conservative/freq_step
[ -f /sys/devices/system/cpu/cpufreq/conservative/sampling_down_factor ] && echo 2 > /sys/devices/system/cpu/cpufreq/conservative/sampling_down_factor

# GPU powersave
[ -f /sys/class/kgsl/kgsl-3d0/devfreq/governor ] && echo powersave > /sys/class/kgsl/kgsl-3d0/devfreq/governor

# Display always-off battery saving
echo Y > /sys/module/lpm_levels/parameters/sleep_disabled 2>/dev/null
```

### Step 7: Flash ROM via Recovery Sideload

Recovery sideload is useful for installing custom ROMs, Gapps, or Magisk when Direct Install fails.

```bash
# 1. Reboot to recovery
adb reboot bootloader    # First to fastboot
fastboot boot recovery.img  # Boot any recovery image temporarily (doesn't flash)
# OR
adb reboot recovery     # If device is running and has a recovery

# 2. On device: navigate to "Apply update" → "Apply from ADB"
# 3. Send the ROM/Gapps/Magisk zip:
adb sideload /path/to/package.zip

# 4. Repeat for additional packages (Gapps, Magisk)
# 5. On device: select "Reboot system now"
```

Notes:
- `Total xfer: 0.98x` or `0.99x` is normal and means success.
- Magisk APK can be renamed to `.zip` and flashed directly.
- LineageOS recovery works for crDroid, LineageOS, and other AOSP-based ROMs.
- For `pm disable`-related bricks: boot recovery → mount data → `rm -rf /data/system/packages.xml.bak` or restore from backup.

### Step 8: Recovery & Unbrick (救砖)

**Warn user before any root tweak**: "If the device won't boot after this, here is the recovery plan."

#### 7.1 Enter Recovery Mode

| Device | Key Combo |
|--------|-----------|
| Xiaomi / Redmi | Power + Volume Up (MIUI/HyperOS fastboot → recovery via menu) |
| OnePlus / OPPO | Power + Volume Down |
| Samsung | Power + Bixby + Volume Up (USB-C connected to PC) |
| Google Pixel | Power + Volume Down → recovery mode menu |
| LineageOS / Custom ROM | Often same as stock, or `adb reboot recovery` |

Via ADB (if bootlooped but ADB debug enabled):

```bash
adb reboot recovery
adb reboot bootloader   # Fastboot mode
```

#### 7.2 Magisk Safe Mode

If a Magisk module causes bootloop, Magisk has a built-in safe mode:

```bash
# Method 1: Boot with volume-down held
# Turn device off → hold Volume Down + Power → release Power when screen turns on
# Keep holding Volume Down until boot completes → Magisk Safe Mode disables ALL modules
```

In Magisk Safe Mode:
- All modules are temporarily disabled
- Device should boot normally (slower, no module effects)
- Open Magisk app → Modules tab → uninstall the bad module → reboot

#### 7.3 Disable Modules via Recovery (TWRP / OrangeFox)

If Magisk Safe Mode doesn't work (e.g., bootloop before Magisk init):

```bash
# Reboot to custom recovery
adb reboot recovery

# In TWRP → Advanced → File Manager → navigate to:
#   /data/adb/modules/
# Each subfolder = one module. Delete or rename the bad one:
#   mv <module-name> <module-name>.disabled

# Via ADB in recovery:
adb shell
# In TWRP shell:
mount /data
cd /data/adb/modules
ls                    # List all modules
mv bad-module bad-module.bak  # Disable by renaming
reboot
```

#### 7.4 Disable Magisk Module via ADB (if device boots but is unstable)

```bash
adb shell
su
cd /data/adb/modules
ls
touch <bad-module>/disable  # Create "disable" flag file
reboot
```

#### 7.5 Remove Root Entirely (Unbrick)

If root itself caused the problem (unsupported device, wrong boot image):

```bash
# 1. Download stock firmware for exact model
# 2. Extract boot.img
# 3. Flash via fastboot:
adb reboot bootloader
fastboot flash boot stock-boot.img
fastboot reboot

# If device can't even boot to bootloader:
# - Xiaomi: EDL mode (requires authorized account)
# - Samsung: Download mode (Power + Volume Down)
# - Other: Recovery → factory reset → then flash stock via fastboot/ODIN
```

#### 7.6 Last Resort — Factory Reset via Recovery

```bash
adb reboot recovery
# In recovery: Wipe data/factory reset
# WARNING: This erases ALL user data
```

#### 7.7 Prevention — Backup Before Root Tweaks

```bash
# Backup current boot.img (can restore if root fails)
adb shell su -c "dd if=/dev/block/bootdevice/by-name/boot of=/sdcard/stock-boot.img"
adb pull /sdcard/stock-boot.img

# Or full backup via TWRP:
# In TWRP → Backup → select Boot + System + Data → Swipe to backup
```

### Step 8: Verify

Check what changed:

```bash
echo "=== Settings ==="
adb shell settings get global window_animation_scale
adb shell settings get global low_power
adb shell settings get global wifi_scan_always_available
adb shell settings get global nfc_on

echo "=== Disabled packages ==="
adb shell "pm list packages -u | grep ':disabled' | cut -d: -f2"

echo "=== Restricted background apps ==="
adb shell "cmd appops query-op RUN_ANY_IN_BACKGROUND reject"

echo "=== Battery ==="
adb shell dumpsys battery | grep -E 'level|temperature|powered'
```

## User's Manual Changes (if ADB can't write settings)

On HyperOS 15+ and Android 15+, ADB `settings put` is blocked. Tell user to set manually:

1. **开发者选项** → 动画缩放 1x → 0.5x
2. **显示** → 自动亮度开启
3. **设置** → 省电模式/电池优化开启
4. **设置** → 屏幕超时调至合适时长

## Revert All

```bash
# Re-enable all disabled packages
adb shell "pm list packages -u | grep ':disabled' | cut -d: -f2" | while read pkg; do
  adb shell pm enable --user 0 "$pkg"
done

# Restore background
adb shell "cmd appops query-op RUN_ANY_IN_BACKGROUND reject | tail -n +2" | while read line; do
  pkg=$(echo $line | awk '{print $1}')
  adb shell cmd appops set $pkg RUN_ANY_IN_BACKGROUND allow
done

# Restore settings defaults
adb shell settings put global window_animation_scale 1
adb shell settings put global transition_animation_scale 1
adb shell settings put global animator_duration_scale 1
adb shell settings put global low_power 0
adb shell settings put global wifi_scan_always_available 1
adb shell settings put global nfc_on 1
```

## Android Version Quirks

| Version | ADB | Root | Notes |
|---------|-----|------|-------|
| **Android 5.x** (SDK 21-22) | ✅ Full | ✅ SuperSU | Very limited shell (no `head`, `wc`, `sed`, `awk`, `cut`). Intel Atom on some devices. Use basic commands only. |
| **Android 9** (SDK 28) | ✅ Most | ⚠️ | `pm disable` without `--user 0` fails. Shell has basic toolbox. |
| **Android 13+** (SDK 33+) | ⚠️ | ✅ | `settings put global/system` blocked from shell. Need `su -c` or user manual. |
| **Android 15+** (SDK 35+) | ⚠️ | ✅ | `cmd package reconcile` removed. FBE (File-Based Encryption) locks CE storage. `adb root` only on userdebug builds. |
| **HyperOS 15+** | ⚠️ | ✅ | Same as Android 15+ but stricter SELinux. Even root may have limitations. |

## FBE (File-Based Encryption) on Android 15+

On Android 15+ with FBE:
- `/data` has two encryption layers: **DE** (Device Encrypted, available after boot) and **CE** (Credential Encrypted, requires user unlock)
- **DE**: `/data/system_de/0/`, `/data_mirror/data_de/` — accessible before user unlocks
- **CE**: `/data/system_ce/0/`, `/data/user/0/`, `/data/app/` — locked until user enters PIN/password
- In recovery, CE appears as encrypted filenames or empty directories
- `df` shows data used on `/data`, but `ls /data/system/` returns empty due to CE lock
- Only `adb root` (on userdebug builds) or `su` (with root) can bypass CE restrictions

**If packages.xml is missing and CE is locked**: Device stays in boot animation forever. The package manager scans APKs but can't write to CE storage to save the database. Fix:
1. Boot normally, `adb root` → restore from `.bak` → reboot
2. Or dirty flash the ROM (replaces system, preserves data)
3. If no backup, you need to create a minimal packages.xml or flash a fresh ROM

## Device-Specific Known Devices

| Device | OS | Shell | Root | Notes |
|--------|----|-------|------|-------|
| **Hisense A5 Pro** (HLTE202N) | Android 9 | Limited | No | E-ink reading phone. Use `pm disable-user --user 0`. 4GB RAM. No `sed/awk`. |
| **Mi Mix 2** (chiron) | Android 15-16 | Full | ✅ Magisk | SD835. Great custom ROM support (LineageOS, crDroid). Userdebug builds support `adb root`. |
| **Mi Pad 2** (latte) | Android 5.1 | Very limited | ✅ SuperSU | Intel Atom x86. Very few shell tools. No `sed/awk/head/wc/cut`. |
| **Redmi Note 13 Pro** (24069RA21C) | Android 16 HyperOS 2 | Full | No | `settings put` blocked. Needs bootloader unlock for root. |
