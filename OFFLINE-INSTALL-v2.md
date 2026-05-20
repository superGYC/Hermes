# Hermes Agent 完全离线安装手册（无脑复制版）

**版本**: v2.0 — 基于官方 install.sh 改写  
**适用**: 服务器完全无外网，离线安装  
**日期**: 2026-05-20

---

## 思路：在有网机器上完整跑一次，把结果搬到离线服务器

官方 install.sh 会**在线下载**这些资源：
- uv 包管理器（astral.sh）
- Python 3.11（uv 自动管理）
- Node.js v22（nodejs.org 下载 tar 包）
- 218 个 Python 包（uv sync 从 PyPI）
- Chromium 浏览器（Playwright）

**离线方案**：在有网络的机器上**预装一遍**，把完整结果打包带走。

---

## 阶段一：有网络机器 —— 预装（准备传输包）

### Step 1 — 前置准备

```bash
# 确认系统：Ubuntu 22.04 / Debian 12 / CentOS 9 / Rocky Linux 9
uname -s    # 输出 Linux
uname -m    # 输出 x86_64 或 aarch64
```

**系统要求**：
- OS: Linux x86_64 或 aarch64
- 内存: 4GB+（安装时需要）
- 磁盘: 5GB 可用空间
- 必须已安装: `git`, `curl`

### Step 2 — 下载 uv 包管理器

```bash
# 下载 uv 安装脚本（约 20KB）
curl -fsSL https://astral.sh/uv/install.sh -o uv-installer.sh

# 执行安装（装到 ~/.local/bin/uv）
sh uv-installer.sh

# 确认
~/.local/bin/uv --version
# 输出类似: uv 0.7.2
```

**下载地址**（如 curl 不可用，手动下载）：
- https://astral.sh/uv/install.sh （安装脚本）
- https://github.com/astral-sh/uv/releases/latest （uv 二进制直链）

### Step 3 — 下载 Python 3.11

```bash
# uv 可以自动下载管理 Python，不需要系统 apt install python3.11
~/.local/bin/uv python install 3.11

# 确认
~/.local/bin/uv python find 3.11
# 输出类似: /home/user/.local/share/uv/python/cpython-3.11.13-linux-x86_64-gnu/bin/python3
```

### Step 4 — 克隆 Hermes 源码

```bash
# 克隆（约 50MB）
git clone --branch main https://github.com/NousResearch/hermes-agent.git

cd hermes-agent

# 当前目录结构
ls -la
# 应看到: pyproject.toml  uv.lock  scripts/  skills/  gateway/  ...
```

### Step 5 — 创建虚拟环境并预装所有依赖（核心步骤）

```bash
# 创建虚拟环境
~/.local/bin/uv venv venv --python 3.11

# 导出虚拟环境路径
export VIRTUAL_ENV="$(pwd)/venv"

# ██████████████████████████████████████████████████████████████
# 核心：用 uv.lock 做 hash-verified 安装，这会下载全部 218 个包
# 第一次运行需要 1-5 分钟，取决于网速
# ██████████████████████████████████████████████████████████████

UV_PROJECT_ENVIRONMENT="$(pwd)/venv" ~/.local/bin/uv sync --extra all --locked

# 成功后会看到:
# ✓ Main package installed (hash-verified via uv.lock)
```

### Step 6 — 下载 Node.js v22（用于浏览器工具）

```bash
# 进入 Hermes 目录
cd ~/hermes-agent

# 下载 Node.js v22 LTS（约 30MB）
# x86_64:
curl -fsSL https://nodejs.org/dist/latest-v22.x/node-v22.14.0-linux-x64.tar.xz -o node-v22.tar.xz

# aarch64 (ARM64):
# curl -fsSL https://nodejs.org/dist/latest-v22.x/node-v22.14.0-linux-arm64.tar.xz -o node-v22.tar.xz

# 解压到 ~/.hermes/node/
mkdir -p ~/.hermes/node
tar xf node-v22.tar.xz -C ~/.hermes/node --strip-components=1

# 确认
~/.hermes/node/bin/node --version
# 输出: v22.14.0
```

**Node.js 下载地址**：
- https://nodejs.org/dist/latest-v22.x/ （选择对应架构的 tar.xz）
- x86_64: `node-v22.14.0-linux-x64.tar.xz`
- aarch64: `node-v22.14.0-linux-arm64.tar.xz`

### Step 7 — 安装 npm 依赖（浏览器自动化）

```bash
cd ~/hermes-agent

# 使用预装的 Node.js
export PATH="$HOME/.hermes/node/bin:$PATH"

# 安装 Hermes 的 npm 依赖
~/.hermes/node/bin/npm install --silent

# 安装 Chromium（可选，如果离线服务器不需要浏览器工具，跳过）
# npx playwright install chromium
```

### Step 8 — 复制配置模板

```bash
mkdir -p ~/.hermes/{cron,sessions,logs,pairing,hooks,image_cache,audio_cache,memories,skills}

# 复制模板
cp ~/hermes-agent/.env.example ~/.hermes/.env 2>/dev/null || touch ~/.hermes/.env
chmod 600 ~/.hermes/.env

cp ~/hermes-agent/cli-config.yaml.example ~/.hermes/config.yaml 2>/dev/null || true

# 创建 SOUL.md
cat > ~/.hermes/SOUL.md << 'EOF'
# Hermes Agent Persona
EOF
```

### Step 9 — 设置 PATH 和命令链接

```bash
# 创建 hermes 命令入口
mkdir -p ~/.local/bin

# 写入启动脚本
cat > ~/.local/bin/hermes << 'EOF'
#!/bin/bash
export PYTHONPATH=""
export PYTHONHOME=""
exec "/home/$(whoami)/hermes-agent/venv/bin/python" -m hermes_cli.main "$@"
EOF
chmod +x ~/.local/bin/hermes

# 同时链接 Node.js 命令
ln -sf ~/.hermes/node/bin/node ~/.local/bin/node
ln -sf ~/.hermes/node/bin/npm ~/.local/bin/npm
ln -sf ~/.hermes/node/bin/npx ~/.local/bin/npx

# 确认 hermes 可用
export PATH="$HOME/.local/bin:$PATH"
hermes --version
```

### Step 10 — 配置本地模型（完全离线）

```bash
# 编辑 .env，只用本地 Ollama
cat > ~/.hermes/.env << 'EOF'
# ===== 本地模型配置 =====
OPENAI_BASE_URL=http://127.0.0.1:11434/v1
OPENAI_API_KEY=ollama
OPENAI_MODEL=llama3.1:70b

# ===== API Server（供外部 Web 调用）=====
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
```

### Step 11 — 打包整个安装成果

```bash
cd ~

# ██████████████████████████████████████████████████████████████
# 打包清单：
# 1. hermes-agent/          — 源码 + venv（含全部 218 个包）
# 2. .hermes/               — 配置 + Node.js + 数据目录
# 3. .local/bin/            — 命令入口（hermes, node, npm, npx）
# 4. .local/bin/uv          — uv 包管理器
# 5. .local/share/uv/       — uv 缓存 + Python 3.11 二进制
# ██████████████████████████████████████████████████████████████

tar -czvf hermes-offline-complete.tar.gz \
  hermes-agent/ \
  .hermes/ \
  .local/bin/hermes \
  .local/bin/node \
  .local/bin/npm \
  .local/bin/npx \
  .local/bin/uv \
  .local/share/uv/

# 查看大小
ls -lh hermes-offline-complete.tar.gz
# 约 1.5-3GB（含 venv 所有包）
```

---

## 阶段二：传输到离线服务器

### 方法 A — USB / 移动硬盘

```bash
# 有网机器
cp ~/hermes-offline-complete.tar.gz /mnt/usb/

# 离线服务器
cp /mnt/usb/hermes-offline-complete.tar.gz /opt/
```

### 方法 B — 内网文件服务器

```bash
# 有网机器
scp ~/hermes-offline-complete.tar.gz fileserver:/shared/packages/

# 离线服务器
cp /shared/packages/hermes-offline-complete.tar.gz /opt/
```

---

## 阶段三：离线服务器 —— 恢复安装

### Step 1 — 解压

```bash
cd /opt
tar -xzvf hermes-offline-complete.tar.gz

# 确认结构
ls -la /opt/hermes-agent/
ls -la /opt/.hermes/
ls -la /opt/.local/bin/
```

### Step 2 — 调整路径（关键！有网机器的用户名 ≠ 离线服务器的用户名）

```bash
# 查看当前用户名
whoami
# 假设离线服务器用户是: hermes-admin

# 把打包时的 /home/xxx 路径改为离线服务器的实际路径
mkdir -p /home/$(whoami)

# 移动到家目录
cd /opt
mv hermes-agent /home/$(whoami)/
mv .hermes /home/$(whoami)/
mv .local /home/$(whoami)/

# 确认
ls -la /home/$(whoami)/hermes-agent/
ls -la /home/$(whoami)/.hermes/
ls -la /home/$(whoami)/.local/bin/
```

### Step 3 — 修复 hermes 启动脚本中的硬编码路径

```bash
# 编辑 hermes 命令入口
nano /home/$(whoami)/.local/bin/hermes

# 改成以下内容（将 hermes-admin 替换为实际用户名）
```

```bash
#!/bin/bash
export PYTHONPATH=""
export PYTHONHOME=""
export UV_NO_CONFIG=1
exec "/home/$(whoami)/hermes-agent/venv/bin/python" -m hermes_cli.main "$@"
```

### Step 4 — 添加到 PATH

```bash
# 确认 .local/bin 在 PATH 中
export PATH="$HOME/.local/bin:$PATH"

# 永久生效（写入 .bashrc）
if ! grep -q 'export PATH="\$HOME/.local/bin:\$PATH"' ~/.bashrc; then
  echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
fi

source ~/.bashrc
```

### Step 5 — 验证所有组件

```bash
# 1. uv
uv --version

# 2. Python
$HOME/.local/share/uv/python/cpython-3.11.13-linux-x86_64-gnu/bin/python3 --version

# 3. Node.js
node --version

# 4. Hermes 命令
hermes --version

# 5. 虚拟环境中的依赖
ls -la ~/hermes-agent/venv/lib/python3.11/site-packages/ | head -30
```

### Step 6 — 配置本地 LLM（Ollama）

**在离线服务器上，Ollama 也要离线安装。** 参考下面的 Ollama 离线方案：

```bash
# 在有网机器上提前准备 Ollama（另见 Ollama 离线安装章节）
# 假设 Ollama 已经在离线服务器上运行于 127.0.0.1:11434

# 编辑 Hermes 配置
nano ~/.hermes/.env
```

```
OPENAI_BASE_URL=http://127.0.0.1:11434/v1
OPENAI_API_KEY=ollama
OPENAI_MODEL=llama3.1:70b
API_SERVER_ENABLED=true
API_SERVER_KEY=your-offline-production-key
```

### Step 7 — 启动 Hermes

```bash
# 方式 A — 前台运行（调试用）
hermes

# 方式 B — API Server 模式（后台）
hermes gateway

# 方式 C — 只启动 API Server
# Hermes v0.14.0 的 API Server 在 gateway 子命令里
```

### Step 8 — 验证 API

```bash
# 健康检查
curl -s http://localhost:8642/health | python3 -m json.tool

# 列出模型
curl -s http://localhost:8642/v1/models \
  -H "Authorization: Bearer your-offline-production-key" | python3 -m json.tool

# 测试对话
curl -s http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer your-offline-production-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1:70b",
    "messages": [{"role": "user", "content": "你好"}]
  }' | python3 -m json.tool
```

---

## 附录 A：Ollama 离线安装（供本地 LLM）

### 有网机器准备

```bash
# 1. 下载 Ollama 安装脚本
curl -fsSL https://ollama.com/install.sh -o ollama-install.sh

# 2. 安装（这会下载 ollama 二进制）
sh ollama-install.sh

# 3. 下载模型
ollama pull llama3.1:70b
# 模型存储在 /usr/share/ollama/.ollama/models/

# 4. 打包
sudo tar -czvf ollama-offline.tar.gz \
  /usr/local/bin/ollama \
  /usr/share/ollama/
```

### 离线服务器安装

```bash
# 1. 解压到对应位置
sudo tar -xzvf ollama-offline.tar.gz -C /

# 2. 创建 ollama 用户（安装脚本通常会创建）
sudo useradd -r -s /bin/false -m -d /usr/share/ollama ollama 2>/dev/null || true

# 3. 设置权限
sudo chown -R ollama:ollama /usr/share/ollama/
sudo chmod +x /usr/local/bin/ollama

# 4. 启动 Ollama 服务
sudo -u ollama /usr/local/bin/ollama serve &

# 5. 验证
curl http://localhost:11434/api/tags
```

---

## 附录 B：系统依赖清单（提前安装）

在离线服务器上，安装前需要这些系统包：

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install -y \
  git \
  curl \
  build-essential \
  python3-dev \
  libffi-dev \
  libssl-dev \
  ripgrep \
  ffmpeg \
  ca-certificates
```

### CentOS / Rocky / RHEL

```bash
sudo dnf install -y \
  git \
  curl \
  gcc \
  gcc-c++ \
  python3-devel \
  libffi-devel \
  openssl-devel \
  ripgrep \
  ffmpeg \
  ca-certificates
```

---

## 附录 C：如果不想要浏览器工具（进一步精简）

如果离线服务器**不需要**网页爬取、浏览器自动化：

### 有网机器 — 跳过浏览器相关步骤

```bash
# 在 Step 5 之后，直接跳到 Step 8
# 不装 Node.js
# 不运行 npm install
# 不装 Playwright Chromium

# 打包时也不包含 .hermes/node/
```

### 修改安装脚本（可选）

```bash
# 创建 .hermes/.env 时多加一行
BROWSER_ENABLED=false
```

---

## 附录 D：所有下载地址汇总

| 资源 | 下载地址 | 大小 |
|------|---------|------|
| **Hermes Agent 源码** | `https://github.com/NousResearch/hermes-agent.git` | ~50MB |
| **uv 安装脚本** | `https://astral.sh/uv/install.sh` | ~20KB |
| **uv 二进制（x86_64）** | `https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-unknown-linux-gnu.tar.gz` | ~15MB |
| **Python 3.11（uv 管理）** | `uv python install 3.11`（自动） | ~30MB |
| **Node.js v22（x86_64）** | `https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-x64.tar.xz` | ~30MB |
| **Node.js v22（aarch64）** | `https://nodejs.org/dist/v22.14.0/node-v22.14.0-linux-arm64.tar.xz` | ~30MB |
| **Ollama 安装脚本** | `https://ollama.com/install.sh` | ~5KB |
| **llama3.1:70b 模型** | `ollama pull llama3.1:70b`（自动） | ~40GB |
| **全部 Python 依赖** | `uv sync --extra all --locked`（自动下载 218 个包） | ~500MB-1GB |

---

## 附录 E：故障排查

### 问题 1: hermes 命令找不到

```bash
which hermes
# 如果无输出
echo $PATH
# 确认包含 $HOME/.local/bin

# 修复
export PATH="$HOME/.local/bin:$PATH"
source ~/.bashrc
```

### 问题 2: Python 包缺失

```bash
# 如果 uv sync 在离线时失败，检查 venv 是否完整
ls ~/hermes-agent/venv/lib/python3.11/site-packages/ | wc -l
# 应该看到 200+ 个包

# 如果不完整，在有网机器重新执行 Step 5
```

### 问题 3: 权限 denied

```bash
chmod +x ~/.local/bin/hermes
chmod +x ~/.hermes/node/bin/*
chmod 600 ~/.hermes/.env
```

### 问题 4: Node.js 版本不对

```bash
node --version
# 必须是 v22.x
# 如果不是，检查 ~/.hermes/node/bin/node 是否存在
```

---

## 附录 F：一键验证脚本（离线服务器用）

保存为 `verify-offline-install.sh`，复制到离线服务器执行：

```bash
#!/bin/bash
set -e

echo "=== Hermes Agent 离线安装验证 ==="
echo ""

echo "[1/6] 检查 uv..."
uv --version || { echo "❌ uv 缺失"; exit 1; }

echo "[2/6] 检查 Python 3.11..."
$HOME/.local/share/uv/python/cpython-3.11*/bin/python3 --version || { echo "❌ Python 3.11 缺失"; exit 1; }

echo "[3/6] 检查 Node.js..."
node --version || { echo "❌ Node.js 缺失"; exit 1; }

echo "[4/6] 检查 hermes 命令..."
hermes --version || { echo "❌ hermes 命令缺失"; exit 1; }

echo "[5/6] 检查虚拟环境包..."
PKG_COUNT=$(ls ~/hermes-agent/venv/lib/python3.11/site-packages/ 2>/dev/null | wc -l)
if [ "$PKG_COUNT" -lt 100 ]; then
  echo "❌ Python 包数量不足 ($PKG_COUNT)，期望 >100"
  exit 1
fi
echo "✓ Python 包数量: $PKG_COUNT"

echo "[6/6] 检查 .env 配置..."
[ -f ~/.hermes/.env ] || { echo "❌ ~/.hermes/.env 缺失"; exit 1; }
grep -q "OPENAI_BASE_URL" ~/.hermes/.env || { echo "❌ 本地模型配置缺失"; exit 1; }

echo ""
echo "✅ 全部验证通过！可以启动 Hermes"
echo "启动命令: hermes"
```

---

*本手册基于 Hermes Agent v0.14.0 官方 install.sh 和 pyproject.toml 编写。*
*所有版本号、下载地址以官方最新为准。安装前请核对应版本。*
