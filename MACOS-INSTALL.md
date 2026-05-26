# Hermes Agent macOS 部署手册（详细版）

**版本**: v1.0  
**适用**: macOS 13+ (Ventura/Sonoma/Sequoia)，Intel & Apple Silicon  
**场景**: macOS 部署 + 外网可用 + 云端 API Key + API Server + Web Dashboard  
**日期**: 2026-05-21

---

## 目录

1. [系统要求](#系统要求)
2. [安装前准备](#安装前准备)
3. [方案 A：一键安装（推荐）](#方案-a一键安装推荐)
4. [方案 B：手动安装](#方案-b手动安装)
5. [配置云端模型（API Key）](#配置云端模型api-key)
6. [开启 API Server](#开启-api-server)
7. [开启 Web Dashboard](#开启-web-dashboard)
8. [验证安装](#验证安装)
9. [启动与使用](#启动与使用)
10. [升级](#升级)
11. [故障排查](#故障排查)
12. [附录](#附录)

---

## 系统要求

| 项目 | 最低要求 | 推荐 |
|------|---------|------|
| **macOS 版本** | 13 Ventura | 14 Sonoma / 15 Sequoia |
| **芯片** | Intel x86_64 或 Apple Silicon (M1/M2/M3/M4) | Apple Silicon |
| **内存** | 8 GB | 16 GB |
| **磁盘** | 5 GB 可用空间 | 10 GB |
| **网络** | 有外网（安装 + 调云端 API） | 有外网 |
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

## 配置云端模型（API Key）

你不需要本地模型。Hermes 直接调用 OpenAI / Anthropic / OpenRouter / Kimi / DeepSeek / 阿里通义等云端 API。

### Step 1 — 交互式配置（推荐）

```bash
hermes model
```

按提示：
1. 选 Provider（OpenAI / Anthropic / OpenRouter / Kimi / DeepSeek / DashScope…）
2. 选具体模型（如 `gpt-4o`、`claude-opus-4.6`、`deepseek-chat`）
3. 输入 API Key（自动加密存到 `~/.hermes/.env`）

### Step 2 — 验证 Key 已写入

```bash
cat ~/.hermes/.env
```

你会看到类似：

```bash
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxx
```

如果没有，手动写：

```bash
nano ~/.hermes/.env
```

主流 Provider 对照：

| Provider | `.env` 变量名 | 获取地址 |
|----------|-------------|----------|
| **OpenAI** | `OPENAI_API_KEY` | https://platform.openai.com/api-keys |
| **Anthropic** | `ANTHROPIC_API_KEY` | https://console.anthropic.com/settings/keys |
| **OpenRouter** | `OPENROUTER_API_KEY` | https://openrouter.ai/keys |
| **Kimi** | `KIMI_API_KEY` | https://platform.moonshot.cn/console/api-keys |
| **DeepSeek** | `DEEPSEEK_API_KEY` | https://platform.deepseek.com/api_keys |
| **阿里通义** | `DASHSCOPE_API_KEY` | https://dashscope.console.aliyun.com/apiKey |

### Step 3 — 确认模型配置

```bash
hermes doctor        # 检查连接是否通
hermes status        # 看当前用的模型
```

---

## 开启 API Server

把 Hermes 暴露为 OpenAI 兼容 API，供前端或他人调用。

### Step 1 — 编辑 `.env`

```bash
nano ~/.hermes/.env
```

在 Provider Key 下面加：

```bash
# ===== Provider Key（上面已有）=====
OPENAI_API_KEY=sk-xxx

# ===== API Server =====
API_SERVER_ENABLED=true
API_SERVER_KEY=sk-your-server-secret-key
```

### Step 2 — 启动 Gateway

```bash
hermes gateway
```

看到：

```
[API Server] API server listening on http://127.0.0.1:8642
```

### Step 3 — 测试

```bash
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer sk-your-server-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

有回复 → API 就绪。

### 暴露给局域网（可选）

```bash
nano ~/.hermes/.env
```

```bash
API_SERVER_ENABLED=true
API_SERVER_KEY=sk-your-server-secret-key
API_SERVER_HOST=0.0.0.0        # 监听所有网卡
API_SERVER_PORT=8642
```

重启 `hermes gateway`，局域网内访问 `http://你的内网IP:8642/v1`。

> ⚠️ 必须设强密钥。公网部署务必加 Nginx/Caddy + HTTPS，不要裸奔。

---

## 开启 Web Dashboard

浏览器里监控和配置 Hermes，不用碰终端。

### Step 1 — 装依赖（Dashboard 额外组件）

```bash
cd ~/.hermes/hermes-agent
export VIRTUAL_ENV="$(pwd)/venv"
uv pip install -e ".[web,pty]"
```

### Step 2 — 启动

```bash
hermes dashboard
```

自动打开浏览器：`http://127.0.0.1:9119`

### Step 3 — 常用功能

| 页面 | 功能 |
|------|------|
| **Status** | 看模型、工具、Gateway 健康度 |
| **Sessions** | 浏览历史对话、搜索、继续聊天 |
| **Configuration** | 改 API Key、切模型、调温度（不用编辑文件） |
| **Cron** | 管理定时任务 |
| **Memory** | 查看 Hermes 记住的内容 |
| **Skills** | 搜索、安装技能 |
| **Logs** | 实时日志 |
| **Chat** | 浏览器内直接和 Hermes 对话 |

### 带聊天功能的 Dashboard

```bash
hermes dashboard --tui
```

左侧菜单出现 **Chat**，流式输出、Markdown 渲染、工具调用展开。

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

### 验证 API Server

```bash
# 健康检查
curl -s http://localhost:8642/health

# 列出模型
curl -s http://localhost:8642/v1/models \
  -H "Authorization: Bearer sk-your-server-secret-key"

# 测试对话
curl -s http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer sk-your-server-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### 验证 Web Dashboard

浏览器访问 `http://127.0.0.1:9119`，能看到状态页面即正常。

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

### Q5: 前端连不上 API Server

```bash
# 确认 Hermes 监听了正确地址
cat ~/.hermes/.env | grep API_SERVER_HOST

# 确认防火墙没拦
sudo lsof -i :8642
```

### Q6: Dashboard 空白或报错

```bash
# 确认 web extra 已装
cd ~/.hermes/hermes-agent
uv pip list | grep -E "fastapi|uvicorn"

# 没装就装
uv pip install -e ".[web,pty]"
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

### 附录 B：环境变量参考（云端场景）

| 变量 | 说明 | 示例 |
|------|------|------|
| `OPENAI_API_KEY` | OpenAI API Key | `sk-xxx` |
| `ANTHROPIC_API_KEY` | Anthropic Key | `sk-ant-xxx` |
| `OPENROUTER_API_KEY` | OpenRouter Key | `sk-or-xxx` |
| `KIMI_API_KEY` | Kimi Key | `xxx` |
| `DEEPSEEK_API_KEY` | DeepSeek Key | `xxx` |
| `DASHSCOPE_API_KEY` | 阿里通义 Key | `xxx` |
| `API_SERVER_ENABLED` | 启用 API Server | `true` |
| `API_SERVER_PORT` | API 端口 | `8642` |
| `API_SERVER_HOST` | 绑定地址 | `127.0.0.1` 或 `0.0.0.0` |
| `API_SERVER_KEY` | API 鉴权密钥 | `sk-yours` |
| `BROWSER_ENABLED` | 浏览器工具 | `true` |
| `HERMES_HOME` | 数据目录 | `~/.hermes` |

### 附录 C：Apple Silicon vs Intel Mac

| 项目 | Apple Silicon (M1/M2/M3/M4) | Intel Mac |
|------|------------------------------|-----------|
| **Node.js 下载** | `node-v22.14.0-darwin-arm64.tar.xz` | `node-v22.14.0-darwin-x64.tar.xz` |
| **内存** | 16GB 推荐 | 16GB 推荐 |
| **安装路径** | `~/.hermes/hermes-agent` | 相同 |

### 附录 D：相关链接

- **Hermes 官方文档**: https://hermes-agent.nousresearch.com/docs
- **Hermes GitHub**: https://github.com/NousResearch/hermes-agent
- **uv 文档**: https://docs.astral.sh/uv
- **本仓库**: https://github.com/superGYC/Hermes

---

*本手册基于 Hermes Agent v0.14.0 官方 install.sh 和文档编写。*  
*场景：macOS + 外网 + 云端 API + API Server + Web Dashboard。*
