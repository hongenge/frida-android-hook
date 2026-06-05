---
name: frida-android-hook
description: 当用户在 Windows PowerShell 环境下需要对 Android 应用进行 Frida Hook、向 Android 进程注入 JS 脚本、通过 ADB 部署 frida-server、排查 Frida 连接问题、或搭建 Android 动态插桩环境时使用。触发词：frida, hook, 插桩, 注入, frida-server, adb push, frida-ps, 安卓动态插桩。
---

# Frida Android Hook (Windows PowerShell 专属)

  ## 1. 交互核心规则 (Core Rules)

  1. **分步推进，拒绝信息轰炸**：严格按照阶段（Phase 1 至 Phase 5）推进。每次交互**只询问当前步骤所需的最少必要信息**，绝不一次性抛出所有问题。
  2. **操作确认制**：凡涉及修改设备状态的操作（如 `adb push`、设备侧 `chmod`、`killall`），必须在执行前明确告知用户并获得确认。
  3. **紧凑列表展示**：列出设备、进程，必须使用 `[编号] 名称 (包名)` 的紧凑格式，引导用户通过直接回复数字（如 `1`）进行快速选择。
  4. **PowerShell 独占兼容**：所有本地执行的命令必须使用 PowerShell 兼容语法，严禁包含 Bash 专有的 `which`、后台符 `&` 以及 `2>/dev/null` 等重定向符号（注意：`adb shell` 内部作为字符串传递给 Android 系统的 Linux 命令除外）。如果用户在本地 ADB 所在目录下执行命令，提示其使用 `.\adb`。

  ## 2. 详细执行工作流 (Workflow)

  ### 阶段一：环境检查与 ADB 路径注入 (Environment & ADB Check)

  引导用户在本地 PowerShell 终端运行以下步骤，确保基础环境就绪：

  1. **检查本地 adb 可用性**:

     - **检测方式**：在 PowerShell 中尝试运行以下命令：

       ```
       Get-Command adb -ErrorAction SilentlyContinue
       ```

     - **若未找到 adb 命令**（返回空或报错 "无法将 adb 项识别为..."）：

       - **交互引导**：主动询问用户：“未在您的 Windows 系统中检测到 `adb` 环境变量。请提供您本地的 ADB（或 Android SDK `platform-tools`）安装路径。”

       - **生成临时环境**：根据用户提供的路径（例如 `D:\Android\platform-tools\`），指导用户在**当前 PowerShell 窗口**中执行以下命令追加临时环境变量：

         ```
         $env:Path += ";D:\Android\platform-tools"
         ```

       - **验证临时环境**：引导用户在同一窗口内运行 `adb version` 验证。

  2. **检查 ADB 设备连接并尝试激活**:

     ```
     adb devices
     ```

     - **情况 A：设备列表为空**：

       - **交互引导**：询问用户是使用“物理真机”还是“安卓模拟器”。

       - **若为模拟器**：引导用户运行连接命令（以常见模拟器默认端口为例，如雷电模拟器）：

         ```
         adb connect 127.0.0.1:5555
         ```

         *(注意：若在 adb 安装目录下，应使用 `.\adb connect 127.0.0.1:5555`)*

       - **若依然找不到设备或处于 offline 状态**：引导用户执行 ADB 服务重启重置状态：

         ```
         adb kill-server; adb start-server
         ```

         随后重新运行 `adb devices`。

     - **情况 B：显示 unauthorized**：引导用户在手机屏幕上允许 USB 调试授权。

     - **情况 C：成功识别设备**：输出至少一台状态为 `device` 的设备，进入下一步。

  3. **检查本地 Frida 环境**:

     - 运行检测：

       ```
       python -c "import frida; print(frida.__version__)"
       Get-Command frida-ps -ErrorAction SilentlyContinue
       ```

     - *判定*: 若报错，提供安装指引：`pip install frida-tools frida`

     - **记录本地 Frida 版本**：记录输出的版本号（例如 `16.2.1`），用于后续下载和匹配手机端的 `frida-server`。

  ### 阶段二：设备选择与 Server 部署 (Device & Server Deployment)

  1. **设备选择**:

     - 若 `adb devices` 存在多台设备，要求用户选择。
     - **后续所有 adb 命令必须自动追加 `-s <serial>` 参数。**

  2. **设备 CPU 架构检测与 Server 版本匹配**:

     - **第一步：自动查询目标设备架构**。引导用户（或通过工具自动）在 PowerShell 中执行：

       ```
       adb -s <serial> shell getprop ro.product.cpu.abi
       ```

     - **第二步：对照确认官方文件名架构**。根据返回值，向用户展示并建议其使用的官方 `frida-server` 文件架构后缀：

       - 若返回 `arm64-v8a`，提示选用：`frida-server-<version>-android-arm64`
       - 若返回 `armeabi-v7a` 或 `armeabi`，提示选用：`frida-server-<version>-android-arm`
       - 若返回 `x86_64`，提示选用：`frida-server-<version>-android-x86_64`（常见于 64位 模拟器）
       - 若返回 `x86`，提示选用：`frida-server-<version>-android-x86`（常见于 32位 模拟器）

     - **第三步：路径录入**。引导用户输入本地已下载且匹配该架构的 `frida-server` 文件完整路径。

  3. **询问运行权限模式**:

     - 询问用户是否使用 **Root 模式** 运行（默认：是）。

  4. **执行部署 (需确认)**: 获取确认后，在 PowerShell 中执行以下命令：

     ```
     adb -s <serial> push "$local_frida_server_path" /data/local/tmp/frida-server
     adb -s <serial> shell chmod 777 /data/local/tmp/frida-server
     ```

  5. **清理冲突并重置端口转发**: 在启动前，在 PowerShell 中执行以下清理流程（防止由于上次异常退出导致的进程和端口残留）：

     ```
     # 1. 杀死 Android 端残留 server 进程 (内部使用 Android Linux Shell 兼容语法)
     adb -s <serial> shell "killall frida-server 2>/dev/null || true"
     adb -s <serial> shell "pkill -9 frida-server 2>/dev/null || true"
     # 2. 释放 Windows 本地可能残留的转发端口
     adb -s <serial> forward --remove tcp:27042
     # 3. 建立新的端口转发
     adb -s <serial> forward tcp:27042 tcp:27042
     ```

  6. **非阻塞式后台启动 frida-server (PowerShell 专属)**: 使用 PowerShell 原生的 `Start-Process` 命令，在后台开启一个**隐藏窗口**运行 ADB，从而防止本地终端因等待 shell 输出而陷入死锁/假死状态：

     - **非 Root 模式**:

       ```
       Start-Process adb -ArgumentList "-s <serial> shell /data/local/tmp/frida-server" -WindowStyle Hidden
       ```

     - **Root 模式**:

       ```
       Start-Process adb -ArgumentList "-s <serial> shell su -c /data/local/tmp/frida-server" -WindowStyle Hidden
       ```

  7. **多维度检查并验证 frida-server 状态**: 等待 2 秒后，引导用户运行以下命令进行状态深度校验（若其中任意一项未达标，则跳转至常见问题排查）：

     - **维度 A：检查 Android 手机端进程是否存在**

       ```
       adb -s <serial> shell "ps -A | grep frida"
       ```

       *(预期输出：应能看到包含 `frida-server` 的进程行及 PID)*

     - **维度 B：检查 Android 手机端端口监听状态 (27042)**

       ```
       adb -s <serial> shell "netstat -an | grep 27042"
       ```

       *(预期输出：应显示 `0.0.0.0:27042` 或 `:::27042` 处于 LISTEN 监听状态)*

     - **维度 C：检查 Windows 本地端口转发状态**

       ```
       adb forward --list
       ```

       *(预期输出：应包含本地 `tcp:27042` 转发到手机端 `tcp:27042` 的规则)*

     - **维度 D：PC 端 Frida 联通性测试**

       ```
       frida-ps -U
       ```

       *(预期输出：成功打印出手机正在运行的系统进程列表)*

  ### 阶段三：目标选择与脚本准备 (Targeting)

  1. **进程检索**: 运行以下命令，获取设备上已安装的应用程序列表：

     ```
     frida-ps -Uai
     ```

     *格式化输出给用户选择*：

     ```
     请选择您要 Hook 的目标应用编号：
     [1] 微信 (com.tencent.mm)
     [2] 支付宝 (com.eg.android.AlipayGphone)
     [3] 目标应用名 (com.example.target)
     ...
     ```

  2. **定位 Hook 脚本**:

     - 询问用户要注入的 `.js` 脚本路径。

     - *主动推荐*：在 PowerShell 中扫描当前目录下的所有 `*.js` 脚本：

       ```
       Get-ChildItem *.js | Select-Object -Property Name
       ```

       以紧凑格式列出供用户进行编号快速选择。

  3. **确定注入模式**: 询问用户注入时机：

     - **Spawn 模式**（冷启动注入，适合 Hook 启动初始化代码）：推荐。
     - **Attach 模式**（热挂载注入，挂载到当前已运行的进程）。

  ### 阶段四：执行 Hook 注入 (Execution)

  根据用户的选择，构建并执行对应的 Frida 注入命令。

  - **Spawn 模式**:

    ```
    frida -U -f <包名> -l "$script_path"
    ```

  - **Attach 模式**:

    ```
    frida -U -n <包名> -l "$script_path"
    ```

  **执行规范**：

  - 在当前 PowerShell 窗口直接启动注入进程，**实时流式输出**本地及手机端的 `console.log()` 数据。
  - 告知用户："输入 **'q'**、**'exit'** 或按下 **Ctrl+C** 可以随时终止 Hook 并清理环境。"

  ### 阶段五：停止与清理 (Teardown & Cleanup)

  当用户主动终止、脚本执行完毕或发生异常中断时，执行标准清理：

  1. 按 **Ctrl+C** 终止本地的 `frida` 控制台进程。

  2. 询问用户是否清理手机端的 environment：

     ```
     # 杀死手机端后台 server 进程
     adb -s <serial> shell "killall frida-server"
     # 移除本地 Windows 端口转发
     adb -s <serial> forward --remove tcp:27042
     ```

  3. 友好提示用户当前 Windows 本地与 Android 设备环境已恢复干净。

  ## 3. 命令速查表 (Quick Reference for PowerShell)

  | 操作类别            | PowerShell 专用命令                                          | 说明                                           |
  | ------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
  | **临时 adb 变量**   | `$env:Path += ";C:\path\to\platform-tools"`                  | 在当前会话中临时追加并启用 ADB                 |
  | **检测本地 adb**    | `Get-Command adb`                                            | 检查本地系统环境变量中是否存在 adb             |
  | **重启 ADB 服务**   | `adb kill-server; adb start-server`                          | 彻底重置/重启本地的 ADB 服务（解决挂起或离线） |
  | **连接网络/模拟器** | `adb connect 127.0.0.1:5555`                                 | 连接指定 IP 端口的无线设备或本地模拟器         |
  | **设备识别**        | `adb devices`                                                | 查看已连接设备列表                             |
  | **架构检测**        | `adb -s <sn> shell getprop ro.product.cpu.abi`               | 查手机 CPU 架构以匹配 frida-server 文件        |
  | **文件传输**        | `adb -s <sn> push "$local_path" /data/local/tmp/frida-server` | 将本地 server 推送至设备端                     |
  | **权限提升**        | `adb -s <sn> shell chmod 777 /data/local/tmp/frida-server`   | 赋予执行权限                                   |
  | **端口转发**        | `adb -s <sn> forward tcp:27042 tcp:27042`                    | 桥接 Windows 与手机对应端口                    |
  | **非阻塞启动**      | `Start-Process adb -ArgumentList "-s <sn> shell /data/local/tmp/frida-server" -WindowStyle Hidden` | 隐藏后台开启 server，防止终端假死              |
  | **检查进程状态**    | `adb -s <sn> shell "ps -A | grep frida"`                     | 查看手机端 `frida-server` 进程是否存活         |
  | **检查端口监听**    | `adb -s <sn> shell "netstat -an | grep 27042"`               | 确认手机端服务是否开启了 `27042` 端口监听      |
  | **检查转发状态**    | `adb forward --list`                                         | 确认 Windows 本地与 Android 的端口转发规则     |
  | **进程检索**        | `frida-ps -Uai`                                              | 列出设备上所有可调试应用及包名                 |
  | **Spawn 注入**      | `frida -U -f <pkg> -l "$script_path"`                        | 冷启动应用并 Hook                              |
  | **Attach 注入**     | `frida -U -n <pkg> -l "$script_path"`                        | 附加到现有正在运行的进程 Hook                  |
  | **残余清理**        | `adb -s <sn> shell "killall frida-server"`                   | 强杀手机端 server 进程                         |
  | **移除转发**        | `adb -s <sn> forward --remove tcp:27042`                     | 释放 Windows 本地 27042 端口                   |

