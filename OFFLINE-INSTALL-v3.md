# Hermes Agent 完全离线安装手册 v3.0（纯文件收集版）

**版本**: v3.0  
**适用场景**: 服务器无外网，且**有网机器也不安装 Hermes**，只做文件中转  
**核心思路**: 有网机器 = 仓库；离线服务器 = 全部安装  
**日期**: 2026-05-20

---

## 目录

1. [思路说明](#思路说明)
2. [阶段一：有网机器 — 纯文件收集](#阶段一有网机器--纯文件收集)
3. [阶段二：传输到离线服务器](#阶段二传输到离线服务器)
4. [阶段三：离线服务器 — 全部安装](#阶段三离线服务器--全部安装)
5. [下载地址汇总](#下载地址汇总)
6. [一键脚本](#一键脚本)
7. [故障排查](#故障排查)
8. [附录](#附录)

---

## 思路说明

官方 install.sh 需要联网下载：
- uv 包管理器（~15MB）
- Python 3.11（~30MB）
- Node.js v22（~30MB）
- 218 个 Python 依赖包（~500MB-1GB）
- Hermes 源码（~50MB）

**传统方案的问题**：在有网机器上预装一遍，再打包整个 venv 带走。但这要求有网机器也完成安装，且打包体积大（含编译缓存、临时文件）。

**本方案的优势**：
- ✅ 有网机器**只下载原始文件**，不创建 venv、不运行 Hermes
- ✅ 离线服务器执行**完整安装**，与官方流程一致
- ✅ 传输体积更小（只传 .whl 文件，不含编译缓存）
- ✅ 离线服务器上可审计每个安装步骤

```
┌─────────────────────────────────────────────────────────────┐
│                      有网机器（可访问互联网）                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ 系统依赖.deb │  │ 二进制工具    │  │ Python whl 包       │ │
│  │ (apt下载)   │  │ (uv/node)    │  │ (pip download)     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│           │                │                │              │
│           └────────────────┴────────────────┘              │
│                          │                                  │
│                    ┌─────┴─────┐                            │
│                    │ 打包传输  │ ← USB/硬盘/内网文件服务器      │
│                    └─────┬─────┘                            │
└──────────────────────────┼──────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────┐
│                      离线服务器（无外网）                      │
│                          │                                   │
│  ┌───────────────────────┴───────────────────────────────┐ │
│  │  Step 1: 安装系统依赖 (.deb/.rpm)                      │ │
│  │  Step 2: 安装 uv 二进制                                  │ │
│  │  Step 3: 安装 Node.js 二进制                             │ │
│  │  Step 4: 解压 Hermes 源码                                │ │
│  │  Step 5: 创建 venv + 从本地 whl 安装所有依赖              │ │
│  │  Step 6: 创建 hermes 命令 + 配置 .env                    │ │
│  │  Step 7: 启动                                            │ │
│  └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 阶段一：有网机器 — 纯文件收集

**目标**：收集所有需要的文件，**不在本机安装 Hermes**。

### 前置要求

- 一台能访问互联网的 Linux 机器（Ubuntu 22.04 / Debian 12 / CentOS 9 均可）
- 磁盘空间：3GB 可用
- 已安装：`curl`, `unzip`

### Step 1 — 创建收集目录

```bash
mkdir -p /tmp/hermes-offline/{system-packages,python-packages,binaries,src}
cd /tmp/hermes-offline
pwd
# 输出: /tmp/hermes-offline
```

### Step 2 — 下载系统依赖包

#### Ubuntu / Debian 系统

```bash
cd /tmp/hermes-offline/system-packages

# 下载所有 .deb 包（不需要 sudo，只下载不安装）
apt-get download \
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

# 验证
cd /tmp/hermes-offline/system-packages
ls -la *.deb | wc -l
# 应输出: 12 或更多（取决于依赖数量）
```

#### CentOS / Rocky / RHEL 系统

```bash
cd /tmp/hermes-offline/system-packages

# 安装 yum-utils（用于 yumdownloader）
sudo dnf install -y yum-utils

# 下载所有 .rpm 包
yumdownloader \
  git \
  python3.11 \
  python3.11-devel \
  gcc \
  gcc-c++ \
  libffi-devel \
  openssl-devel \
  ripgrep \
  ffmpeg \
  ca-certificates

# 验证
ls -la *.rpm | wc -l
# 应输出: 12 或更多
```

### Step 3 — 下载 uv 包管理器

```bash
cd /tmp/hermes-offline/binaries

# x86_64 架构
curl -fsSL -o uv.tar.gz \
  https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-unknown-linux-gnu.tar.gz

# aarch64 (ARM64) 架构用这个替代：
# curl -fsSL -o uv.tar.gz \
#   https://github.com/astral-sh/uv/releases/latest/download/uv-aarch64-unknown-linux-gnu.tar.gz

# 验证（查看压缩包内容，不解压）
tar -tzf uv.tar.gz
# 应输出: uv, uvx
```

### Step 4 — 下载 Node.js v22

```bash
cd /tmp/hermes-offline/binaries

# x86_64
curl -fsSL -o node.tar.xz \
  https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-x64.tar.xz

# aarch64 (ARM64) 用这个替代：
# curl -fsSL -o node.tar.xz \
#   https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-arm64.tar.xz

# 验证（查看压缩包内容，不解压）
tar -tJf node.tar.xz | head -5
# 应输出: node-v22.14.0-linux-x64/bin/node ...
```

### Step 5 — 下载 Hermes 源码

```bash
cd /tmp/hermes-offline/src

# 下载源码 zip（不需要 git）
curl -fsSL -o hermes.zip \
  https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.zip

# 验证（测试解压，不实际解压）
unzip -t hermes.zip | tail -5
# 应输出: hermes-agent-main/pyproject.toml ... No errors detected
```

### Step 6 — 下载所有 Python 依赖包（核心步骤）

这是最复杂的部分。需要在有网机器上**临时**创建一个 venv，让 uv 解析出完整依赖树，然后下载所有 .whl 文件。

```bash
cd /tmp

# 1. 临时解压源码
unzip -q /tmp/hermes-offline/src/hermes.zip
cd hermes-agent-main

# 2. 临时安装 uv 到 /usr/local/bin（用完删除）
sudo tar -xzf /tmp/hermes-offline/binaries/uv.tar.gz -C /usr/local/bin/

# 3. 创建临时 venv（仅用于解析依赖，用完删除）
/usr/local/bin/uv venv /tmp/temp-venv --python 3.11
export VIRTUAL_ENV=/tmp/temp-venv

# 4. 安装 Hermes 到临时 venv（让 uv 解析并下载全部 218 个包）
#    注意：这步是在有网机器上临时安装，目的是让 uv 把 whl 文件下载到缓存中
/usr/local/bin/uv pip install -e ".[all]"

# 5. 导出完整依赖清单
/tmp/temp-venv/bin/pip freeze > /tmp/hermes-offline/requirements-all.txt

# 查看清单
cat /tmp/hermes-offline/requirements-all.txt | wc -l
# 应输出: 200+ 行
```

**关键步骤**：将临时 venv 中下载的所有 .whl 文件复制到收集目录。

```bash
# 6. 找到 uv 缓存目录中的所有 whl 文件
# uv 的缓存通常在 ~/.cache/uv/ 或 /tmp/ 下
mkdir -p /tmp/hermes-offline/python-packages

# 方法 A: 从临时 venv 复制已下载的 whl
find /tmp/temp-venv -name "*.whl" -exec cp {} /tmp/hermes-offline/python-packages/ \; 2>/dev/null || true

# 方法 B（更可靠）: 用 pip download 重新下载到指定目录
/tmp/temp-venv/bin/pip download \
  -r /tmp/hermes-offline/requirements-all.txt \
  -d /tmp/hermes-offline/python-packages/ \
  --only-binary :all: \
  --platform manylinux2014_x86_64 \
  --python-version 3.11

# 7. 验证结果
cd /tmp/hermes-offline/python-packages
ls *.whl | wc -l
# 应输出: 200+ 个文件

# 8. 清理临时环境（有网机器上不留下任何 Hermes 痕迹）
rm -rf /tmp/temp-venv /tmp/hermes-agent-main
sudo rm -f /usr/local/bin/uv /usr/local/bin/uvx
```

### Step 7 — 打包

```bash
cd /tmp

# 打包所有收集的文件
tar -czvf hermes-offline-pure.tar.gz hermes-offline/

# 查看大小
ls -lh /tmp/hermes-offline-pure.tar.gz
# 预期: 800MB - 1.5GB

# 查看内部结构
tar -tzf hermes-offline-pure.tar.gz | head -20
```

### 阶段一验证清单

| 检查项 | 命令 | 预期结果 |
|--------|------|----------|
| 系统依赖包 | `ls /tmp/hermes-offline/system-packages/ | wc -l` | >10 个文件 |
| uv 二进制 | `tar -tzf /tmp/hermes-offline/binaries/uv.tar.gz` | 包含 `uv`, `uvx` |
| Node.js | `tar -tJf /tmp/hermes-offline/binaries/node.tar.xz \| head -3` | 包含 `bin/node` |
| Hermes 源码 | `unzip -t /tmp/hermes-offline/src/hermes.zip` | No errors |
| Python whl | `ls /tmp/hermes-offline/python-packages/*.whl \| wc -l` | >200 个 |

---

## 阶段二：传输到离线服务器

### 方法 A — USB / 移动硬盘

```bash
# 在有网机器上
sudo cp /tmp/hermes-offline-pure.tar.gz /mnt/usb/

# 在离线服务器上
sudo cp /mnt/usb/hermes-offline-pure.tar.gz /opt/
```

### 方法 B — 内网文件服务器 / 堡垒机

```bash
# 有网机器 → 文件服务器
scp /tmp/hermes-offline-pure.tar.gz fileserver:/shared/packages/

# 离线服务器 ← 文件服务器
cp /shared/packages/hermes-offline-pure.tar.gz /opt/
```

### 方法 C — 直接硬盘挂载（物理搬运）

```bash
# 如果离线服务器没有 USB 接口，拆硬盘直连
sudo mount /dev/sdb1 /mnt/usb
cp /mnt/usb/hermes-offline-pure.tar.gz /opt/
```

---

## 阶段三：离线服务器 — 全部安装

**目标**：在完全无外网的服务器上，执行 Hermes 的完整安装。

**前置要求**：
- 已接收 `hermes-offline-pure.tar.gz`
- 服务器用户具有 sudo 权限
- 磁盘空间：3GB 可用

### Step 1 — 解压收集包

```bash
cd /opt

# 解压
sudo tar -xzvf hermes-offline-pure.tar.gz

# 确认目录结构
ls -la /opt/hermes-offline/
# 应输出: binaries/ python-packages/ src/ system-packages/ requirements-all.txt
```

### Step 2 — 安装系统依赖

#### Ubuntu / Debian

```bash
cd /opt/hermes-offline/system-packages

# 安装所有 .deb 包
sudo dpkg -i *.deb

# 如果提示依赖缺失，自动修复
sudo apt-get install -f -y

# 验证
git --version
python3.11 --version
node --version 2>/dev/null || echo "Node 尚未安装"
```

#### CentOS / Rocky / RHEL

```bash
cd /opt/hermes-offline/system-packages

# 安装所有 .rpm 包
sudo rpm -ivh *.rpm

# 如果提示依赖缺失
sudo dnf install -y *.rpm

# 验证
git --version
python3.11 --version
```

### Step 3 — 安装 uv 包管理器

```bash
cd /opt/hermes-offline/binaries

# 解压到 /usr/local/bin
sudo tar -xzf uv.tar.gz -C /usr/local/bin/

# 验证
/usr/local/bin/uv --version
# 应输出: uv x.x.x
```

### Step 4 — 安装 Node.js v22

```bash
cd /opt/hermes-offline/binaries

# 创建 Node.js 目录
sudo mkdir -p /opt/node

# 解压
sudo tar -xJf node.tar.xz -C /opt/node --strip-components=1

# 创建命令链接（放入 PATH）
sudo ln -sf /opt/node/bin/node /usr/local/bin/node
sudo ln -sf /opt/node/bin/npm /usr/local/bin/npm
sudo ln -sf /opt/node/bin/npx /usr/local/bin/npx

# 验证
node --version
# 应输出: v22.14.0

npm --version
# 应输出: 10.x.x
```

### Step 5 — 解压 Hermes 源码

```bash
cd /opt/hermes-offline/src

# 解压到 /opt/
sudo unzip -q hermes.zip -d /opt/

# 重命名为标准路径
sudo mv /opt/hermes-agent-main /opt/hermes-agent

# 验证
cd /opt/hermes-agent
ls -la
# 应看到: pyproject.toml  uv.lock  scripts/  gateway/ ...
```

### Step 6 — 创建虚拟环境并从本地 whl 安装依赖（核心步骤）

```bash
cd /opt/hermes-agent

# 1. 创建虚拟环境
sudo /usr/local/bin/uv venv /opt/hermes-agent/venv --python 3.11

# 2. 设置环境变量
export VIRTUAL_ENV="/opt/hermes-agent/venv"

# 3. ████████████████████████████████████████████████████████████
#    关键：从本地 whl 文件安装（--no-index 表示不连外网）
# ████████████████████████████████████████████████████████████

sudo /usr/local/bin/uv pip install \
  --no-index \
  --find-links /opt/hermes-offline/python-packages/ \
  -e ".[all]"

# 安装过程会看到类似输出：
# Resolved 218 packages in 0.01s
# Downloaded 0 packages   ← 0 是因为从本地读取
# Installed 218 packages  ← 全部来自本地

# 4. 验证包数量
ls /opt/hermes-agent/venv/lib/python3.11/site-packages/ | wc -l
# 应输出: 200+
```

**⚠️ 常见问题**：如果某些包安装失败，通常是系统架构不匹配（有网机器是 x86_64，离线服务器是 aarch64）。解决方法是**在有网机器上与离线服务器同架构**，或下载 `--platform` 指定目标架构的 whl。

### Step 7 — 创建 hermes 命令

```bash
# 创建系统级命令
sudo tee /usr/local/bin/hermes << 'EOF'
#!/bin/bash
export PYTHONPATH=""
export PYTHONHOME=""
export UV_NO_CONFIG=1
exec "/opt/hermes-agent/venv/bin/python" -m hermes_cli.main "$@"
EOF

sudo chmod +x /usr/local/bin/hermes

# 验证
hermes --version
# 应输出: Hermes Agent x.x.x
```

### Step 8 — 创建配置目录和 .env

```bash
# 创建数据目录
mkdir -p ~/.hermes/{cron,sessions,logs,pairing,hooks,image_cache,audio_cache,memories,skills}

# 创建 .env（只用本地 Ollama，禁用所有外网功能）
cat > ~/.hermes/.env << 'EOF'
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
EOF

chmod 600 ~/.hermes/.env

# 创建空配置文件
touch ~/.hermes/config.yaml

# 创建 SOUL.md
cat > ~/.hermes/SOUL.md << 'EOF'
# Hermes Agent
EOF
```

### Step 9 — 启动 Hermes

```bash
# 方式 A: 交互式 CLI（前台，适合调试）
hermes

# 方式 B: API Server 模式（后台常驻）
hermes gateway

# 方式 C: 用 tmux 保持会话
tmux new-session -d -s hermes "hermes gateway"
tmux attach -t hermes
```

### Step 10 — 验证 API

```bash
# 1. 健康检查
curl -s http://localhost:8642/health 2>/dev/null | python3 -m json.tool

# 2. 列出模型
curl -s http://localhost:8642/v1/models \
  -H "Authorization: Bearer your-offline-production-key" 2>/dev/null | python3 -m json.tool

# 3. 测试对话（需要 Ollama 已运行）
curl -s http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer your-offline-production-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1:70b",
    "messages": [{"role": "user", "content": "Hello"}]
  }' 2>/dev/null | python3 -m json.tool
```

---

## 下载地址汇总

### 必下资源

| 资源 | 下载地址 | 大小 | 说明 |
|------|---------|------|------|
| **uv (x86_64)** | https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-unknown-linux-gnu.tar.gz | ~15MB | 包管理器 |
| **uv (aarch64)** | https://github.com/astral-sh/uv/releases/latest/download/uv-aarch64-unknown-linux-gnu.tar.gz | ~15MB | ARM64 用 |
| **Node.js v22 (x86_64)** | https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-x64.tar.xz | ~30MB | 浏览器工具 |
| **Node.js v22 (aarch64)** | https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-arm64.tar.xz | ~30MB | ARM64 用 |
| **Hermes 源码** | https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.zip | ~50MB | 主分支源码 |

### 系统依赖（通过包管理器下载）

**Ubuntu/Debian**:
```bash
apt-get download git python3.11 python3.11-venv python3.11-dev build-essential libffi-dev libssl-dev ripgrep ffmpeg ca-certificates
```

**CentOS/Rocky/RHEL**:
```bash
yumdownloader git python3.11 python3.11-devel gcc gcc-c++ libffi-devel openssl-devel ripgrep ffmpeg ca-certificates
```

### Python 依赖（自动下载）

```bash
# 通过临时 venv 解析后，用 pip download 批量下载
pip download -r requirements-all.txt -d ./python-packages/ --only-binary :all:
```

---

## 一键脚本

### 脚本 A：文件收集（有网机器执行）

保存为 `collect-offline.sh`:

```bash
#!/bin/bash
set -e

echo "=============================================="
echo "Hermes Agent 离线文件收集脚本"
echo "此脚本只下载文件，不在本机安装 Hermes"
echo "=============================================="
echo ""

# 检查架构
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then
    UV_ARCH="x86_64"
    NODE_ARCH="x64"
elif [ "$ARCH" = "aarch64" ]; then
    UV_ARCH="aarch64"
    NODE_ARCH="arm64"
else
    echo "❌ 不支持的架构: $ARCH"
    exit 1
fi

mkdir -p /tmp/hermes-offline/{system-packages,python-packages,binaries,src}
cd /tmp/hermes-offline

# 1. 系统依赖
echo "[1/6] 下载系统依赖..."
cd system-packages
if command -v apt-get &>/dev/null; then
    apt-get download git python3.11 python3.11-venv python3.11-dev build-essential libffi-dev libssl-dev ripgrep ffmpeg ca-certificates 2>/dev/null || true
elif command -v dnf &>/dev/null; then
    sudo dnf install -y yum-utils
    yumdownloader git python3.11 python3.11-devel gcc gcc-c++ libffi-devel openssl-devel ripgrep ffmpeg ca-certificates 2>/dev/null || true
else
    echo "⚠ 无法自动下载系统依赖，请手动准备"
fi
cd ..

# 2. uv
echo "[2/6] 下载 uv ($UV_ARCH)..."
curl -fsSL -o binaries/uv.tar.gz \
  "https://github.com/astral-sh/uv/releases/latest/download/uv-${UV_ARCH}-unknown-linux-gnu.tar.gz"

# 3. Node.js
echo "[3/6] 下载 Node.js v22 ($NODE_ARCH)..."
curl -fsSL -o binaries/node.tar.xz \
  "https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-${NODE_ARCH}.tar.xz"

# 4. Hermes 源码
echo "[4/6] 下载 Hermes 源码..."
curl -fsSL -o src/hermes.zip \
  https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.zip

# 5. Python 依赖解析
echo "[5/6] 解析 Python 依赖..."
cd /tmp
unzip -q hermes-offline/src/hermes.zip
cd hermes-agent-main

# 临时安装 uv
sudo tar -xzf /tmp/hermes-offline/binaries/uv.tar.gz -C /usr/local/bin/

# 临时 venv
/usr/local/bin/uv venv /tmp/temp-venv --python 3.11
export VIRTUAL_ENV=/tmp/temp-venv

# 解析依赖
/usr/local/bin/uv pip install -e ".[all]"
/tmp/temp-venv/bin/pip freeze > /tmp/hermes-offline/requirements-all.txt

echo "[6/6] 下载 Python whl 包..."
/tmp/temp-venv/bin/pip download \
  -r /tmp/hermes-offline/requirements-all.txt \
  -d /tmp/hermes-offline/python-packages/ \
  --only-binary :all: \
  --platform manylinux2014_${ARCH} \
  --python-version 3.11

# 清理
rm -rf /tmp/temp-venv /tmp/hermes-agent-main
sudo rm -f /usr/local/bin/uv /usr/local/bin/uvx

# 打包
echo ""
echo "=============================================="
echo "收集完成！打包命令："
echo "  cd /tmp && tar -czvf hermes-offline-pure.tar.gz hermes-offline/"
echo ""
echo "文件统计："
du -sh /tmp/hermes-offline/* 2>/dev/null
echo "Python whl 数量: $(ls /tmp/hermes-offline/python-packages/*.whl 2>/dev/null | wc -l)"
echo "=============================================="
```

### 脚本 B：离线安装（离线服务器执行）

保存为 `install-offline.sh`:

```bash
#!/bin/bash
set -e

echo "=============================================="
echo "Hermes Agent 离线安装脚本"
echo "=============================================="
echo ""

if [ ! -d "/opt/hermes-offline" ]; then
    echo "❌ 未找到 /opt/hermes-offline/"
    echo "请先解压 hermes-offline-pure.tar.gz 到 /opt/"
    exit 1
fi

cd /opt/hermes-offline

# 1. 系统依赖
echo "[1/7] 安装系统依赖..."
cd system-packages
if ls *.deb &>/dev/null; then
    sudo dpkg -i *.deb
    sudo apt-get install -f -y
elif ls *.rpm &>/dev/null; then
    sudo rpm -ivh *.rpm || sudo dnf install -y *.rpm
else
    echo "⚠ 未找到系统依赖包，请手动安装"
fi
cd ..

# 2. uv
echo "[2/7] 安装 uv..."
sudo tar -xzf binaries/uv.tar.gz -C /usr/local/bin/

# 3. Node.js
echo "[3/7] 安装 Node.js..."
sudo mkdir -p /opt/node
sudo tar -xJf binaries/node.tar.xz -C /opt/node --strip-components=1
sudo ln -sf /opt/node/bin/{node,npm,npx} /usr/local/bin/

# 4. Hermes 源码
echo "[4/7] 解压 Hermes 源码..."
cd src
sudo unzip -q hermes.zip -d /opt/
sudo mv /opt/hermes-agent-main /opt/hermes-agent
cd ..

# 5. Python 依赖
echo "[5/7] 安装 Python 依赖..."
cd /opt/hermes-agent
sudo /usr/local/bin/uv venv /opt/hermes-agent/venv --python 3.11
export VIRTUAL_ENV="/opt/hermes-agent/venv"
sudo /usr/local/bin/uv pip install \
  --no-index \
  --find-links /opt/hermes-offline/python-packages/ \
  -e ".[all]"

# 6. 创建命令
echo "[6/7] 创建 hermes 命令..."
sudo tee /usr/local/bin/hermes << 'EOF'
#!/bin/bash
export PYTHONPATH=""
export PYTHONHOME=""
export UV_NO_CONFIG=1
exec "/opt/hermes-agent/venv/bin/python" -m hermes_cli.main "$@"
EOF
sudo chmod +x /usr/local/bin/hermes

# 7. 配置
echo "[7/7] 创建配置..."
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

echo ""
echo "=============================================="
echo "✅ 安装完成！"
echo ""
echo "验证命令："
echo "  hermes --version"
echo "  uv --version"
echo "  node --version"
echo ""
echo "启动命令："
echo "  hermes          (交互式 CLI)"
echo "  hermes gateway  (API Server)"
echo ""
echo "API 测试："
echo "  curl http://localhost:8642/health"
echo "=============================================="
```

---

## 故障排查

### Q1: dpkg 安装报错 "依赖关系问题"

```bash
# 修复命令
sudo apt-get install -f -y
```

### Q2: `uv pip install --no-index` 提示某些包找不到

**原因**: 有网机器和离线服务器架构不同（x86_64 vs aarch64），或某些包只有 sdist（源码包）没有 whl。

**解决**:
```bash
# 方案 1: 在有网机器上与离线服务器同架构执行收集
# 方案 2: 下载时指定目标架构
pip download ... --platform manylinux2014_aarch64

# 方案 3: 允许源码编译（离线服务器需要 gcc）
# 去掉 --only-binary :all: 参数
```

### Q3: `hermes: command not found`

```bash
# 检查
which hermes
ls -la /usr/local/bin/hermes

# 修复
export PATH="/usr/local/bin:$PATH"
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Q4: Python 包安装后 Hermes 启动报错

```bash
# 检查 venv 完整性
ls /opt/hermes-agent/venv/lib/python3.11/site-packages/ | grep hermes

# 检查 .env 配置
cat ~/.hermes/.env

# 运行诊断
hermes doctor
```

### Q5: 提示需要浏览器但 BROWSER_ENABLED=false 已设置

某些工具可能默认尝试加载 Playwright。在 `~/.hermes/config.yaml` 中显式禁用：

```yaml
browser:
  enabled: false
tools:
  web_search:
    enabled: false
```

---

## 附录

### 附录 A：系统依赖清单

**Ubuntu / Debian 必装**:
```
git                # 源码管理
python3.11         # Python 运行时
python3.11-venv    # 虚拟环境
python3.11-dev     # 开发头文件
build-essential    # gcc/make 等编译工具
libffi-dev         # cffi 依赖
libssl-dev         # cryptography 依赖
ripgrep            # 文件搜索工具（可选）
ffmpeg             # 音视频处理（可选）
ca-certificates    # SSL 证书
```

**CentOS / Rocky / RHEL 必装**:
```
git
python3.11
python3.11-devel
gcc
gcc-c++
libffi-devel
openssl-devel
ripgrep
ffmpeg
ca-certificates
```

### 附录 B：Ollama 离线安装（供本地 LLM）

Ollama 也需要离线安装，步骤类似：

**有网机器准备**:
```bash
# 下载 Ollama 安装脚本
curl -fsSL https://ollama.com/install.sh -o ollama-install.sh

# 下载 Ollama 二进制（脚本会下载，也可手动）
curl -fsSL -o ollama-linux-amd64 \
  https://github.com/ollama/ollama/releases/latest/download/ollama-linux-amd64

# 下载模型（需要运行 Ollama 服务后执行）
# sudo chmod +x ollama-linux-amd64
# sudo ./ollama-linux-amd64 serve &
# ./ollama-linux-amd64 pull llama3.1:70b
# 模型存储在 /usr/share/ollama/.ollama/models/

# 打包 Ollama
tar -czvf ollama-offline.tar.gz \
  ollama-linux-amd64 \
  /usr/share/ollama/
```

**离线服务器安装**:
```bash
# 解压
tar -xzvf ollama-offline.tar.gz

# 安装
sudo chmod +x ollama-linux-amd64
sudo mv ollama-linux-amd64 /usr/local/bin/ollama
sudo useradd -r -s /bin/false -m -d /usr/share/ollama ollama 2>/dev/null || true
sudo chown -R ollama:ollama /usr/share/ollama/

# 启动
sudo -u ollama /usr/local/bin/ollama serve &

# 验证
curl http://localhost:11434/api/tags
```

### 附录 C：架构对照表

| 服务器架构 | uv 下载地址 | Node.js 下载地址 |
|-----------|------------|-----------------|
| x86_64 (AMD64) | `uv-x86_64-unknown-linux-gnu.tar.gz` | `node-v22.14.0-linux-x64.tar.xz` |
| aarch64 (ARM64) | `uv-aarch64-unknown-linux-gnu.tar.gz` | `node-v22.14.0-linux-arm64.tar.xz` |

**确认架构**:
```bash
uname -m
# x86_64  → 用 x64 版本
# aarch64 → 用 arm64 版本
```

### 附录 D：安装后文件结构

```
/opt/
├── hermes-agent/              # Hermes 源码 + venv
│   ├── venv/                  # Python 虚拟环境（218 个包）
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── scripts/
│   ├── gateway/
│   └── ...
├── hermes-offline/            # 安装介质（可保留或删除）
│   ├── binaries/
│   │   ├── uv.tar.gz
│   │   └── node.tar.xz
│   ├── python-packages/       # 218 个 .whl 文件
│   ├── src/
│   │   └── hermes.zip
│   ├── system-packages/       # .deb 或 .rpm 文件
│   └── requirements-all.txt
├── node/                      # Node.js v22
└── ...

~/.hermes/                     # 用户数据
├── .env                       # 环境变量（API Key 等）
├── config.yaml                # 主配置
├── SOUL.md
├── cron/                      # 定时任务
├── sessions/                  # 会话记录
├── logs/                      # 日志
├── memories/                  # 记忆
└── skills/                    # 技能
```

---

*本手册基于 Hermes Agent 官方 install.sh、pyproject.toml 和 uv.lock 编写。*  
*所有版本号以官方最新发布为准，安装前请核实 URL 有效性。*
