# Android Optimizer

ADB 安卓设备优化 skill — 省电、提速、去预装、救砖，一套走完。

适用于 AI Agent（Claude / OpenCode 等）交互式操作，也适合手动 ADB 用户参考。

## Features

- **省电优化** — CPU 调度、屏幕超时、动画、蓝牙/WiFi 扫描、自动同步
- **禁用预装** — MIUI / ColorOS / OneUI 三体系安全可禁清单
- **后台限制** — 淘宝、抖音、小红书等 15+ 常用 app 的后台跑电拦截
- **存储清理** — 大文件定位、缓存清理、APK 批量删除
- **Root 专属** — CPU governor 调参、GPU powersave、Magisk 模块持久化
- **救砖恢复** — Magisk Safe Mode、TWRP 删模块、fastboot 刷回、factory reset 全路径

## Install

### As AI Agent Skill

将 `android-optimizer.skill` 放入你的 agent skills 目录：

```bash
# Claude
cp android-optimizer.skill ~/.claude/skills/

# OpenCode
cp android-optimizer.skill ~/.opencode/skills/
```

### As Standalone Reference

纯文档浏览，直接打开 `SKILL.md`。

## Usage

### Quick Start

```bash
# 1. Connect device
adb devices -l

# 2. Diagnostics
adb shell "dumpsys battery | grep -E 'level|temperature'"
adb shell "df -h /data | tail -1"
adb shell "settings get global low_power"
adb shell "pm list packages -3 | sort"

# 3. Disable bloat
adb shell pm disable-user --user 0 com.miui.compass

# 4. Restrict background
adb shell cmd appops set com.taobao.taobao RUN_ANY_IN_BACKGROUND deny

# 5. Verify
adb shell "pm list packages -u | grep ':disabled'"
adb shell "cmd appops query-op RUN_ANY_IN_BACKGROUND reject"
```

详见 `SKILL.md` 完整工作流。

## Safety Principles

1. **从不禁用关键系统包** — SystemUI、Launcher、Google Play Services、Phone、Settings
2. **`pm disable-user` 优先** — 可逆（`pm enable`），`pm uninstall` 不可逆
3. **改前必提醒** — 根操作前告知救砖方案
4. **不动 `/system`** — 无完全备份不做
5. **留退路** — 改前至少备份 `boot.img`

## What Works on Different Devices

| Device / ROM | Non-Root | Root |
|---|---|---|
| LineageOS / AOSP | ✅ Full | ✅ Full + CPU/GPU |
| MIUI / HyperOS 14- | ✅ Most (settings put works) | ✅ Full |
| HyperOS 15+ | ⚠️ pm + appops only (settings blocked) | ✅ Full (via su) |
| ColorOS 16 | ⚠️ pm + appops only | ✅ Full |
| One UI (Samsung) | ✅ Most | ✅ Full |

## Brick Recovery Quick Command

```bash
# Magisk Safe Mode: Power off → hold Vol Down + Power → release Power at logo
# TWRP delete module: mount /data → rm -rf /data/adb/modules/<bad-module>
# Fastboot unbrick: fastboot flash boot stock-boot.img
# ADB backup before tweak: adb shell su -c "dd if=/dev/block/bootdevice/by-name/boot of=/sdcard/stock-boot.img" && adb pull /sdcard/stock-boot.img
```

## License

MIT
