# 3 个真实排错样例

> 这 3 条是**本机复现记录**，不是客户成交案例，不含客户隐私；报错文本为终端原样输出，仅省略本机路径中段。
> **客户成交案例：案例待补**（截至 2026-10-23 无成交记录，不编造订单、收入或评价）。
> 用途：让你先看清「现象 → 原因 → 命令 → 验证」的交付格式，再决定要不要付费。

---

## 案例 1 · PowerShell 禁止运行脚本，venv 激活失败

> 环境：Windows 10.0.26200.9711 / Windows PowerShell 5.1 / Python 3.13.7

### 现象

```powershell
PS> .\.venv\Scripts\Activate.ps1
& : File C:\...\venv3\Scripts\Activate.ps1 cannot be loaded because running scripts is disabled on this system. For more information, see about_Execution_Policies at https:/go.microsoft.com/fwlink/?LinkID=135170.
At line:1 char:3
+ & 'C:\...\venv3\Scripts\Activate.ps1'
+   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : SecurityError: (:) [], PSSecurityException
    + FullyQualifiedErrorId : UnauthorizedAccess
```

同一个 venv 里，`Scripts\python.exe` 可以直接运行，`import` 也正常；只有 `Activate.ps1` 被拦。所以不是 Python 或虚拟环境坏了。

本机 `Get-ExecutionPolicy -List` 的实际输出：

```text
        Scope ExecutionPolicy
        ----- ---------------
MachinePolicy       Undefined
   UserPolicy       Undefined
      Process       Undefined
  CurrentUser       Undefined
 LocalMachine       Undefined
```

五项都是 `Undefined`，Windows 客户端会按默认的 `Restricted` 处理：所有 `.ps1` 都不许跑。

### 原因

1. Windows 客户端默认执行策略是 `Restricted`，`Activate.ps1` 属于脚本，被策略层拦下，跟包有没有装好无关。
2. 用 `cmd.exe` 或 Git Bash 激活过同一个 venv，以为 PowerShell 也能用；两种 shell 的激活脚本不是同一个。
3. 公司组策略（`MachinePolicy` / `UserPolicy`）把策略锁成 `Restricted` 或 `AllSigned`，这时改 `CurrentUser` 不生效。
4. 只改了 `LocalMachine` 但当前进程没刷新，或者改的是另一个用户。

### 命令

```powershell
# 先确认是哪一层在管
Get-ExecutionPolicy -List

# 方案 1：只放开当前用户，重开终端生效（推荐）
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned

# 方案 2：只放开当前窗口，不改系统设置
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass

# 方案 3：不激活，直接用 venv 里的解释器（最不容易被策略影响）
& .\.venv\Scripts\python.exe -m pip install <包名>
& .\.venv\Scripts\python.exe your_script.py
```

`RemoteSigned` 的含义是：本地写的脚本可以跑，从网上下载的脚本要有有效签名。不要用 `Unrestricted` 或 `Bypass` 去改 `LocalMachine` 长期兜底。

如果 `Get-ExecutionPolicy -List` 里 `MachinePolicy` / `UserPolicy` 不是 `Undefined`，说明是域策略在管，需要找管理员，单机改不了。

### 验证

```powershell
powershell -NoProfile -ExecutionPolicy Bypass `
  -Command "& 'C:\...\venv3\Scripts\Activate.ps1'; python -c 'import sys; print(sys.prefix)'"
```

本机实际输出（退出码 0）：

```text
C:\...\venv3
```

`sys.prefix` 指向 venv 目录，说明激活脚本本身能执行，问题只在执行策略。之后重开一个 PowerShell，执行 `Get-ExecutionPolicy -Scope CurrentUser`，应看到 `RemoteSigned`；再 `Activate.ps1` 不报错才算修好。

---

## 案例 2 · Python 3.13 装 torch==2.0.1：No matching distribution

> 环境：Windows 10.0.26200.9711 / Python 3.13.7 / pip 26.2.1 / NVIDIA GeForce RTX 5060（驱动 610.88）

### 现象

```powershell
PS> python -m pip install --dry-run "torch==2.0.1"
Defaulting to user installation because normal site-packages is not writeable
ERROR: Could not find a version that satisfies the requirement torch==2.0.1 (from versions: 2.6.0, 2.7.0, 2.7.1, 2.8.0, 2.9.0, 2.9.1, 2.10.0, 2.11.0, 2.12.0, 2.12.1, 2.13.0, 2.14.0, 2.14.1)
ERROR: No matching distribution found for torch==2.0.1
```

`--dry-run` 只做解析、不下载安装，所以这个报错跟磁盘、网速、显卡都无关。`from versions:` 那一行已经把当前解释器能装的版本列全了。

### 原因

1. Python 3.13 的 wheel tag 是 `cp313`。PyTorch 2.0.1 发布时只做到 `cp311`，没有 `cp313` 的 Windows wheel，pip 不会拿旧 ABI 的包硬装。
2. PyPI 上 `cp313` 的 torch 从 **2.6.0** 才开始有；`2.0.1` 在 `pip index versions torch` 里根本不在候选列表。
3. 老教程、旧 `requirements.txt`、别人给的 `pip install torch==2.0.1` 把版本写死了，换到新 Python 就复现。
4. 换 CUDA 源（`download.pytorch.org/whl/cu118`）也没用：源里同样没有 `cp313` 的 2.0.1 wheel。这是 Python ABI 不匹配，不是 CUDA 版本问题。

### 命令

```powershell
# 1. 先确认解释器和 pip 是同一套
python -V
python -m pip -V

# 2. 看当前源上这个解释器到底能装哪些版本
python -m pip index versions torch

# 3. 安全复现：只解析，不安装
python -m pip install --dry-run "torch==2.0.1"

# 4A. 继续用 Python 3.13：装候选列表里的版本（示例 2.14.1）
python -m pip install "torch==2.14.1"

# 4B. 必须留 2.0.1：换 Python 3.11 建独立环境，别动全局
py -3.11 -m venv .venv311
.\.venv311\Scripts\python.exe -m pip install --upgrade pip
.\.venv311\Scripts\python.exe -m pip install "torch==2.0.1"

# 需要 GPU 版时，先看驱动支持的 CUDA 上限，再选对应 index-url
nvidia-smi
# 例：cu124 源（按你的驱动改，不要照抄）
python -m pip install torch --index-url https://download.pytorch.org/whl/cu124
```

不要在 3.13 环境里反复 `pip install torch==2.0.1`；解析阶段就失败，重试不会变出 wheel。

### 验证

```powershell
PS> python -m pip index versions torch
torch (2.14.1)
Available versions: 2.14.1, 2.14.0, 2.13.0, 2.12.1, 2.12.0, 2.11.0, 2.10.0, 2.9.1, 2.9.0, 2.8.0, 2.7.1, 2.7.0, 2.6.0
```

本机实际输出如上（退出码 0）：候选列表里没有 `2.0.1`，但 `2.14.1` 可用，说明问题在版本选择，不在网络。

装完后再跑：

```powershell
python -c "import torch; print(torch.__version__)"
```

应打印出你实际安装的版本（例如 `2.14.1`）。如果打印的仍是旧版本，说明 `python` 和 `pip` 不是同一个解释器，回到第 1 步用 `sys.executable` 对齐。

---

## 案例 3 · pip 一直 Retrying，最后却说 No matching distribution（代理指错）

> 环境：Windows 10.0.26200.9711 / Python 3.13.7 / pip 26.2.1  
> 复现方式：把 `HTTPS_PROXY` / `HTTP_PROXY` 指向本机没有监听的 `127.0.0.1:9`，然后查包。

### 现象

```powershell
PS> $env:HTTPS_PROXY = "http://127.0.0.1:9"
PS> $env:HTTP_PROXY  = "http://127.0.0.1:9"
PS> python -m pip index versions torch
WARNING: Retrying (Retry(total=4, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError("HTTPSConnection(host='127.0.0.1', port=9): Failed to establish a new connection: [WinError 10061] 由于目标计算机积极拒绝，无法连接。")': /simple/torch/
WARNING: Retrying (Retry(total=3, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError("HTTPSConnection(host='127.0.0.1', port=9): Failed to establish a new connection: [WinError 10061] 由于目标计算机积极拒绝，无法连接。")': /simple/torch/
...（Retry(total=2) / (total=1) 两条内容相同，省略）
WARNING: Retrying (Retry(total=0, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError("HTTPSConnection(host='127.0.0.1', port=9): Failed to establish a new connection: [WinError 10061] 由于目标计算机积极拒绝，无法连接。")': /simple/torch/
ERROR: No matching distribution found for torch
```

关键点：**它最后说的是包不存在，但真正错的是代理连不上。** 只看最后一行会误判成“源上没有这个包”，然后去换镜像、降版本，越走越偏。

### 原因

1. `HTTPS_PROXY` / `HTTP_PROXY` 指到了 `127.0.0.1:9`，这个端口没有代理进程在监听，连接被系统直接拒绝（`WinError 10061`）。
2. pip 对每个索引地址默认重试 5 次，5 次都失败后，把“所有候选地址都拿不到”归并成 `No matching distribution found`。
3. 代理软件退出后环境变量还留在当前终端；新开一个窗口可能就好了，所以表现为“时好时坏”。
4. 也可能是 `pip.ini` / `pip.conf` 里写了 `proxy`，或者公司代理只允许特定域名。

### 命令

```powershell
# 1. 看当前终端到底带了什么代理
$env:HTTPS_PROXY
$env:HTTP_PROXY
$env:ALL_PROXY

# 2. 看 pip 自己的配置文件（用户级 / 全局）
python -m pip config debug

# 3. 确认这个代理端口有没有人监听
Test-NetConnection 127.0.0.1 -Port 9

# 4A. 代理是误设的：清掉当前窗口的变量再试
Remove-Item Env:HTTPS_PROXY, Env:HTTP_PROXY, Env:ALL_PROXY -ErrorAction SilentlyContinue
python -m pip index versions torch

# 4B. 确实需要代理：把端口改成代理软件实际监听的端口
$env:HTTPS_PROXY = "http://127.0.0.1:7890"
$env:HTTP_PROXY  = "http://127.0.0.1:7890"
python -m pip index versions torch

# 4C. 代理要认证时，不要把密码写进仓库；用环境变量或 pip 的 keyring
# 4D. 只是某个镜像慢：换源并放宽超时，不要和代理问题混在一起
python -m pip install "torch==2.14.1" `
  -i https://pypi.tuna.tsinghua.edu.cn/simple `
  --timeout 60 --retries 3
```

如果在 `pip config debug` 里看到 `global.proxy`，用 `python -m pip config unset global.proxy` 去掉；改完重开终端再验。

### 验证

```powershell
PS> Remove-Item Env:HTTPS_PROXY, Env:HTTP_PROXY -ErrorAction SilentlyContinue
PS> python -m pip index versions torch
torch (2.14.1)
Available versions: 2.14.1, 2.14.0, 2.13.0, 2.12.1, 2.12.0, 2.11.0, 2.10.0, 2.9.1, 2.9.0, 2.8.0, 2.7.1, 2.7.0, 2.6.0
```

本机实际输出如上（退出码 0）。能列出候选版本，说明到索引源的链路通了；这时再装包才有意义。若清掉代理仍然 `Retrying`，把 `python -m pip index versions torch -v` 的完整输出贴进 Issue，重点看它连的是哪个 host。

---

## 这些案例怎么用

- 你可以直接照命令自查；能自己解决就不用付费。
- 报错不一样也没关系：开一条 [排错咨询](https://github.com/autumnpyfaheyjjd32-jpg/s1-ai-env-rescue/issues/new?template=consult.md)，贴完整报错，我先判断档位。
- 这 3 条只证明“会按现象查因、给命令、做验证”，不证明你的项目一定能修；边界见 [price.md](./price.md)。
