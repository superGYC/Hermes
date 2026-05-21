# Hermes Agent Windows 完全离线安装手册

**版本**: v1.0 — Windows 专属版  
**适用场景**: Windows 10/11 64位，无外网，无虚拟机  
**核心思路**: 有网 Windows 机器 = 文件仓库；离线 Windows 机器 = 全部安装  
**日期**: 2026-05-21

---

## 目录

1. [和 Linux 版的区别](#和-linux-版的区别)
2. [阶段一：有网 Windows — 文件收集](#阶段一有网-windows--文件收集)
3. [阶段二：传输到离线 Windows](#阶段二传输到离线-windows)
4. [阶段三：离线 Windows — 全部安装](#阶段三离线-windows--全部安装)
5. [验证与启动](#验证与启动)
6. [下载地址汇总](#下载地址汇总)
7. [一键 PowerShell 脚本](#一键-powershell-脚本)
8. [故障排查](#故障排查)
9. [附录](#附录)

---

## 和 Linux 版的区别

| 项目 | Linux (v3.0 手册) | **Windows (本手册)** |
|------|-------------------|-------------------|
| 系统依赖 | `.deb` / `.rpm` 包 | **无**（Windows 不需要） |
| Python | `apt install python3.11` | **python.org 下载 `.exe`** |
| uv | `tar.gz` 解压到 `/usr/local/bin` | **`.zip` 解压到 `C:\tools\uv\`** |
| Node.js | `tar.xz` 解压到 `/opt/node` | **`.zip` 解压到 `C:\tools\node\`** |
| 命令入口 | `/usr/local/bin/hermes` (bash) | **`C:\tools\hermes.bat` (CMD)** |
| 配置目录 | `~/.hermes/` | **`%USERPROFILE%\.hermes\`** |
| Python whl | `manylinux2014_x86_64` | **`win_amd64`** |
| PATH 设置 | `~/.bashrc` | **系统环境变量 / PowerShell $env** |

**核心变化**：Windows 上不需要 "系统依赖包" 这一层，但需要正确配置环境变量（PATH）。

---

## 前置要求

### 有网机器
- Windows 10/11 64位，能访问互联网
- 磁盘空间：3GB 可用
- 已安装：PowerShell 5.1+（Windows 自带）

### 离线机器
- Windows 10/11 64位，无外网
- 磁盘空间：3GB 可用
- 已接收收集包

---

## 阶段一：有网 Windows — 文件收集

**目标**：下载所有需要的文件，**不在本机安装 Hermes**。

### Step 1 — 创建收集目录

打开 **PowerShell**（按 `Win + X`，选 "Windows PowerShell" 或 "终端"）：

```powershell
New-Item -ItemType Directory -Force -Path "C:\temp\hermes-offline\binaries"
New-Item -ItemType Directory -Force -Path "C:\temp\hermes-offline\src"
New-Item -ItemType Directory -Force -Path "C:\temp\hermes-offline\python-packages"
New-Item -ItemType Directory -Force -Path "C:\temp\hermes-offline\tools"

Set-Location C:\temp\hermes-offline
```

### Step 2 — 下载 Python 3.11 for Windows

```powershell
# 下载 Python 3.11.9 64位安装程序（约 27MB）
# 如果有更新的 3.11.x，替换版本号即可
Invoke-WebRequest -Uri "https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe" -OutFile "C:\temp\hermes-offline\binaries\python-3.11.9-amd64.exe"

# 验证文件存在
Get-Item "C:\temp\hermes-offline\binaries\python-3.11.9-amd64.exe"
```

> 💡 **如果官网下载慢**，也可以从 [Python Releases for Windows](https://www.python.org/downloads/windows/) 页面手动点击下载。

### Step 3 — 下载 uv for Windows

```powershell
# 下载 uv Windows 版（约 15MB，.zip 格式）
Invoke-WebRequest -Uri "https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-pc-windows-msvc.zip" -OutFile "C:\temp\hermes-offline\binaries\uv.zip"

# 验证
Expand-Archive -Path "C:\temp\hermes-offline\binaries\uv.zip" -DestinationPath "C:\temp\hermes-offline\tools\uv-temp" -Force
Get-ChildItem "C:\temp\hermes-offline\tools\uv-temp"
# 应看到: uv.exe, uvx.exe

# 删除临时解压，只保留 zip
Remove-Item -Recurse -Force "C:\temp\hermes-offline\tools\uv-temp"
```

### Step 4 — 下载 Node.js v22 for Windows

```powershell
# 下载 Node.js v22 64位（约 30MB，.zip 便携版）
Invoke-WebRequest -Uri "https://nodejs.org/dist/v22.14.0/node-v22.14.0-win-x64.zip" -OutFile "C:\temp\hermes-offline\binaries\node.zip"

# 验证
Expand-Archive -Path "C:\temp\hermes-offline\binaries\node.zip" -DestinationPath "C:\temp\hermes-offline\tools\node-temp" -Force
Get-ChildItem "C:\temp\hermes-offline\tools\node-temp\node-v22.14.0-win-x64"
# 应看到: node.exe, npm.cmd, npx.cmd ...

# 删除临时解压
Remove-Item -Recurse -Force "C:\temp\hermes-offline\tools\node-temp"
```

### Step 5 — 下载 Hermes 源码

```powershell
# 下载源码 zip（约 50MB，不需要安装 Git）
Invoke-WebRequest -Uri "https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.zip" -OutFile "C:\temp\hermes-offline\src\hermes.zip"

# 验证
Expand-Archive -Path "C:\temp\hermes-offline\src\hermes.zip" -DestinationPath "C:\temp\hermes-offline\tools\src-temp" -Force
Get-ChildItem "C:\temp\hermes-offline\tools\src-temp"
# 应看到: hermes-agent-main 文件夹

# 删除临时解压
Remove-Item -Recurse -Force "C:\temp\hermes-offline\tools\src-temp"
```

### Step 6 — 临时安装 Python + uv，解析并下载 Python 依赖（核心步骤）

**这是最难的一步**：需要在有网机器上临时装一遍 Python 和 uv，让 uv 解析 Hermes 的 218 个依赖，然后把所有 `.whl` 文件下载下来。

#### 6.1 临时安装 Python 3.11（收集用）

```powershell
# 静默安装 Python 3.11 到临时目录
# 参数说明:
#   /quiet          = 静默安装，不弹窗
#   /passive        = 只显示进度条
#   InstallAllUsers = 为所有用户安装
#   PrependPath     = 添加到 PATH
#   TargetDir       = 指定安装目录

Start-Process -FilePath "C:\temp\hermes-offline\binaries\python-3.11.9-amd64.exe" `
  -ArgumentList "/quiet", "InstallAllUsers=0", "PrependPath=1", "TargetDir=C:\temp\hermes-offline\tools\python311" `
  -Wait

# 验证
C:\temp\hermes-offline\tools\python311\python.exe --version
# 应输出: Python 3.11.9
```

#### 6.2 临时安装 uv（收集用）

```powershell
# 解压 uv 到临时目录
Expand-Archive -Path "C:\temp\hermes-offline\binaries\uv.zip" -DestinationPath "C:\temp\hermes-offline\tools\uv" -Force

# 验证
C:\temp\hermes-offline\tools\uv\uv.exe --version
# 应输出: uv x.x.x
```

#### 6.3 创建临时 venv，解析依赖

```powershell
# 进入临时源码目录
New-Item -ItemType Directory -Force -Path "C:\temp\hermes-offline\tools\src"
Expand-Archive -Path "C:\temp\hermes-offline\src\hermes.zip" -DestinationPath "C:\temp\hermes-offline\tools\src" -Force

Set-Location "C:\temp\hermes-offline\tools\src\hermes-agent-main"

# 创建临时 venv
$env:PATH = "C:\temp\hermes-offline\tools\uv;" + $env:PATH
uv venv "C:\temp\hermes-offline\tools\temp-venv" --python "C:\temp\hermes-offline\tools\python311\python.exe"

# 设置虚拟环境变量
$env:VIRTUAL_ENV = "C:\temp\hermes-offline\tools\temp-venv"

# 安装 Hermes 到临时 venv（让 uv 解析并下载所有 whl）
uv pip install -e ".[all]"

# 安装完成后，导出依赖清单
C:\temp\hermes-offline\tools\temp-venv\Scripts\pip.exe freeze | Out-File "C:\temp\hermes-offline\requirements-all.txt" -Encoding utf8

# 查看数量
(Get-Content "C:\temp\hermes-offline\requirements-all.txt").Count
# 应输出: 200+
```

#### 6.4 下载所有 Windows 版 .whl 文件

```powershell
# 使用 pip download 下载 Windows 专用 whl 文件
C:\temp\hermes-offline\tools\temp-venv\Scripts\pip.exe download `
  -r "C:\temp\hermes-offline\requirements-all.txt" `
  -d "C:\temp\hermes-offline\python-packages" `
  --only-binary :all: `
  --platform win_amd64 `
  --python-version 3.11

# 验证结果
(Get-ChildItem "C:\temp\hermes-offline\python-packages\*.whl").Count
# 应输出: 200+ 个文件
```

> ⚠️ **关键提醒**：Windows 的 `.whl` 和 Linux 的 `.whl` **不通用**。必须在 Windows 机器上执行这步，下载 `win_amd64` 平台的 whl。

#### 6.5 清理临时环境

```powershell
# 删除临时 Python、uv、venv（有网机器上不留下任何 Hermes 痕迹）
Remove-Item -Recurse -Force "C:\temp\hermes-offline\tools\python311"
Remove-Item -Recurse -Force "C:\temp\hermes-offline\tools\uv"
Remove-Item -Recurse -Force "C:\temp\hermes-offline\tools\temp-venv"
Remove-Item -Recurse -Force "C:\temp\hermes-offline\tools\src"

# 如果之前把 Python 安装到了系统，可能还需要卸载（可选）
# 但因为我们指定了 TargetDir 到临时目录，不会影响系统
```

### Step 7 — 下载 ffmpeg for Windows（可选，推荐）

Hermes 的媒体处理功能需要 ffmpeg。

```powershell
# 下载 ffmpeg 便携版（约 150MB）
Invoke-WebRequest -Uri "https://github.com/BtbN/FFmpeg-Builds/releases/download/latest/ffmpeg-master-latest-win64-gpl.zip" -OutFile "C:\temp\hermes-offline\binaries\ffmpeg.zip"

# 验证
Expand-Archive -Path "C:\temp\hermes-offline\binaries\ffmpeg.zip" -DestinationPath "C:\temp\hermes-offline\tools\ffmpeg-temp" -Force
Get-ChildItem "C:\temp\hermes-offline\tools\ffmpeg-temp"
# 应看到: ffmpeg-master-latest-win64-gpl 文件夹

# 删除临时解压
Remove-Item -Recurse -Force "C:\temp\hermes-offline\tools\ffmpeg-temp"
```

### Step 8 — 打包所有文件

```powershell
Set-Location C:\temp

# 使用 Compress-Archive（PowerShell 自带）
Compress-Archive -Path "hermes-offline" -DestinationPath "hermes-offline-windows.zip" -Force

# 查看大小
Get-Item "C:\temp\hermes-offline-windows.zip" | Select-Object Name, @{Name="Size(MB)";Expression={[math]::Round($_.Length/1MB,2)}}
# 预期: 800MB - 1.5GB
```

> 💡 **如果 zip 太大**，可以分卷压缩，或者直接用文件夹复制到移动硬盘。

### 阶段一验证清单

| 检查项 | PowerShell 命令 | 预期结果 |
|--------|----------------|----------|
| Python 安装包 | `Get-Item C:\temp\hermes-offline\binaries\python*.exe` | 文件存在 |
| uv 压缩包 | `Get-Item C:\temp\hermes-offline\binaries\uv.zip` | 文件存在 |
| Node.js 压缩包 | `Get-Item C:\temp\hermes-offline\binaries\node.zip` | 文件存在 |
| Hermes 源码 | `Get-Item C:\temp\hermes-offline\src\hermes.zip` | 文件存在 |
| Python whl | `(Get-ChildItem C:\temp\hermes-offline\python-packages\*.whl).Count` | >200 |
| 依赖清单 | `Get-Item C:\temp\hermes-offline\requirements-all.txt` | 文件存在 |

---

## 阶段二：传输到离线 Windows

### 方法 A — USB / 移动硬盘

```powershell
# 有网机器（PowerShell）
Copy-Item -Path "C:\temp\hermes-offline-windows.zip" -Destination "D:\hermes-offline-windows.zip" -Force
# D: 是 U 盘盘符，根据实际情况调整

# 离线机器（插入 U 盘后）
Copy-Item -Path "E:\hermes-offline-windows.zip" -Destination "C:\temp\hermes-offline-windows.zip" -Force
# E: 是 U 盘在离线机器上的盘符
```

### 方法 B — 局域网文件共享

```powershell
# 有网机器: 复制到共享文件夹
Copy-Item -Path "C:\temp\hermes-offline-windows.zip" -Destination "\\fileserver\shared\hermes-offline-windows.zip" -Force

# 离线机器: 从共享复制
Copy-Item -Path "\\fileserver\shared\hermes-offline-windows.zip" -Destination "C:\temp\hermes-offline-windows.zip" -Force
```

### 方法 C — 直接复制文件夹（不压缩）

如果 zip 太大不好处理，直接复制整个文件夹：

```powershell
# 有网机器
robocopy "C:\temp\hermes-offline" "D:\hermes-offline" /E /COPY:DAT

# 离线机器
robocopy "E:\hermes-offline" "C:\temp\hermes-offline" /E /COPY:DAT
```

---

## 阶段三：离线 Windows — 全部安装

**目标**：在完全无外网的 Windows 机器上执行 Hermes 的完整安装。

### Step 1 — 解压收集包

```powershell
# 解压到 C:\temp
Set-Location C:\temp
Expand-Archive -Path "hermes-offline-windows.zip" -DestinationPath "." -Force

# 确认结构
Get-ChildItem "C:\temp\hermes-offline"
# 应看到: binaries, python-packages, src, requirements-all.txt
```

### Step 2 — 安装 Python 3.11（离线）

```powershell
Set-Location "C:\temp\hermes-offline\binaries"

# 静默安装 Python 3.11 到 C:\Python311
Start-Process -FilePath "python-3.11.9-amd64.exe" `
  -ArgumentList "/quiet", "InstallAllUsers=1", "PrependPath=1", "TargetDir=C:\Python311" `
  -Wait

# 验证（需要新开 PowerShell 窗口读取新的 PATH）
C:\Python311\python.exe --version
# 应输出: Python 3.11.9
```

> 💡 **如果离线机器上有 Python 3.11 了**，跳过这步，确认版本即可。

### Step 3 — 安装 uv（离线）

```powershell
# 创建 uv 目录
New-Item -ItemType Directory -Force -Path "C:\tools\uv"

# 解压
Expand-Archive -Path "C:\temp\hermes-offline\binaries\uv.zip" -DestinationPath "C:\tools\uv" -Force

# 添加到用户 PATH（永久）
$currentPath = [Environment]::GetEnvironmentVariable("Path", "User")
if ($currentPath -notlike "*C:\tools\uv*") {
    [Environment]::SetEnvironmentVariable("Path", $currentPath + ";C:\tools\uv", "User")
}

# 临时添加到当前会话 PATH
$env:Path += ";C:\tools\uv"

# 验证
uv --version
# 应输出: uv x.x.x
```

### Step 4 — 安装 Node.js v22（离线）

```powershell
# 创建 Node.js 目录
New-Item -ItemType Directory -Force -Path "C:\tools\node"

# 解压
Expand-Archive -Path "C:\temp\hermes-offline\binaries\node.zip" -DestinationPath "C:\tools\node-temp" -Force

# node-v22.14.0-win-x64 是解压后的子文件夹，需要把里面的内容移到 C:\tools\node
Move-Item -Path "C:\tools\node-temp\node-v22.14.0-win-x64\*" -Destination "C:\tools\node" -Force
Remove-Item -Recurse -Force "C:\tools\node-temp"

# 添加到用户 PATH
$currentPath = [Environment]::GetEnvironmentVariable("Path", "User")
if ($currentPath -notlike "*C:\tools\node*") {
    [Environment]::SetEnvironmentVariable("Path", $currentPath + ";C:\tools\node", "User")
}

# 临时添加到当前会话
$env:Path += ";C:\tools\node"

# 验证
node --version
# 应输出: v22.14.0

npm --version
# 应输出: 10.x.x
```

### Step 5 — 解压 Hermes 源码

```powershell
# 解压到 C:\
Expand-Archive -Path "C:\temp\hermes-offline\src\hermes.zip" -DestinationPath "C:\" -Force

# 重命名
Rename-Item -Path "C:\hermes-agent-main" -NewName "hermes-agent" -Force

# 验证
Get-ChildItem "C:\hermes-agent"
# 应看到: pyproject.toml, uv.lock, scripts/, gateway/ ...
```

### Step 6 — 创建虚拟环境并从本地 whl 安装（核心步骤）

```powershell
Set-Location "C:\hermes-agent"

# 1. 创建虚拟环境
uv venv "C:\hermes-agent\venv" --python "C:\Python311\python.exe"

# 2. 设置环境变量
$env:VIRTUAL_ENV = "C:\hermes-agent\venv"

# 3. ████████████████████████████████████████████████████████████
#    关键：从本地 whl 文件安装（--no-index = 不连外网）
# ████████████████████████████████████████████████████████████

uv pip install `
  --no-index `
  --find-links "C:\temp\hermes-offline\python-packages" `
  -e ".[all]"

# 安装过程会看到类似输出：
# Resolved 218 packages in 0.01s
# Downloaded 0 packages   ← 0 是因为从本地读取
# Installed 218 packages  ← 全部来自本地
```

### Step 7 — 创建 hermes 命令（批处理文件）

```powershell
# 创建 C:\tools\hermes.bat
$batContent = @'
@echo off
set PYTHONPATH=
set PYTHONHOME=
set UV_NO_CONFIG=1
"C:\hermes-agent\venv\Scripts\python.exe" -m hermes_cli.main %*
'@

# 写入文件
$batContent | Out-File -FilePath "C:\tools\hermes.bat" -Encoding ascii

# 添加到 PATH（如果 C:\tools 还没在 PATH 里）
$currentPath = [Environment]::GetEnvironmentVariable("Path", "User")
if ($currentPath -notlike "*C:\tools*") {
    [Environment]::SetEnvironmentVariable("Path", $currentPath + ";C:\tools", "User")
}

# 临时添加到当前会话
$env:Path += ";C:\tools"

# 验证
hermes --version
# 应输出: Hermes Agent x.x.x
```

### Step 8 — 安装 ffmpeg（可选）

```powershell
# 解压 ffmpeg
Expand-Archive -Path "C:\temp\hermes-offline\binaries\ffmpeg.zip" -DestinationPath "C:\tools" -Force

# 重命名为简单路径
Rename-Item -Path "C:\tools\ffmpeg-master-latest-win64-gpl" -NewName "ffmpeg" -Force

# 添加到 PATH
$currentPath = [Environment]::GetEnvironmentVariable("Path", "User")
if ($currentPath -notlike "*C:\tools\ffmpeg\bin*") {
    [Environment]::SetEnvironmentVariable("Path", $currentPath + ";C:\tools\ffmpeg\bin", "User")
}

# 验证
ffmpeg -version
```

### Step 9 — 创建配置目录和 .env

```powershell
# 创建数据目录
$hermesHome = Join-Path $env:USERPROFILE ".hermes"
New-Item -ItemType Directory -Force -Path $hermesHome
New-Item -ItemType Directory -Force -Path (Join-Path $hermesHome "cron")
New-Item -ItemType Directory -Force -Path (Join-Path $hermesHome "sessions")
New-Item -ItemType Directory -Force -Path (Join-Path $hermesHome "logs")
New-Item -ItemType Directory -Force -Path (Join-Path $hermesHome "memories")
New-Item -ItemType Directory -Force -Path (Join-Path $hermesHome "skills")

# 创建 .env（只用本地 Ollama）
$envContent = @"
# ===== LLM 配置（本地 Ollama） =====
OPENAI_BASE_URL=http://127.0.0.1:11434/v1
OPENAI_API_KEY=ollama
OPENAI_MODEL=llama3.1:70b

# ===== API Server（供外部 Web 调用） =====
API_SERVER_ENABLED=true
API_SERVER_KEY=your-offline-production-key

# ===== 禁用所有需要外网的功能 =====
WEB_SEARCH_ENABLED=false
TTS_ENABLED=false
STT_ENABLED=false
IMAGE_GENERATION_ENABLED=false
BROWSER_ENABLED=false
"@

$envContent | Out-File -FilePath (Join-Path $hermesHome ".env") -Encoding utf8

# 创建空配置文件
New-Item -ItemType File -Force -Path (Join-Path $hermesHome "config.yaml")
```

---

## 验证与启动

### 验证安装

```powershell
# 验证 1: hermes 命令
hermes --version

# 验证 2: Python 包数量
(Get-ChildItem "C:\hermes-agent\venv\Lib\site-packages").Count
# 应输出: 200+

# 验证 3: uv
uv --version

# 验证 4: Node.js
node --version
```

### 启动 Hermes

```powershell
# 方式 A: 交互式 CLI（前台，适合调试）
hermes

# 方式 B: API Server 模式（后台常驻）
hermes gateway

# 方式 C: 用 PowerShell 后台运行
Start-Process -FilePath "hermes" -ArgumentList "gateway" -WindowStyle Hidden
```

### 验证 API

```powershell
# 健康检查
Invoke-RestMethod -Uri "http://localhost:8642/health" | ConvertTo-Json

# 列出模型
Invoke-RestMethod -Uri "http://localhost:8642/v1/models" -Headers @{"Authorization"="Bearer your-offline-production-key"} | ConvertTo-Json

# 测试对话（需要 Ollama 已运行）
$body = @{
    model = "llama3.1:70b"
    messages = @(@{role="user"; content="Hello"})
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:8642/v1/chat/completions" `
  -Method POST `
  -Headers @{"Authorization"="Bearer your-offline-production-key"; "Content-Type"="application/json"} `
  -Body $body | ConvertTo-Json
```

---

## 下载地址汇总

| 资源 | Windows 下载地址 | 文件名 | 大小 |
|------|-----------------|--------|------|
| **Python 3.11** | https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe | python-3.11.9-amd64.exe | ~27MB |
| **uv (x64)** | https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-pc-windows-msvc.zip | uv-x86_64-pc-windows-msvc.zip | ~15MB |
| **Node.js v22 (x64)** | https://nodejs.org/dist/v22.14.0/node-v22.14.0-win-x64.zip | node-v22.14.0-win-x64.zip | ~30MB |
| **Hermes 源码** | https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.zip | hermes-agent-main.zip | ~50MB |
| **ffmpeg (x64)** | https://github.com/BtbN/FFmpeg-Builds/releases/download/latest/ffmpeg-master-latest-win64-gpl.zip | ffmpeg-master-latest-win64-gpl.zip | ~150MB |

> 💡 **版本号**：以上 URL 中的 `3.11.9`、`v22.14.0` 是示例。下载前请去官网确认最新稳定版，替换版本号即可。

---

## 一键 PowerShell 脚本

### 脚本 A：文件收集（有网 Windows 执行）

保存为 `collect-offline-windows.ps1`：

```powershell
#Requires -Version 5.1
param(
    [string]$OutputDir = "C:\temp\hermes-offline"
)

$ErrorActionPreference = "Stop"
Write-Host "=== Hermes Agent Windows 离线文件收集 ===" -ForegroundColor Cyan
Write-Host "注意：此脚本只下载文件，不在本机安装 Hermes" -ForegroundColor Yellow
Write-Host ""

# 创建目录
New-Item -ItemType Directory -Force -Path "$OutputDir\binaries" | Out-Null
New-Item -ItemType Directory -Force -Path "$OutputDir\src" | Out-Null
New-Item -ItemType Directory -Force -Path "$OutputDir\python-packages" | Out-Null
New-Item -ItemType Directory -Force -Path "$OutputDir\tools" | Out-Null

# 1. Python
Write-Host "[1/6] 下载 Python 3.11..."
Invoke-WebRequest -Uri "https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe" -OutFile "$OutputDir\binaries\python-3.11.9-amd64.exe"

# 2. uv
Write-Host "[2/6] 下载 uv..."
Invoke-WebRequest -Uri "https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-pc-windows-msvc.zip" -OutFile "$OutputDir\binaries\uv.zip"

# 3. Node.js
Write-Host "[3/6] 下载 Node.js..."
Invoke-WebRequest -Uri "https://nodejs.org/dist/v22.14.0/node-v22.14.0-win-x64.zip" -OutFile "$OutputDir\binaries\node.zip"

# 4. Hermes 源码
Write-Host "[4/6] 下载 Hermes 源码..."
Invoke-WebRequest -Uri "https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.zip" -OutFile "$OutputDir\src\hermes.zip"

# 5. ffmpeg
Write-Host "[5/6] 下载 ffmpeg..."
Invoke-WebRequest -Uri "https://github.com/BtbN/FFmpeg-Builds/releases/download/latest/ffmpeg-master-latest-win64-gpl.zip" -OutFile "$OutputDir\binaries\ffmpeg.zip"

# 6. Python 依赖
Write-Host "[6/6] 解析并下载 Python whl 包（需要 3-8 分钟）..."

# 临时安装 Python
Start-Process -FilePath "$OutputDir\binaries\python-3.11.9-amd64.exe" `
  -ArgumentList "/quiet", "InstallAllUsers=0", "PrependPath=1", "TargetDir=$OutputDir\tools\python311" -Wait

# 临时安装 uv
Expand-Archive -Path "$OutputDir\binaries\uv.zip" -DestinationPath "$OutputDir\tools\uv" -Force
$env:Path = "$OutputDir\tools\uv;" + $env:Path

# 临时 venv
Expand-Archive -Path "$OutputDir\src\hermes.zip" -DestinationPath "$OutputDir\tools\src" -Force
Set-Location "$OutputDir\tools\src\hermes-agent-main"

uv venv "$OutputDir\tools\temp-venv" --python "$OutputDir\tools\python311\python.exe"
$env:VIRTUAL_ENV = "$OutputDir\tools\temp-venv"
uv pip install -e ".[all]"

# 导出清单
& "$OutputDir\tools\temp-venv\Scripts\pip.exe" freeze | Out-File "$OutputDir\requirements-all.txt" -Encoding utf8

# 下载 whl
& "$OutputDir\tools\temp-venv\Scripts\pip.exe" download `
  -r "$OutputDir\requirements-all.txt" `
  -d "$OutputDir\python-packages" `
  --only-binary :all: `
  --platform win_amd64 `
  --python-version 3.11

# 清理
Remove-Item -Recurse -Force "$OutputDir\tools\python311"
Remove-Item -Recurse -Force "$OutputDir\tools\uv"
Remove-Item -Recurse -Force "$OutputDir\tools\temp-venv"
Remove-Item -Recurse -Force "$OutputDir\tools\src"

# 打包
Write-Host ""
Write-Host "=== 收集完成 ===" -ForegroundColor Green
Write-Host "打包命令:" -ForegroundColor Cyan
Write-Host "  Compress-Archive -Path '$OutputDir' -DestinationPath 'C:\temp\hermes-offline-windows.zip' -Force"
Write-Host ""
Write-Host "文件统计:" -ForegroundColor Cyan
Get-ChildItem $OutputDir -Recurse -File | Group-Object Directory | Select-Object Name, @{Name="Count";Expression={$_.Count}}, @{Name="Size(MB)";Expression={[math]::Round(($_.Group | Measure-Object Length -Sum).Sum/1MB,2)}}
```

### 脚本 B：离线安装（离线 Windows 执行）

保存为 `install-offline-windows.ps1`：

```powershell
#Requires -Version 5.1
param(
    [string]$SourceDir = "C:\temp\hermes-offline"
)

$ErrorActionPreference = "Stop"
Write-Host "=== Hermes Agent Windows 离线安装 ===" -ForegroundColor Cyan
Write-Host ""

if (-not (Test-Path $SourceDir)) {
    Write-Host "未找到 $SourceDir，请先解压收集包" -ForegroundColor Red
    exit 1
}

# 1. Python
Write-Host "[1/8] 安装 Python 3.11..."
Start-Process -FilePath "$SourceDir\binaries\python-3.11.9-amd64.exe" `
  -ArgumentList "/quiet", "InstallAllUsers=1", "PrependPath=1", "TargetDir=C:\Python311" -Wait

# 2. uv
Write-Host "[2/8] 安装 uv..."
New-Item -ItemType Directory -Force -Path "C:\tools\uv" | Out-Null
Expand-Archive -Path "$SourceDir\binaries\uv.zip" -DestinationPath "C:\tools\uv" -Force
$env:Path += ";C:\tools\uv"

# 3. Node.js
Write-Host "[3/8] 安装 Node.js..."
New-Item -ItemType Directory -Force -Path "C:\tools\node" | Out-Null
Expand-Archive -Path "$SourceDir\binaries\node.zip" -DestinationPath "C:\tools\node-temp" -Force
Move-Item -Path "C:\tools\node-temp\node-v22.14.0-win-x64\*" -Destination "C:\tools\node" -Force
Remove-Item -Recurse -Force "C:\tools\node-temp"
$env:Path += ";C:\tools\node"

# 4. ffmpeg
Write-Host "[4/8] 安装 ffmpeg..."
Expand-Archive -Path "$SourceDir\binaries\ffmpeg.zip" -DestinationPath "C:\tools" -Force
Rename-Item -Path "C:\tools\ffmpeg-master-latest-win64-gpl" -NewName "ffmpeg" -Force
$env:Path += ";C:\tools\ffmpeg\bin"

# 5. Hermes 源码
Write-Host "[5/8] 解压 Hermes 源码..."
Expand-Archive -Path "$SourceDir\src\hermes.zip" -DestinationPath "C:\" -Force
Rename-Item -Path "C:\hermes-agent-main" -NewName "hermes-agent" -Force

# 6. Python 依赖
Write-Host "[6/8] 安装 Python 依赖..."
Set-Location "C:\hermes-agent"
uv venv "C:\hermes-agent\venv" --python "C:\Python311\python.exe"
$env:VIRTUAL_ENV = "C:\hermes-agent\venv"
uv pip install --no-index --find-links "$SourceDir\python-packages" -e ".[all]"

# 7. hermes 命令
Write-Host "[7/8] 创建 hermes 命令..."
$batContent = "@echo off`nset PYTHONPATH=`nset PYTHONHOME=`nset UV_NO_CONFIG=1`n`"C:\hermes-agent\venv\Scripts\python.exe`" -m hermes_cli.main %*"
$batContent | Out-File -FilePath "C:\tools\hermes.bat" -Encoding ascii
$env:Path += ";C:\tools"

# 8. 配置
Write-Host "[8/8] 创建配置..."
$hermesHome = Join-Path $env:USERPROFILE ".hermes"
New-Item -ItemType Directory -Force -Path $hermesHome | Out-Null
"cron","sessions","logs","memories","skills" | ForEach-Object {
    New-Item -ItemType Directory -Force -Path (Join-Path $hermesHome $_) | Out-Null
}

$envContent = @"
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
"@
$envContent | Out-File -FilePath (Join-Path $hermesHome ".env") -Encoding utf8
New-Item -ItemType File -Force -Path (Join-Path $hermesHome "config.yaml") | Out-Null

# 永久添加 PATH
$currentPath = [Environment]::GetEnvironmentVariable("Path", "User")
foreach ($p in @("C:\tools\uv","C:\tools\node","C:\tools\ffmpeg\bin","C:\tools")) {
    if ($currentPath -notlike "*$p*") {
        $currentPath += ";$p"
    }
}
[Environment]::SetEnvironmentVariable("Path", $currentPath, "User")

Write-Host ""
Write-Host "=== 安装完成 ===" -ForegroundColor Green
Write-Host "验证命令:" -ForegroundColor Cyan
Write-Host "  hermes --version"
Write-Host "  uv --version"
Write-Host "  node --version"
Write-Host ""
Write-Host "启动命令:" -ForegroundColor Cyan
Write-Host "  hermes         (交互式 CLI)"
Write-Host "  hermes gateway (API Server)"
Write-Host ""
Write-Host "注意: 请重启 PowerShell 或重新登录以应用 PATH 更改" -ForegroundColor Yellow
```

---

## 故障排查

### Q1: Python 安装后 `python` 命令找不到

```powershell
# 确认安装路径
C:\Python311\python.exe --version

# 如果 PATH 没生效，手动添加
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Python311;C:\Python311\Scripts", "User")

# 重启 PowerShell
```

### Q2: `uv pip install --no-index` 提示某些包找不到

**原因**：Windows 和 Linux 的 `.whl` 不通用。如果在 Linux 机器上收集的 whl，Windows 上无法使用。

**解决**：必须在 **Windows 机器** 上执行收集脚本（Step 6），下载 `--platform win_amd64` 的 whl。

### Q3: `hermes` 命令提示不是可识别命令

```powershell
# 确认文件存在
Get-Item "C:\tools\hermes.bat"

# 确认 PATH
$env:Path

# 如果 C:\tools 不在 PATH 中
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\tools", "User")

# 重新加载 PATH（或重启 PowerShell）
$env:Path = [Environment]::GetEnvironmentVariable("Path", "Machine") + ";" + [Environment]::GetEnvironmentVariable("Path", "User")
```

### Q4: 提示缺少 `VCRUNTIME140.dll` 或 `MSVCP140.dll`

需要安装 **Visual C++ Redistributable**。

```powershell
# 在有网机器上下载
Invoke-WebRequest -Uri "https://aka.ms/vs/17/release/vc_redist.x64.exe" -OutFile "C:\temp\hermes-offline\binaries\vc_redist.x64.exe"

# 离线机器上安装
Start-Process -FilePath "C:\temp\hermes-offline\binaries\vc_redist.x64.exe" -ArgumentList "/quiet", "/norestart" -Wait
```

### Q5: Hermes 启动报错，提示缺少某个 Python 包

```powershell
# 检查 whl 数量
(Get-ChildItem "C:\temp\hermes-offline\python-packages\*.whl").Count
# 如果 < 200，说明收集不完整，需要在有网机器上重新执行收集
```

### Q6: Ollama 在 Windows 上离线安装

Ollama 也支持 Windows：

```powershell
# 有网机器下载
Invoke-WebRequest -Uri "https://ollama.com/download/OllamaSetup.exe" -OutFile "OllamaSetup.exe"

# 离线机器安装（双击或命令行）
Start-Process -FilePath "OllamaSetup.exe" -ArgumentList "/S" -Wait
```

模型下载和 Linux 类似，需要在有网机器上 `ollama pull` 后，将模型文件从 `C:\Users\<用户名>\.ollama\models\` 复制到离线机器同位置。

---

## 附录

### 附录 A：Python 静默安装参数

| 参数 | 说明 |
|------|------|
| `/quiet` | 完全静默，无 UI |
| `/passive` | 只显示进度条 |
| `InstallAllUsers=1` | 为所有用户安装（到 `C:\Python311`） |
| `InstallAllUsers=0` | 只给当前用户安装 |
| `PrependPath=1` | 自动添加到 PATH |
| `TargetDir=C:\Python311` | 指定安装目录 |
| `Include_doc=0` | 不安装文档（减少体积） |
| `Include_test=0` | 不安装测试套件 |
| `Include_launcher=1` | 安装 py.exe 启动器 |

### 附录 B：Windows vs Linux 路径对照

| 概念 | Linux | Windows |
|------|-------|---------|
| 用户主目录 | `~` | `%USERPROFILE%` 或 `$env:USERPROFILE` |
| 全局程序 | `/usr/local/bin/` | `C:\tools\` 或 `C:\Program Files\` |
| PATH 分隔符 | `:` | `;` |
| 路径分隔符 | `/` | `\` |
| 虚拟环境 | `venv/bin/python` | `venv\Scripts\python.exe` |
| 配置目录 | `~/.hermes/` | `C:\Users\<用户名>\.hermes\` |

### 附录 C：安装后文件结构（Windows）

```
C:\
├── hermes-agent\              # Hermes 源码 + venv
│   ├── venv\                   # Python 虚拟环境（218 个包）
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── scripts\
│   ├── gateway\
│   └── ...
├── Python311\                  # Python 3.11
│   ├── python.exe
│   └── ...
├── tools\                      # 工具链
│   ├── uv\                     # uv 包管理器
│   │   ├── uv.exe
│   │   └── uvx.exe
│   ├── node\                   # Node.js v22
│   │   ├── node.exe
│   │   ├── npm.cmd
│   │   └── ...
│   ├── ffmpeg\                 # ffmpeg
│   │   └── bin\ffmpeg.exe
│   └── hermes.bat              # Hermes 命令入口
└── ...

C:\Users\<用户名>\
└── .hermes\                    # 用户数据
    ├── .env                    # 环境变量（API Key 等）
    ├── config.yaml             # 主配置
    ├── SOUL.md
    ├── cron\                   # 定时任务
    ├── sessions\               # 会话记录
    ├── logs\                   # 日志
    ├── memories\               # 记忆
    └── skills\                 # 技能
```

---

*本手册基于 Hermes Agent 官方 install.sh、pyproject.toml 和 uv.lock 编写，适配 Windows 10/11 64位环境。*  
*所有版本号以官方最新发布为准，安装前请核实 URL 有效性。*
