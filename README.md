[中文](./README_zh.md)

# Frida Android Hook Skill

An AI-assisted Skill for performing Frida dynamic instrumentation on Android applications in a Windows PowerShell environment. It provides a complete interactive workflow covering environment checks, frida-server deployment, target selection, hook injection, and cleanup.

## Use Cases

- Inject JavaScript scripts into Android processes for dynamic analysis
- Deploy frida-server to Android devices/emulators via ADB
- Troubleshoot Frida connection issues (port forwarding, stale processes, etc.)
- Set up an Android dynamic instrumentation environment

## Prerequisites

- **Windows** OS + PowerShell
- **Python** environment + `frida` / `frida-tools` (`pip install frida-tools frida`)
- **ADB** (Android Debug Bridge), recommended to add to system PATH
- A rooted (recommended) or non-rooted Android device/emulator
- A `frida-server` binary matching the local Frida version

## Workflow Overview

This Skill breaks down the entire Frida Hook process into five phases, guiding you step by step:

| Phase | Content | Key Operations |
|-------|---------|----------------|
| **Phase 1** | Environment Check & ADB Path Injection | Detect adb & frida availability; connect device; record Frida version |
| **Phase 2** | Device Selection & Server Deployment | Detect CPU architecture; push frida-server; set up port forwarding; start service |
| **Phase 3** | Target Selection & Script Preparation | List device apps; select target process; locate Hook script; choose injection mode |
| **Phase 4** | Execute Hook Injection | Inject script via Spawn or Attach mode; view real-time output |
| **Phase 5** | Stop & Cleanup | Terminate injection; clean up device-side processes and local port forwarding |

## Key Features

### Interactive Experience

- **Step-by-step guidance**: Only asks for the minimum information needed at each step — no information overload
- **Operation confirmation**: All device-modifying operations (push, chmod, kill) require explicit confirmation before execution
- **Number-based quick select**: Device lists, process lists, and script lists are displayed in `[#] Name` format — reply with a number to select

### Environment Guarantees

- Auto-detects and guides configuration of temporary ADB environment variables
- Auto-detects device CPU architecture and matches the correct `frida-server` binary
- Cleans up stale processes and port forwarding before startup to ensure a clean environment
- Multi-dimensional frida-server status verification (process, port, forwarding, connectivity)

### PowerShell-Specific Adaptations

- Uses `Start-Process -WindowStyle Hidden` for non-blocking frida-server startup, preventing terminal hangs
- All local commands use PowerShell-compatible syntax
- Supports `adb kill-server; adb start-server` to reset the ADB service

## Triggers

Use the following trigger words in conversation to automatically load this Skill:

`frida` `hook` `插桩` `注入` `frida-server` `adb push` `frida-ps` `安卓动态插桩`

## Injection Modes

| Mode | Command | Description |
|------|---------|-------------|
| **Spawn** | `frida -U -f <pkg> -l "$script_path"` | Cold-start app and inject — ideal for hooking initialization code |
| **Attach** | `frida -U -n <pkg> -l "$script_path"` | Attach to a running process — suitable for runtime debugging |

Press `q`, `exit`, or `Ctrl+C` to stop the hook and enter the cleanup flow.

## Quick Command Reference

| Operation | PowerShell Command |
|-----------|-------------------|
| Temporarily add ADB to PATH | `$env:Path += ";C:\path\to\platform-tools"` |
| Check local ADB | `Get-Command adb` |
| Connect to emulator | `adb connect 127.0.0.1:5555` |
| Restart ADB service | `adb kill-server; adb start-server` |
| List devices | `adb devices` |
| Detect CPU architecture | `adb -s <sn> shell getprop ro.product.cpu.abi` |
| Push frida-server | `adb -s <sn> push "$path" /data/local/tmp/frida-server` |
| Grant execute permission | `adb -s <sn> shell chmod 777 /data/local/tmp/frida-server` |
| Set up port forwarding | `adb -s <sn> forward tcp:27042 tcp:27042` |
| List app processes | `frida-ps -Uai` |
| Check server process | `adb -s <sn> shell "ps -A \| grep frida"` |
| Kill server process | `adb -s <sn> shell "killall frida-server"` |
| Remove port forwarding | `adb -s <sn> forward --remove tcp:27042` |

## Project Structure

```
frida-android-hook/
├── SKILL.md       # Skill definition & complete workflow specification
└── README.md      # This file
```
