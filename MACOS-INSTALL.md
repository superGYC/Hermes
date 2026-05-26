# Hermes Agent macOS 部署手册（详细版）

**版本**: v1.0  
**适用**: macOS 13+ (Ventura/Sonoma/Sequoia)，Intel & Apple Silicon  
**场景**: 本地部署 + 外网可用 + 本地 LLM (Ollama)  
**日期**: 2026-05-21

---

## 目录

1. [系统要求](#系统要求)
2. [安装前准备](#安装前准备)
3. [方案 A：一键安装（推荐）](#方案-a一键安装推荐)
4. [方案 B：手动安装](#方案-b手动安装)
5. [配置本地 LLM (Ollama)](#配置本地-llm-ollama)
6. [验证安装](#验证安装)
7. [启动与使用](#启动与使用)
8. [升级](#升级)
9. [故障排查](#故障排查)
10. [附录](#附录)

---

## 系统要求

| 项目 | 最低要求 | 推荐 |
|------|---------|------|
| **macOS 版本** | 13 Ventura | 14 Sonoma / 15 Sequoia |
| **芯片** | Intel x86_64 或 Apple Silicon (M1/M2/M3/M4) | Apple Silicon |
| **内存** | 8 GB | 16 GB+（本地模型需要更多） |
| **磁盘** | 5 GB 可用空间 | 20 GB+（含模型文件） |
| **网络** | 有外网（安装时需要） | 有外网 |
| **前置依赖** | git | git + Homebrew |

> ⚠️ **Apple Silicon 注意**：Ollama 在 Apple Silicon 上会自动使用 Metal GPU 加速，性能远好于 Intel Mac。

---

## 安装前准备

### Step 1 — 安装 Xcode Command Line Tools

macOS 编译某些 Python 包需要 C 编译器：

```bash
xcode-select --install
```

如果提示已安装，直接下一步。

### Step 2 — 安装 Homebrew（推荐但非必须）

Homebrew 让安装系统工具更方便，Hermes 的 install.sh 会自动用 brew 装 ripgrep 和 ffmpeg：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

装完后按提示把 brew 加入 PATH（通常是 `eval "$(/opt/homebrew/bin/brew shellenv)"`）。

### Step 3 — 确认 git 已安装

```bash
git --version
# 输出类似: git version 2.39.0
```

macOS 自带 git，如果没有：

```bash
# 方案 A: 通过 Xcode Command Line Tools（上面已装）
# 方案 B: 通过 Homebrew
brew install git
```

---

## 方案 A：一键安装（推荐）

官方 install.sh 支持 macOS，60 秒内完成：

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

脚本会自动完成：
1. ✅ 检测 macOS 系统
2. ✅ 安装 uv 包管理器
3. ✅ 安装 Python 3.11（通过 uv）
4. ✅ 安装 Node.js v22（通过官网二进制）
5. ✅ 克隆 Hermes 源码
6. ✅ 创建 venv 并安装全部 218 个依赖
7. ✅ 安装 ripgrep（通过 brew）
8. ✅ 安装 ffmpeg（通过 brew）
9. ✅ 安装 Chromium（通过 Playwright）
10. ✅ 创建 `~/.hermes/` 配置目录
11. ✅ 创建 `hermes` 命令链接
12. ✅ 启动交互式 setup 向导

### 安装后重新加载 shell

```bash
# 根据你的 shell 选择
source ~/.zshrc    # zsh（macOS 默认）
source ~/.bashrc   # bash
```

### 验证

```bash
hermes --version
# 输出: Hermes Agent x.x.x
```

---

## 方案 B：手动安装

如果你想看清每一步在做什么，或者一键脚本出了错：

### Step 1 — 安装 uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

装完后：

```bash
source ~/.zshrc
uv --version
```

### Step 2 — 克隆 Hermes 源码

```bash
cd ~
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
```

### Step 3 — 创建虚拟环境并安装依赖

```bash
cd ~/hermes-agent

# 创建 venv（Python 3.11）
uv venv venv --python 3.11

# 导出 VIRTUAL_ENV
export VIRTUAL_ENV="$(pwd)/venv"

# 安装 Hermes + 全部功能
uv pip install -e ".[all]"
```

### Step 4 — 创建 hermes 命令

```bash
mkdir -p ~/.local/bin
ln -sf "$(pwd)/venv/bin/hermes" ~/.local/bin/hermes
```

### Step 5 — 确认 PATH

```bash
# 检查 ~/.local/bin 是否在 PATH
echo $PATH | grep .local/bin

# 如果不在，加入 ~/.zshrc
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Step 6 — 初始化配置目录

```bash
mkdir -p ~/.hermes/{cron,sessions,logs,pairing,hooks,image_cache,audio_cache,memories,skills}

cp ~/hermes-agent/.env.example ~/.hermes/.env 2>/dev/null || touch ~/.hermes/.env
chmod 600 ~/.hermes/.env

cp ~/hermes-agent/cli-config.yaml.example ~/.hermes/config.yaml 2>/dev/null || touch ~/.hermes/config.yaml
```

### Step 7 — 安装 Node.js + 浏览器工具（可选）

```bash
# 安装 Node.js v22（官网二进制）
curl -fsSL https://nodejs.org/dist/v22.14.0/node-v22.14.0-darwin-arm64.tar.xz | tar -xJf - -C ~/.hermes/
mv ~/.hermes/node-v22.14.0-darwin-arm64 ~/.hermes/node

# 链接命令
mkdir -p ~/.local/bin
ln -sf ~/.hermes/node/bin/node ~/.local/bin/node
ln -sf ~/.hermes/node/bin/npm ~/.local/bin/npm

# 安装 Hermes 的 npm 依赖
cd ~/hermes-agent
npm install

# 安装 Chromium（macOS 上 Playwright 会下载 Chromium）
npx playwright install chromium
```

> 💡 **Intel Mac** 把 `darwin-arm64` 换成 `darwin-x64`：
> `https://nodejs.org/dist/v22.14.0/node-v22.14.0-darwin-x64.tar.xz`

---

## 配置本地 LLM (Ollama)

### Step 1 — 安装 Ollama

```bash
# 一键安装（官网脚本）
curl -fsSL https://ollama.com/install.sh | sh
```

或手动下载：

```bash
# Apple Silicon
sudo curl -L https://ollama.com/download/ollama-darwin -o /usr/local/bin/ollama
sudo chmod +x /usr/local/bin/ollama

# Intel Mac 用 ollama-darwin-amd64
```

### Step 2 — 下载本地模型

```bash
# 启动 Ollama 服务
ollama serve &

# 下载 Llama 3.1 70B（约 40GB，推荐 32GB+ 内存）
ollama pull llama3.1:70b

# 或者下载轻量版（8B，约 5GB，8GB 内存可跑）
ollama pull llama3.1:8b

# 或者下载 Qwen 2.5（中文表现好）
ollama pull qwen2.5:14b
```

> ⚠️ **内存不足怎么办**：用 8B 或 14B 模型，或加 `--quantize q4_0` 参数。Apple Silicon 的 Unified Memory 架构下，16GB 内存可以跑 14B 模型。

### Step 3 — 配置 Hermes 使用本地模型

```bash
# 编辑 Hermes 配置
nano ~/.hermes/.env
```

写入以下内容：

```bash
# ===== LLM 配置（本地 Ollama） =====
OPENAI_BASE_URL=http://127.0.0.1:11434/v1
OPENAI_API_KEY=ollama

# 模型选择（根据你下载的）
# OPENAI_MODEL=llama3.1:70b      # 70B，质量最高，需要 32GB+ 内存
OPENAI_MODEL=llama3.1:8b        # 8B，轻量，8GB 内存可跑
# OPENAI_MODEL=qwen2.5:14b       # 14B，中文好，16GB 内存可跑

# ===== API Server（可选，供外部调用） =====
API_SERVER_ENABLED=true
API_SERVER_KEY=your-local-api-key-here

# ===== 禁用不需要的云端功能 =====
WEB_SEARCH_ENABLED=false
TTS_ENABLED=false
STT_ENABLED=false
IMAGE_GENERATION_ENABLED=false
# BROWSER_ENABLED=true           # 如果装了 Chromium，可以保留
```

### Step 4 — 测试 Ollama

```bash
# 确认 Ollama 在跑
curl http://localhost:11434/api/tags

# 直接对话测试
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.1:8b",
  "messages": [{"role": "user", "content": "你好"}]
}'
```

---

## 验证安装

### 基础验证清单

```bash
# 1. hermes 命令
hermes --version

# 2. 诊断工具
hermes doctor

# 3. 查看配置
hermes status

# 4. 列出可用技能
hermes skills list

# 5. 查看模型配置
hermes model
```

### 运行第一次对话

```bash
# 经典 CLI
hermes

# 或现代 TUI
hermes --tui
```

测试指令：

```
Summarize this repo in 5 bullets and tell me what the main entrypoint is.
```

如果 Hermes 能回复且没有报错，安装成功。

### 验证 API Server（如果启用了）

```bash
# 健康检查
curl -s http://localhost:8642/health | python3 -m json.tool

# 列出模型
curl -s http://localhost:8642/v1/models \
  -H "Authorization: Bearer your-local-api-key-here" | python3 -m json.tool

# 测试对话
curl -s http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer your-local-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1:8b",
    "messages": [{"role": "user", "content": "Hello"}]
  }' | python3 -m json.tool
```

---

## 启动与使用

### 交互式 CLI

```bash
# 启动新对话
hermes

# 继续上次对话
hermes --continue
hermes -c
```

### 常用命令

| 命令 | 作用 |
|------|------|
| `hermes` | 启动聊天 |
| `hermes --tui` | 启动 TUI 界面（推荐） |
| `hermes model` | 切换模型/提供商 |
| `hermes setup` | 完整配置向导 |
| `hermes doctor` | 诊断问题 |
| `hermes status` | 查看当前配置 |
| `hermes tools` | 管理工具开关 |
| `hermes skills search xxx` | 搜索技能 |
| `hermes skills install xxx` | 安装技能 |
| `/help` | 聊天中查看所有命令 |
| `/tools` | 查看可用工具 |
| `/save` | 保存对话 |
| `/voice on` | 开启语音模式 |

### 作为后台服务运行（常驻）

```bash
# 前台运行（适合调试）
hermes gateway

# 用 tmux 保持会话
tmux new-session -d -s hermes "hermes gateway"
tmux attach -t hermes

# 安装为 macOS LaunchAgent（开机自启）
hermes gateway install
```

---

## 升级

### 方案 A：内置更新命令

```bash
hermes update
```

### 方案 B：手动拉取最新代码

```bash
cd ~/hermes-agent
git pull
export VIRTUAL_ENV="$(pwd)/venv"
uv pip install -e ".[all]"
```

### 方案 C：重新运行安装脚本

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

---

## 故障排查

### Q1: `hermes: command not found`

```bash
# 确认 hermes 命令位置
ls -la ~/.local/bin/hermes

# 确认 PATH
echo $PATH | grep .local/bin

# 修复：加入 PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Q2: 安装脚本提示 "uv install failed"

```bash
# 手动安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.zshrc

# 确认 uv 版本
uv --version

# 重新运行 Hermes 安装
```

### Q3: `uv pip install` 报错编译失败

macOS 上某些包需要编译 C 扩展：

```bash
# 确认 Xcode Command Line Tools 已装
xcode-select --install

# 如果已装但路径不对
sudo xcode-select --reset

# 或指定 Xcode 路径
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
```

### Q4: Playwright/Chromium 安装失败

```bash
# 手动安装 Chromium
cd ~/hermes-agent
npx playwright install chromium

# 如果还是失败，跳过浏览器工具
# 编辑 ~/.hermes/.env，添加:
BROWSER_ENABLED=false
```

### Q5: Ollama 模型下载慢

```bash
# 设置镜像（国内用户）
export OLLAMA_HOST=0.0.0.0
export OLLAMA_ORIGINS=*

# 或使用代理
export HTTPS_PROXY=http://your-proxy:port
ollama pull llama3.1:8b
```

### Q6: Apple Silicon 上 Ollama 提示 "model requires more system memory"

```bash
# 换更小的模型
ollama pull llama3.1:8b

# 或量化版本
ollama pull llama3.1:8b --quantize q4_0

# 或限制并发
# 在 ~/.ollama/config 中设置 num_thread
```

### Q7: `hermes doctor` 报 Python 版本不对

```bash
# 确认 venv 用的是 Python 3.11
~/hermes-agent/venv/bin/python --version

# 如果不是 3.11，重建 venv
cd ~/hermes-agent
rm -rf venv
uv venv venv --python 3.11
export VIRTUAL_ENV="$(pwd)/venv"
uv pip install -e ".[all]"
```

---

## 附录

### 附录 A：完整安装后文件结构

```
~/
├── .hermes/                    # 用户数据 & 配置
│   ├── .env                    # API Key、模型配置
│   ├── config.yaml             # 主配置
│   ├── SOUL.md                 # Agent 人格设定
│   ├── cron/                   # 定时任务
│   ├── sessions/               # 对话记录
│   ├── logs/                   # 日志
│   ├── memories/               # 记忆
│   ├── skills/                 # 技能
│   ├── node/                   # Node.js v22（脚本安装）
│   └── ...
├── hermes-agent/               # 源码（手动安装）
│   ├── venv/                   # Python 虚拟环境
│   ├── pyproject.toml
│   ├── uv.lock
│   └── ...
└── .local/bin/hermes           # 命令入口
```

### 附录 B：环境变量参考

| 变量 | 说明 | 示例 |
|------|------|------|
| `OPENAI_BASE_URL` | LLM API 地址 | `http://127.0.0.1:11434/v1` |
| `OPENAI_API_KEY` | API Key | `ollama` |
| `OPENAI_MODEL` | 模型名称 | `llama3.1:8b` |
| `API_SERVER_ENABLED` | 启用 API Server | `true` |
| `API_SERVER_KEY` | API Server 鉴权 | `sk-xxxx` |
| `WEB_SEARCH_ENABLED` | 网页搜索 | `false` |
| `TTS_ENABLED` | 语音合成 | `false` |
| `BROWSER_ENABLED` | 浏览器工具 | `true` |
| `HERMES_HOME` | 数据目录 | `~/.hermes` |

### 附录 C：Apple Silicon vs Intel Mac

| 项目 | Apple Silicon (M1/M2/M3/M4) | Intel Mac |
|------|------------------------------|-----------|
| **Node.js 下载** | `node-v22.14.0-darwin-arm64.tar.xz` | `node-v22.14.0-darwin-x64.tar.xz` |
| **Ollama GPU 加速** | Metal（原生支持，速度快） | 无（纯 CPU） |
| **推荐模型** | 14B/70B 均可跑 | 建议 8B |
| **内存要求** | 16GB 可跑 14B | 32GB 才能跑 14B |

### 附录 D：相关链接

- **Hermes 官方文档**: https://hermes-agent.nousresearch.com/docs
- **Hermes GitHub**: https://github.com/NousResearch/hermes-agent
- **Ollama 官网**: https://ollama.com
- **Ollama 模型库**: https://ollama.com/library
- **uv 文档**: https://docs.astral.sh/uv
- **本仓库**: https://github.com/superGYC/Hermes

---

*本手册基于 Hermes Agent v0.14.0 官方 install.sh 和文档编写。*  
*macOS 版本以 Sonoma/Sequoia 为主，Ventura 亦兼容。*
