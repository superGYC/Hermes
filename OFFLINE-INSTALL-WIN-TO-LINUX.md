# Hermes Agent 跨平台离线安装手册（Windows → Linux）

**版本**: v1.0 — Windows 收集 → Linux 安装  
**适用场景**: 有网机器是 Windows，离线服务器是 Linux，无虚拟机  
**核心挑战**: Python .whl 跨平台不兼容，需在 Windows 上下载 Linux 专用包  
**日期**: 2026-05-21

---

## 目录

1. [为什么复杂](#为什么复杂)
2. [核心思路](#核心思路)
3. [阶段一：Windows 有网机器 — 收集 Linux 文件](#阶段一windows-有网机器--收集-linux-文件)
4. [阶段二：传输到 Linux 离线服务器](#阶段二传输到-linux-离线服务器)
5. [阶段三：Linux 离线服务器 — 全部安装](#阶段三linux-离线服务器--全部安装)
6. [下载地址汇总（Linux 版）](#下载地址汇总linux-版)
7. [一键 PowerShell 收集脚本](#一键-powershell-收集脚本)
8. [故障排查](#故障排查)
9. [附录](#附录)

---

## 为什么复杂

### 问题 1: Python 包不跨平台

Windows 上的 `.whl` 文件 ≠ Linux 上的 `.whl` 文件。

```
# ❌ Windows 版（不能在 Linux 上用）
pydantic-2.12.5-cp311-cp311-win_amd64.whl

# ✅ Linux 版（我们需要下载这种）
pydantic-2.12.5-cp311-cp311-manylinux_2_17_x86_64.manylinux2014_x86_64.whl

# ✅ 纯 Python 包（跨平台，哪都能用）
requests-2.31.0-py3-none-any.whl
```

### 问题 2: 系统依赖包格式不同

Windows 没有 `apt-get`，无法直接下载 `.deb` 包。

### 解决方案

| 资源 | Windows 收集方式 |
|------|-----------------|
| **Python whl** | `pip download --platform manylinux2014_x86_64`（pip 支持跨平台下载） |
| **系统依赖 .deb** | 手动从 Ubuntu 软件包网站下载，或假设离线服务器有内网 apt 源 |
| **uv / Node.js** | 直接从 GitHub 下载 Linux 版二进制（`.tar.gz` / `.tar.xz`） |
| **Hermes 源码** | zip 文件，跨平台 |

---

## 核心思路

```
┌─────────────────────────────────────────────────────────────┐
│              Windows 10/11 有网机器（文件收集站）              │
│                                                              │
│  1. 安装 Python 3.11 for Windows（临时用）                     │
│  2. pip download --platform manylinux2014_x86_64               │
│     → 下载 Linux 版 .whl 到收集目录                            │
│  3. 浏览器下载 Linux 版 uv.tar.gz / node.tar.xz               │
│  4. 下载 Hermes 源码 zip                                      │
│  5. （可选）浏览器下载 .deb 系统依赖包                          │
│                                                              │
│  打包 → hermes-offline-linux.tar.gz                           │
└──────────────────────────┬───────────────────────────────────┘
                           │ USB / 硬盘 / 内网文件服务器
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Linux 离线服务器（无外网）                         │
│                                                              │
│  1. 解压收集包                                               │
│  2. 安装系统依赖（dpkg -i 或 apt-get 本地源）                  │
│  3. tar 解压 uv / Node.js                                    │
│  4. unzip 解压 Hermes 源码                                    │
│  5. uv venv + uv pip install --no-index --find-links          │
│     → 从 Windows 收集的 Linux 版 .whl 安装                      │
│  6. 创建 hermes 命令 + .env                                   │
│  7. 启动                                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 阶段一：Windows 有网机器 — 收集 Linux 文件

### 前置要求

- Windows 10/11 64位，能访问互联网
- 磁盘空间：3GB 可用
- 已安装：PowerShell 5.1+

### Step 1 — 创建收集目录

打开 **PowerShell**：

```powershell
$collectDir = "C:\temp\hermes-offline-linux"
New-Item -ItemType Directory -Force -Path "$collectDir\binaries"
New-Item -ItemType Directory -Force -Path "$collectDir\src"
New-Item -ItemType Directory -Force -Path "$collectDir\python-packages"
New-Item -ItemType Directory -Force -Path "$collectDir\system-packages"
New-Item -ItemType Directory -Force -Path "$collectDir\tools"

Set-Location $collectDir
```

### Step 2 — 临时安装 Python 3.11 for Windows（解析依赖用）

```powershell
# 下载 Python 3.11 安装程序
Invoke-WebRequest -Uri "https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe" -OutFile "$collectDir\binaries\python-installer.exe"

# 静默安装到临时目录
Start-Process -FilePath "$collectDir\binaries\python-installer.exe" `
  -ArgumentList "/quiet", "InstallAllUsers=0", "PrependPath=1", "TargetDir=$collectDir\tools\python311" `
  -Wait

# 验证
& "$collectDir\tools\python311\python.exe" --version
# 输出: Python 3.11.9
```

### Step 3 — 下载 Linux 版 uv

```powershell
# x86_64 Linux 版 uv（不是 Windows 版！）
Invoke-WebRequest -Uri "https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-unknown-linux-gnu.tar.gz" -OutFile "$collectDir\binaries\uv-linux.tar.gz"

# 验证（不解压，只看文件名）
Get-Item "$collectDir\binaries\uv-linux.tar.gz"
```

### Step 4 — 下载 Linux 版 Node.js v22

```powershell
# x86_64 Linux 版 Node.js
Invoke-WebRequest -Uri "https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-x64.tar.xz" -OutFile "$collectDir\binaries\node-linux.tar.xz"

# 验证
Get-Item "$collectDir\binaries\node-linux.tar.xz"
```

### Step 5 — 下载 Hermes 源码

```powershell
Invoke-WebRequest -Uri "https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.zip" -OutFile "$collectDir\src\hermes.zip"

# 验证
Expand-Archive -Path "$collectDir\src\hermes.zip" -DestinationPath "$collectDir\tools\src-temp" -Force
Get-ChildItem "$collectDir\tools\src-temp"
# 应看到: hermes-agent-main
Remove-Item -Recurse -Force "$collectDir\tools\src-temp"
```

### Step 6 — 解析依赖并下载 Linux 版 Python whl（核心步骤）

这是最难的一步：Windows 上必须下载 **Linux 专用** 的 `.whl` 文件。

#### 6.1 创建临时 venv，解析依赖清单

```powershell
# 解压 Hermes 源码到临时目录
Expand-Archive -Path "$collectDir\src\hermes.zip" -DestinationPath "$collectDir\tools\src" -Force
Set-Location "$collectDir\tools\src\hermes-agent-main"

# 创建临时 venv（用 Windows Python）
& "$collectDir\tools\python311\python.exe" -m venv "$collectDir\tools\temp-venv"

# 安装 Hermes（解析全部 218 个依赖）
& "$collectDir\tools\temp-venv\Scripts\pip.exe" install -e ".[all]"

# 导出完整依赖清单
& "$collectDir\tools\temp-venv\Scripts\pip.exe" freeze | Out-File "$collectDir\requirements-all.txt" -Encoding utf8

# 查看数量
(Get-Content "$collectDir\requirements-all.txt").Count
# 输出: 200+
```

#### 6.2 下载 Linux 版 .whl 文件

**关键命令**：`pip download --platform manylinux2014_x86_64`

pip 支持**跨平台下载**：在 Windows 上运行，但下载指定平台的包。

```powershell
# ████████████████████████████████████████████████████████████████
# 关键：下载 Linux 版 whl（不是 Windows 版！）
# --platform manylinux2014_x86_64 = x86_64 Linux
# --python-version 3.11 = Python 3.11
# --abi cp311 = CPython 3.11 ABI
# ████████████████████████████████████████████████████████████████

& "$collectDir\tools\temp-venv\Scripts\pip.exe" download `
  -r "$collectDir\requirements-all.txt" `
  -d "$collectDir\python-packages" `
  --platform manylinux2014_x86_64 `
  --python-version 3.11 `
  --abi cp311 `
  --only-binary :all:
```

**常见问题处理**：

如果命令报错 "No matching distribution found"，说明某个包没有 manylinux 版本：

```powershell
# 方案 A: 去掉 --only-binary，允许下载 sdist（源码包，离线服务器编译）
& "$collectDir\tools\temp-venv\Scripts\pip.exe" download `
  -r "$collectDir\requirements-all.txt" `
  -d "$collectDir\python-packages" `
  --platform manylinux2014_x86_64 `
  --python-version 3.11 `
  --abi cp311

# 方案 B: 分两步，先下载纯 Python 包（跨平台），再下载平台特定包
# 步骤 1: 下载纯 Python 包（py3-none-any，不指定平台）
& "$collectDir\tools\temp-venv\Scripts\pip.exe" download `
  -r "$collectDir\requirements-all.txt" `
  -d "$collectDir\python-packages" `
  --no-deps  # 先不递归，配合手动处理

# 步骤 2: 下载平台特定包（指定 manylinux）
# 这个更复杂，建议直接用方案 A
```

#### 6.3 验证下载结果

```powershell
# 查看 whl 数量
$whlCount = (Get-ChildItem "$collectDir\python-packages\*.whl").Count
$sdistCount = (Get-ChildItem "$collectDir\python-packages\*.tar.gz").Count
Write-Host "whl 文件: $whlCount"
Write-Host "sdist 源码包: $sdistCount"
# whl 应占大多数（200+），sdist 可能有少量

# 确认是 Linux 版 whl（不是 win_amd64）
Get-ChildItem "$collectDir\python-packages\*.whl" | Select-Object -First 10 Name
# 应看到: manylinux, linux, 或 py3-none-any
# ❌ 不应看到: win_amd64, win32
```

#### 6.4 清理临时环境

```powershell
# 删除临时 Python、venv、源码（Windows 上不留下 Hermes 痕迹）
Remove-Item -Recurse -Force "$collectDir\tools\python311"
Remove-Item -Recurse -Force "$collectDir\tools\temp-venv"
Remove-Item -Recurse -Force "$collectDir\tools\src"
```

### Step 7 — 下载系统依赖 .deb 包（可选，推荐）

**问题**：Windows 上没有 `apt-get`，怎么下载 `.deb`？

**方案 A（推荐）**：离线服务器有内网 apt 源
- 不需要在 Windows 上收集 .deb
- 离线服务器安装时 `apt-get install` 从内网源获取

**方案 B**：离线服务器完全无外网也无内网源
- 需要在 Windows 上用浏览器手动下载 .deb 包
- 或找一台有网的 Linux 机器下载后传给 Windows 打包

**方案 C**：最稳妥——找一台临时 Linux 机器下载 .deb

如果你有临时 Linux 机器（哪怕虚拟机，只要有网）：

```bash
# 在临时 Linux 机器上执行
mkdir -p /tmp/debs
apt-get download git python3.11 python3.11-venv python3.11-dev build-essential libffi-dev libssl-dev ripgrep ffmpeg ca-certificates
cp *.deb /tmp/debs/
tar -czvf debs.tar.gz /tmp/debs/
# 然后把 debs.tar.gz 传到 Windows 机器，放入 $collectDir\system-packages\
```

如果没有临时 Linux 机器，Windows 上手动下载 .deb（较麻烦）：

```powershell
# 用浏览器访问以下 URL，逐个下载（以 Ubuntu 22.04 Jammy 为例）
# 注意：版本号会变化，这里只是示例格式

# git: https://packages.ubuntu.com/jammy/git
# python3.11: https://packages.ubuntu.com/jammy/python3.11
# ...

# 更简单的方法：使用 Ubuntu 软件包搜索页面
# https://packages.ubuntu.com/
# 搜索每个包名，下载 amd64 版本的 .deb
```

**建议**：如果你的离线服务器能访问**内网 apt 源**（公司/园区内部源），跳过这步，直接在离线服务器上 `apt-get install`。

### Step 8 — 打包

```powershell
Set-Location C:\temp

# 使用 7-Zip（如果安装了）
# & "C:\Program Files\7-Zip\7z.exe" a -ttar -so hermes-offline-linux.tar hermes-offline-linux | & "C:\Program Files\7-Zip\7z.exe" a -si hermes-offline-linux.tar.gz

# 或者用 PowerShell 的 Compress-Archive（压缩率较低，但通用）
Compress-Archive -Path "hermes-offline-linux" -DestinationPath "hermes-offline-linux.zip" -Force

# 查看大小
Get-Item "C:\temp\hermes-offline-linux.zip" | Select-Object Name, @{Name="Size(MB)";Expression={[math]::Round($_.Length/1MB,2)}}
# 预期: 800MB - 1.5GB
```

### 阶段一验证清单

| 检查项 | PowerShell 命令 | 预期结果 |
|--------|----------------|----------|
| Linux uv | `Get-Item $collectDir\binaries\uv-linux.tar.gz` | 文件存在 |
| Linux Node.js | `Get-Item $collectDir\binaries\node-linux.tar.xz` | 文件存在 |
| Hermes 源码 | `Get-Item $collectDir\src\hermes.zip` | 文件存在 |
| Python whl | `(Get-ChildItem $collectDir\python-packages\*.whl).Count` | >200 |
| whl 是 Linux 版 | `Get-ChildItem ...\*.whl \| Where-Object {$_.Name -match 'win'}` | **无匹配**（不应有 win_amd64） |
| 依赖清单 | `Get-Item $collectDir\requirements-all.txt` | 文件存在 |

---

## 阶段二：传输到 Linux 离线服务器

和 Linux v3.0 手册相同。

### 方法 A — USB / 移动硬盘

```powershell
# Windows 有网机器
Copy-Item -Path "C:\temp\hermes-offline-linux.zip" -Destination "D:\hermes-offline-linux.zip" -Force

# 拔下 U 盘，插到 Linux 离线服务器
# Linux 上执行:
sudo mkdir -p /mnt/usb
sudo mount /dev/sdb1 /mnt/usb  # 根据实际情况调整设备名
cp /mnt/usb/hermes-offline-linux.zip /opt/
```

### 方法 B — 局域网文件共享 / 堡垒机

```powershell
# Windows → 共享文件夹
Copy-Item -Path "C:\temp\hermes-offline-linux.zip" -Destination "\\fileserver\shared\hermes-offline-linux.zip" -Force

# Linux 从共享复制（需安装 cifs-utils）
sudo apt-get install -y cifs-utils  # 如果有内网源
sudo mount -t cifs //fileserver/shared /mnt/shared -o username=xxx
sudo cp /mnt/shared/hermes-offline-linux.zip /opt/
```

---

## 阶段三：Linux 离线服务器 — 全部安装

**和 Linux v3.0 手册完全一致**，以下是完整步骤。

### Step 1 — 解压

```bash
cd /opt

# 安装 unzip（如果没有）
# sudo apt-get install -y unzip

unzip -q hermes-offline-linux.zip

# 确认结构
ls -la /opt/hermes-offline-linux/
# 应看到: binaries/ python-packages/ src/ system-packages/ requirements-all.txt
```

### Step 2 — 安装系统依赖

#### 方案 A：离线服务器有内网 apt 源（推荐）

```bash
# 不需要 .deb 包，直接从内网源安装
sudo apt-get update
sudo apt-get install -y \
  git \
  python3.11 \
  python3.11-venv \
  python3.11-dev \
  build-essential \
  libffi-dev \
  libssl-dev \
  ripgrep \
  ffmpeg \
  ca-certificates
```

#### 方案 B：使用 Windows 收集的 .deb 包

```bash
cd /opt/hermes-offline-linux/system-packages

# 安装所有 .deb
sudo dpkg -i *.deb
sudo apt-get install -f -y
```

#### 方案 C：离线服务器没有 apt 源也没有 .deb

需要编译安装 Python 3.11 和相关工具。参考附录 C。

### Step 3 — 安装 uv

```bash
cd /opt/hermes-offline-linux/binaries

# 解压到 /usr/local/bin
sudo tar -xzf uv-linux.tar.gz -C /usr/local/bin/

# 验证
/usr/local/bin/uv --version
```

### Step 4 — 安装 Node.js

```bash
cd /opt/hermes-offline-linux/binaries

# 创建目录
sudo mkdir -p /opt/node

# 解压 Linux 版 Node.js
sudo tar -xJf node-linux.tar.xz -C /opt/node --strip-components=1

# 创建命令链接
sudo ln -sf /opt/node/bin/node /usr/local/bin/node
sudo ln -sf /opt/node/bin/npm /usr/local/bin/npm
sudo ln -sf /opt/node/bin/npx /usr/local/bin/npx

# 验证
node --version
npm --version
```

### Step 5 — 解压 Hermes 源码

```bash
cd /opt/hermes-offline-linux/src

# 解压
sudo unzip -q hermes.zip -d /opt/

# 重命名
sudo mv /opt/hermes-agent-main /opt/hermes-agent

# 验证
cd /opt/hermes-agent
ls -la
```

### Step 6 — 创建虚拟环境并从本地 whl 安装

```bash
cd /opt/hermes-agent

# 创建 venv
sudo /usr/local/bin/uv venv /opt/hermes-agent/venv --python 3.11

# 设置环境变量
export VIRTUAL_ENV="/opt/hermes-agent/venv"

# 从本地 whl 安装（--no-index = 不连外网）
sudo /usr/local/bin/uv pip install \
  --no-index \
  --find-links /opt/hermes-offline-linux/python-packages/ \
  -e ".[all]"
```

### Step 7 — 创建 hermes 命令

```bash
sudo tee /usr/local/bin/hermes << 'EOF'
#!/bin/bash
export PYTHONPATH=""
export PYTHONHOME=""
export UV_NO_CONFIG=1
exec "/opt/hermes-agent/venv/bin/python" -m hermes_cli.main "$@"
EOF

sudo chmod +x /usr/local/bin/hermes
hermes --version
```

### Step 8 — 创建配置

```bash
mkdir -p ~/.hermes/{cron,sessions,logs,pairing,hooks,image_cache,audio_cache,memories,skills}

cat > ~/.hermes/.env << 'EOF'
OPENAI_BASE_URL=http://127.0.0.1:11434/v1
OPENAI_API_KEY=ollama
OPENAI_MODEL=llama3.1:70b
API_SERVER_ENABLED=true
API_SERVER_KEY=your-offline-production-key
WEB_SEARCH_ENABLED=false
TTS_ENABLED=false
STT_ENABLED=false
IMAGE_GENERATION_ENABLED=false
BROWSER_ENABLED=false
EOF

chmod 600 ~/.hermes/.env
touch ~/.hermes/config.yaml
```

### Step 9 — 启动

```bash
# 交互式 CLI
hermes

# API Server
hermes gateway
```

---

## 下载地址汇总（Linux 版）

| 资源 | Linux 下载地址 | 文件名 | 大小 |
|------|---------------|--------|------|
| **uv (x86_64 Linux)** | https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-unknown-linux-gnu.tar.gz | uv-x86_64-unknown-linux-gnu.tar.gz | ~15MB |
| **uv (aarch64 Linux)** | https://github.com/astral-sh/uv/releases/latest/download/uv-aarch64-unknown-linux-gnu.tar.gz | uv-aarch64-unknown-linux-gnu.tar.gz | ~15MB |
| **Node.js v22 (x64 Linux)** | https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-x64.tar.xz | node-v22.14.0-linux-x64.tar.xz | ~30MB |
| **Node.js v22 (arm64 Linux)** | https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-arm64.tar.xz | node-v22.14.0-linux-arm64.tar.xz | ~30MB |
| **Hermes 源码** | https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.zip | hermes-agent-main.zip | ~50MB |
| **Python 3.11 (Windows 临时用)** | https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe | python-3.11.9-amd64.exe | ~27MB |

### 系统依赖（优先从离线服务器内网 apt 源安装）

如果必须从 Windows 下载 .deb，搜索地址：
- https://packages.ubuntu.com/jammy/git
- https://packages.ubuntu.com/jammy/python3.11
- https://packages.ubuntu.com/jammy/build-essential
- 等等

---

## 一键 PowerShell 脚本

### 脚本：Windows 收集 Linux 文件

保存为 `collect-for-linux.ps1`：

```powershell
#Requires -Version 5.1
param(
    [string]$OutputDir = "C:\temp\hermes-offline-linux"
)

$ErrorActionPreference = "Stop"
Write-Host "=== Windows → Linux 离线文件收集 ===" -ForegroundColor Cyan
Write-Host "将在 Windows 上下载 Linux 专用的文件" -ForegroundColor Yellow
Write-Host ""

# 确认架构
Write-Host "离线服务器架构？"
Write-Host "1. x86_64 (Intel/AMD，大多数服务器)"
Write-Host "2. aarch64 (ARM64，如 AWS Graviton)"
$archChoice = Read-Host "输入 1 或 2"

if ($archChoice -eq "2") {
    $uvArch = "aarch64"
    $nodeArch = "arm64"
} else {
    $uvArch = "x86_64"
    $nodeArch = "x64"
}

# 创建目录
New-Item -ItemType Directory -Force -Path "$OutputDir\binaries" | Out-Null
New-Item -ItemType Directory -Force -Path "$OutputDir\src" | Out-Null
New-Item -ItemType Directory -Force -Path "$OutputDir\python-packages" | Out-Null
New-Item -ItemType Directory -Force -Path "$OutputDir\system-packages" | Out-Null
New-Item -ItemType Directory -Force -Path "$OutputDir\tools" | Out-Null

# 1. 临时安装 Python
Write-Host "[1/6] 临时安装 Python 3.11..." -ForegroundColor Green
Invoke-WebRequest -Uri "https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe" -OutFile "$OutputDir\tools\python-installer.exe"
Start-Process -FilePath "$OutputDir\tools\python-installer.exe" -ArgumentList "/quiet", "InstallAllUsers=0", "PrependPath=1", "TargetDir=$OutputDir\tools\python311" -Wait

# 2. 下载 Linux uv
Write-Host "[2/6] 下载 uv for Linux ($uvArch)..." -ForegroundColor Green
Invoke-WebRequest -Uri "https://github.com/astral-sh/uv/releases/latest/download/uv-${uvArch}-unknown-linux-gnu.tar.gz" -OutFile "$OutputDir\binaries\uv-linux.tar.gz"

# 3. 下载 Linux Node.js
Write-Host "[3/6] 下载 Node.js for Linux ($nodeArch)..." -ForegroundColor Green
Invoke-WebRequest -Uri "https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-${nodeArch}.tar.xz" -OutFile "$OutputDir\binaries\node-linux.tar.xz"

# 4. 下载 Hermes 源码
Write-Host "[4/6] 下载 Hermes 源码..." -ForegroundColor Green
Invoke-WebRequest -Uri "https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.zip" -OutFile "$OutputDir\src\hermes.zip"

# 5. 解析依赖
Write-Host "[5/6] 解析 Python 依赖清单..." -ForegroundColor Green
Expand-Archive -Path "$OutputDir\src\hermes.zip" -DestinationPath "$OutputDir\tools\src" -Force
Set-Location "$OutputDir\tools\src\hermes-agent-main"

& "$OutputDir\tools\python311\python.exe" -m venv "$OutputDir\tools\temp-venv"
& "$OutputDir\tools\temp-venv\Scripts\pip.exe" install -e ".[all]"
& "$OutputDir\tools\temp-venv\Scripts\pip.exe" freeze | Out-File "$OutputDir\requirements-all.txt" -Encoding utf8

# 6. 下载 Linux 版 whl
Write-Host "[6/6] 下载 Linux 版 Python whl（约 3-8 分钟）..." -ForegroundColor Green

# 先尝试严格模式（只下载二进制 whl）
Write-Host "  尝试下载 manylinux 版 whl..." -ForegroundColor Gray
& "$OutputDir\tools\temp-venv\Scripts\pip.exe" download `
  -r "$OutputDir\requirements-all.txt" `
  -d "$OutputDir\python-packages" `
  --platform manylinux2014_${uvArch} `
  --python-version 3.11 `
  --abi cp311 `
  --only-binary :all: 2>$null

if ($LASTEXITCODE -ne 0) {
    Write-Host "  严格模式失败，切换到宽松模式（允许 sdist）..." -ForegroundColor Yellow
    & "$OutputDir\tools\temp-venv\Scripts\pip.exe" download `
      -r "$OutputDir\requirements-all.txt" `
      -d "$OutputDir\python-packages" `
      --platform manylinux2014_${uvArch} `
      --python-version 3.11 `
      --abi cp311
}

# 清理
Remove-Item -Recurse -Force "$OutputDir\tools\python311"
Remove-Item -Recurse -Force "$OutputDir\tools\temp-venv"
Remove-Item -Recurse -Force "$OutputDir\tools\src"
Remove-Item -Force "$OutputDir\tools\python-installer.exe"

# 验证
$whlCount = (Get-ChildItem "$OutputDir\python-packages\*.whl").Count
$winWhl = Get-ChildItem "$OutputDir\python-packages\*.whl" | Where-Object { $_.Name -match "win_amd64" }

Write-Host ""
Write-Host "=== 收集完成 ===" -ForegroundColor Green
Write-Host "whl 数量: $whlCount" -ForegroundColor Cyan
if ($winWhl) {
    Write-Host "⚠ 警告: 发现 Windows 版 whl，数量: $($winWhl.Count)" -ForegroundColor Red
    Write-Host "  这些包需要在 Linux 上编译安装" -ForegroundColor Yellow
} else {
    Write-Host "✓ 未发现 Windows 版 whl" -ForegroundColor Green
}
Write-Host ""
Write-Host "打包命令:" -ForegroundColor Cyan
Write-Host "  Compress-Archive -Path '$OutputDir' -DestinationPath 'C:\temp\hermes-offline-linux.zip' -Force"
```

---

## 故障排查

### Q1: pip download 报错 "No matching distribution found for xxx"

**原因**: 某个包没有提供 `manylinux2014_x86_64` 版本的 whl。

**解决**: 去掉 `--only-binary :all:` 参数，允许下载 sdist（源码包）。离线服务器上需要有 `gcc`、`python3-dev` 等编译工具来安装这些 sdist。

```powershell
# 去掉 --only-binary :all:
& "$collectDir\tools\temp-venv\Scripts\pip.exe" download `
  -r "$collectDir\requirements-all.txt" `
  -d "$collectDir\python-packages" `
  --platform manylinux2014_x86_64 `
  --python-version 3.11 `
  --abi cp311
```

### Q2: 离线服务器 architecture 不同

Windows 收集时用了 `--platform manylinux2014_x86_64`，但离线服务器是 **aarch64** (ARM64)。

**解决**: 收集时改为 `--platform manylinux2014_aarch64`，并且下载 `uv-aarch64-unknown-linux-gnu.tar.gz` 和 `node-v22.14.0-linux-arm64.tar.xz`。

### Q3: 离线服务器没有内网 apt 源，也没有 .deb 包

**最棘手的情况**。

**方案 A**: 在有网机器上启动一个临时 Docker 容器下载 .deb：

```powershell
# Windows 上安装 Docker Desktop，然后执行:
# docker run --rm -v C:\temp\hermes-offline-linux\system-packages:/out ubuntu:22.04 bash -c "apt-get update && apt-get download -o /out git python3.11 python3.11-venv build-essential libffi-dev libssl-dev ripgrep ffmpeg ca-certificates"
```

**方案 B**: 手动从 Ubuntu 软件包网站下载每个 .deb（ tedious，不推荐）。

**方案 C**: 离线服务器上源码编译安装 Python 3.11（见附录 C）。

### Q4: 打包后的 zip 太大，超过 4GB

FAT32 格式的 U 盘最大只支持 4GB 文件。

**解决**: 
- 格式化为 exFAT 或 NTFS
- 或者不压缩，直接用 `robocopy` 复制文件夹

```powershell
# 不压缩，直接复制文件夹到 U 盘
robocopy "C:\temp\hermes-offline-linux" "D:\hermes-offline-linux" /E /COPY:DAT
```

### Q5: 离线服务器提示 " ELF file OS ABI invalid"

**原因**: Windows 上误下载了 Windows 版的 uv/Node.js，或者 Linux 版下载成了 Windows 版。

**检查**:
```bash
# Linux 上执行
file /usr/local/bin/uv
# 应输出: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux)...
# ❌ 不应输出: PE32+ executable (console) x86-64, for MS Windows
```

---

## 附录

### 附录 A：pip download 跨平台参数详解

| 参数 | 说明 |
|------|------|
| `--platform` | 目标平台，`manylinux2014_x86_64` / `manylinux2014_aarch64` / `win_amd64` |
| `--python-version` | Python 版本，`3.11` |
| `--abi` | ABI 标签，`cp311` (CPython 3.11) |
| `--only-binary :all:` | 只下载 whl，不下载 sdist |
| `--implementation` | Python 实现，`cp` (CPython) |

### 附录 B：纯 Python 包 vs 平台特定包

| 文件名模式 | 类型 | 跨平台？ |
|-----------|------|---------|
| `xxx-py3-none-any.whl` | 纯 Python | ✅ 是（Windows/Linux/Mac 通用） |
| `xxx-cp311-cp311-manylinux*.whl` | 平台特定 | ❌ 只能 Linux 用 |
| `xxx-cp311-cp311-win_amd64.whl` | 平台特定 | ❌ 只能 Windows 用 |
| `xxx-*.tar.gz` | sdist 源码 | ⚠️ 需编译（离线服务器要有 gcc） |

### 附录 C：离线服务器无 apt 源时的 Python 源码编译方案

如果离线服务器既没有内网 apt 源，也没有 .deb 包：

```bash
# 在离线服务器上，从源码编译 Python 3.11（需要 gcc，通常预装）

# 1. 下载 Python 3.11 源码（在有网机器上下载后传来）
# https://www.python.org/ftp/python/3.11.9/Python-3.11.9.tgz

# 2. 解压编译
tar -xzf Python-3.11.9.tgz
cd Python-3.11.9
./configure --prefix=/opt/python311
make -j$(nproc)
sudo make install

# 3. 使用
/opt/python311/bin/python3 --version
```

### 附录 D：架构确认

**有网 Windows 机器执行**：
```powershell
# 查看 Windows 架构（几乎总是 x64）
$env:PROCESSOR_ARCHITECTURE
# AMD64
```

**离线 Linux 服务器执行**：
```bash
# 查看 Linux 架构
uname -m
# x86_64    → 下载 x86_64 / x64 版本
# aarch64   → 下载 aarch64 / arm64 版本
```

---

*本手册解决 "Windows 有网 + Linux 离线" 的跨平台离线安装问题。*  
*核心要点：pip 支持跨平台下载，但必须显式指定 `--platform` 参数。*
