# Frida Android Hook Skill

Windows PowerShell 环境下对 Android 应用进行 Frida 动态插桩的 AI 辅助 Skill，提供从环境检查、frida-server 部署、目标选择到 Hook 注入与清理的完整交互式工作流。

## 适用场景

- 向 Android 进程注入 JavaScript 脚本进行动态分析
- 通过 ADB 部署 frida-server 到 Android 设备/模拟器
- 排查 Frida 连接问题（端口转发、进程残留等）
- 搭建 Android 动态插桩调试环境

## 前置环境

- **Windows** 操作系统 + PowerShell
- **Python** 环境 + `frida` / `frida-tools`（`pip install frida-tools frida`）
- **ADB**（Android Debug Bridge），建议加入系统环境变量
- 一台已 Root（推荐）或非 Root 的 Android 设备/模拟器
- 与本地 Frida 版本匹配的 `frida-server` 二进制文件

## 工作流概览

本 Skill 将整个 Frida Hook 流程拆分为五个阶段，逐步引导用户完成：

| 阶段 | 内容 | 关键操作 |
|------|------|----------|
| **阶段一** | 环境检查与 ADB 路径注入 | 检测 adb、frida 可用性；连接设备；记录 Frida 版本 |
| **阶段二** | 设备选择与 Server 部署 | 检测 CPU 架构；推送 frida-server；建立端口转发；启动服务 |
| **阶段三** | 目标选择与脚本准备 | 列出设备应用；选择目标进程；定位 Hook 脚本；确定注入模式 |
| **阶段四** | 执行 Hook 注入 | 以 Spawn 或 Attach 模式注入脚本；实时查看输出 |
| **阶段五** | 停止与清理 | 终止注入；清理手机端进程与本地端口转发 |

## 核心特性

### 交互体验

- **分步引导**：每次只问当前步骤所需的最少信息，拒绝信息轰炸
- **操作确认制**：所有修改设备状态的操作（推送、提权、杀进程）均需确认后执行
- **编号快速选择**：设备列表、进程列表、脚本列表均以 `[编号] 名称` 格式展示，回复数字即可选择

### 环境保障

- 自动检测并引导配置 ADB 临时环境变量
- 自动检测设备 CPU 架构，匹配正确的 `frida-server` 文件
- 启动前自动清理残留进程和端口转发，确保环境干净
- 多维度验证 frida-server 运行状态（进程、端口、转发、联通性）

### PowerShell 专属适配

- 使用 `Start-Process -WindowStyle Hidden` 非阻塞启动 frida-server，避免终端假死
- 所有本地命令使用 PowerShell 兼容语法
- 支持 `adb kill-server; adb start-server` 重置 ADB 服务

## 触发方式

在对话中使用以下触发词即可自动加载本 Skill：

`frida` `hook` `插桩` `注入` `frida-server` `adb push` `frida-ps` `安卓动态插桩`

## 注入模式

| 模式 | 命令 | 说明 |
|------|------|------|
| **Spawn** | `frida -U -f <包名> -l "$script_path"` | 冷启动应用并注入，适合 Hook 启动初始化代码 |
| **Attach** | `frida -U -n <包名> -l "$script_path"` | 挂载到已运行的进程，适合运行时动态调试 |

运行中可按 `q`、`exit` 或 `Ctrl+C` 终止 Hook 并进入清理流程。

## 常用命令速查

| 操作 | PowerShell 命令 |
|------|----------------|
| 临时添加 ADB 环境变量 | `$env:Path += ";C:\path\to\platform-tools"` |
| 检测本地 ADB | `Get-Command adb` |
| 连接模拟器 | `adb connect 127.0.0.1:5555` |
| 重启 ADB 服务 | `adb kill-server; adb start-server` |
| 查看设备 | `adb devices` |
| 检测 CPU 架构 | `adb -s <sn> shell getprop ro.product.cpu.abi` |
| 推送 frida-server | `adb -s <sn> push "$path" /data/local/tmp/frida-server` |
| 赋予执行权限 | `adb -s <sn> shell chmod 777 /data/local/tmp/frida-server` |
| 建立端口转发 | `adb -s <sn> forward tcp:27042 tcp:27042` |
| 列出应用进程 | `frida-ps -Uai` |
| 检查 server 进程 | `adb -s <sn> shell "ps -A \| grep frida"` |
| 清理 server 进程 | `adb -s <sn> shell "killall frida-server"` |
| 移除端口转发 | `adb -s <sn> forward --remove tcp:27042` |

## 项目结构

```
frida-android-hook/
├── SKILL.md       # Skill 定义与完整工作流规范
└── README.md      # 本文件
```
