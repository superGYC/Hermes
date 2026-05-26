# Hermes Agent Web Dashboard 使用手册

**版本**: v1.0  
**适用**: macOS / Linux（已部署 Hermes Agent）  
**场景**: 浏览器中监控、配置和管理 Hermes Agent  
**日期**: 2026-05-26

---

## 目录

1. [Dashboard 是什么](#dashboard-是什么)
2. [安装依赖](#安装依赖)
3. [启动 Dashboard](#启动-dashboard)
4. [Dashboard 各页面详解](#dashboard-各页面详解)
5. [浏览器内聊天（Chat Tab）](#浏览器内聊天chat-tab)
6. [安全注意事项](#安全注意事项)
7. [远程访问（SSH 隧道）](#远程访问ssh-隧道)
8. [常见问题](#常见问题)

---

## Dashboard 是什么

Hermes Web Dashboard 是 Hermes Agent 的**浏览器控制面板**。启动后访问 `http://127.0.0.1:9119`，你可以：

- 📊 **监控状态** — 模型、Provider、工具、Gateway 健康度
- 💬 **管理会话** — 查看历史对话、切换会话
- 🔧 **编辑配置** — API Key、模型参数、工具开关（不用碰 YAML）
- 🕐 **管理定时任务** — Cron jobs 的增删改查
- 🧠 **查看记忆** — Hermes 记住的东西
- 🛠️ **管理技能** — 安装、卸载、查看 Skills
- 📜 **查看日志** — 实时日志流
- 💻 **浏览器内聊天** — 不用终端，直接网页里和 Hermes 对话

> ⚡ **核心优势**：不用记 CLI 命令，不用编辑配置文件，点点鼠标就能管理。

---

## 安装依赖

Dashboard **不在**默认安装包里，需要额外装 `web` 和 `pty` 两个 extra：

### 方案 A：如果你用的一键安装（install.sh）

```bash
# 进入 Hermes 源码目录
cd ~/.hermes/hermes-agent

# 安装 web 和 pty 依赖
export VIRTUAL_ENV="$(pwd)/venv"
uv pip install -e ".[web,pty]"
```

> 💡 如果你之前装的是 `.[all]`，其实已经包含了，直接跳到启动。

### 方案 B：如果你手动安装的

```bash
cd ~/hermes-agent
export VIRTUAL_ENV="$(pwd)/venv"
uv pip install -e ".[web,pty]"
```

### 方案 C：用 pip（如果你通过 pip 安装的 hermes-agent）

```bash
pip install "hermes-agent[web,pty]"
```

### 验证安装

```bash
hermes dashboard --help
# 应该显示 port、host、no-open 等选项
```

---

## 启动 Dashboard

### 基本启动

```bash
hermes dashboard
```

执行后：
1. 启动本地 Web 服务器（默认 `127.0.0.1:9119`）
2. **自动打开浏览器**访问 `http://127.0.0.1:9119`
3. 数据全部在本地，**不出本机**

### 常用启动选项

| 命令 | 作用 |
|------|------|
| `hermes dashboard` | 默认启动，自动开浏览器 |
| `hermes dashboard --port 8080` | 自定义端口 |
| `hermes dashboard --no-open` | 不自动打开浏览器 |
| `hermes dashboard --tui` | 启用浏览器内 Chat 标签页 |
| `hermes dashboard --host 0.0.0.0` | ⚠️ **危险**：暴露到局域网 |
| `hermes dashboard --insecure` | ⚠️ **危险**：允许非 localhost 访问 |

### 停止 Dashboard

在终端按 `Ctrl+C`，或者直接关闭终端窗口。

---

## Dashboard 各页面详解

### 🏠 主页 / Status

打开 `http://127.0.0.1:9119` 默认进入。

显示内容：
- **当前模型** — 正在用的 Provider 和模型名称
- **Gateway 状态** — Telegram、Discord、Slack 等是否连接正常
- **工具列表** — 已启用的 Tools（绿色 = 可用，灰色 = 关闭）
- **内存使用** — Hermes 当前的上下文占用
- **最近会话** — 最近的几个对话记录

操作：
- 点击工具可以**快速开关**
- 点击模型名称可以**切换模型**
- 绿色的 Gateway 状态 = 健康，红色 = 断开

### 💬 Sessions（会话管理）

左侧菜单 → **Sessions**

功能：
- 📋 **列表** — 所有历史会话，按时间倒序
- 🔍 **搜索** — 按关键词搜索历史对话
- 📌 **固定** — 把常用会话置顶
- 🗑️ **删除** — 清理不需要的会话
- 💾 **导出** — 导出对话为 Markdown / JSON

点击任意会话 → 右侧显示完整对话记录，可以**继续聊天**。

### 🔧 Configuration（配置管理）

左侧菜单 → **Configuration**

不用编辑 YAML，直接在网页改：

| 配置项 | 说明 |
|--------|------|
| **Model Provider** | 下拉选择：OpenAI、Anthropic、Ollama、OpenRouter... |
| **API Key** | 输入框，自动加密保存到 `~/.hermes/.env` |
| **Base URL** | 自定义 API 地址（如本地 Ollama `http://127.0.0.1:11434/v1`） |
| **Model Name** | 输入模型 ID，如 `llama3.1:8b`、`claude-opus-4.6` |
| **System Prompt** | 修改 Hermes 的系统提示词 |
| **Temperature** | 温度参数滑动条（0-2） |
| **Max Tokens** | 最大生成长度 |
| **Context Window** | 上下文窗口大小 |

修改后 → 点击 **Save** → 立即生效，不用重启。

### 🕐 Cron（定时任务）

左侧菜单 → **Cron**

查看和管理所有定时任务：

- 📋 **任务列表** — 名称、调度表达式、下次执行时间、状态
- ➕ **新建** — 填写名称、Cron 表达式、要执行的指令
- ▶️ **手动触发** — 点击 Run Now 立即执行一次
- 📝 **编辑** — 修改调度或指令
- 🗑️ **删除** — 移除任务
- 📜 **查看输出** — 上次执行的结果日志

示例任务：
- `0 8 * * *` — 每天早上 8 点执行
- `*/30 * * * *` — 每 30 分钟执行一次

### 🧠 Memory（记忆管理）

左侧菜单 → **Memory**

查看 Hermes 记住的信息：
- **Facts** — Hermes 记住的事实（如"用户喜欢 Python"）
- **Preferences** — 用户偏好设置
- **Project Context** — 项目相关上下文
- **SOUL.md** — Agent 的人格设定预览

可以手动添加、编辑或删除记忆条目。

### 🛠️ Skills（技能管理）

左侧菜单 → **Skills**

- 🔍 **搜索** — 按名称或描述搜索
- 📂 **分类浏览** — 按类别文件夹查看
- ⬇️ **安装** — 点击 Install 从仓库拉取
- ⬆️ **更新** — 检查并更新已安装 Skills
- 🗑️ **卸载** — 移除不需要的 Skill

### 📜 Logs（日志查看）

左侧菜单 → **Logs**

实时显示 Hermes 的运行日志：
- 🔄 **自动刷新** — 新日志自动追加
- 🔍 **过滤** — 按级别（INFO/WARN/ERROR）筛选
- 📥 **下载** — 导出完整日志文件
- ⏸️ **暂停** — 暂停刷新，方便查看某条日志

### 📊 Gateway（网关状态）

左侧菜单 → **Gateway**（如果有配置 Gateway）

监控已连接的聊天平台：
- Telegram Bot — 在线/离线、最后心跳时间
- Discord — 服务器连接状态
- Slack — WebSocket 连接状态
- WhatsApp — 二维码扫描状态
- 其他平台...

可以在这里**重启 Gateway** 或**重新配置**某个平台。

---

## 浏览器内聊天（Chat Tab）

Dashboard 最有意思的功能：**在浏览器里直接和 Hermes 聊天**，完全不用终端。

### 启用方式

启动时加 `--tui` 参数：

```bash
hermes dashboard --tui
```

或者在环境变量里设置：

```bash
export HERMES_DASHBOARD_TUI=1
hermes dashboard
```

### Chat Tab 功能

打开 Dashboard → 左侧菜单 → **Chat**

界面布局：
- 左边：会话列表（和 Sessions 页面同步）
- 中间：聊天区域（流式输出、Markdown 渲染、代码高亮）
- 右边：工具调用详情（展开看 Hermes 调用了什么工具）
- 底部：输入框 + 附件按钮 + 模型选择器

操作：
- **发送消息** — 底部输入框，Enter 发送，Shift+Enter 换行
- **上传文件** — 附件按钮，支持图片、PDF、代码文件
- **切换模型** — 底部模型选择器，实时切换不用重启
- **查看工具调用** — 点击消息旁边的 🔧 图标，展开看 Hermes 调用了什么工具、参数是什么、结果是什么
- **复制代码** — 代码块右上角有复制按钮
- **中断生成** — 发送新消息或点击停止按钮

> 💡 **和 CLI 的区别**：网页版有 Markdown 渲染、代码高亮、文件预览，体验比终端好。但底层是同一个 Hermes，会话完全同步。

---

## 安全注意事项

### 默认是安全的

- Dashboard 默认绑定 `127.0.0.1`（localhost），**只有本机能访问**
- 数据不出本机，没有上传到云端

### ⚠️ 不要做的危险操作

| 危险操作 | 后果 |
|----------|------|
| `hermes dashboard --host 0.0.0.0` | 局域网内所有人都能访问你的 Dashboard |
| `hermes dashboard --insecure` | 暴露 API Key 和配置到网络 |
| 在公共 Wi-Fi 下开 Dashboard | 同网段的人可能扫描到你的端口 |

### 如果需要远程访问

**正确做法**：用 SSH 隧道（见下一节），**不要**直接暴露到公网。

---

## 远程访问（SSH 隧道）

场景：Hermes 跑在服务器/VPS 上，你想在本地电脑用浏览器打开 Dashboard。

### 建立 SSH 隧道

在**本地电脑**（Mac/Windows）的终端执行：

```bash
ssh -L 9119:localhost:9119 user@your-server-ip
```

参数说明：
- `-L 9119:localhost:9119` — 把本地 9119 端口映射到服务器的 9119 端口
- `user@your-server-ip` — 你的服务器 SSH 地址

### 访问 Dashboard

隧道建立后，在**本地浏览器**打开：

```
http://localhost:9119
```

看起来像是本地服务，实际流量通过 SSH 加密传输到服务器。

### 断开连接

关闭 SSH 会话即可断开隧道。

---

## 常见问题

### Q1: `hermes dashboard` 报错 "Missing web dependencies"

```bash
# 装 web extra
cd ~/.hermes/hermes-agent
export VIRTUAL_ENV="$(pwd)/venv"
uv pip install -e ".[web,pty]"
```

### Q2: Dashboard 打开后前端空白

可能是前端还没构建。确保 npm 已安装：

```bash
# 检查 npm
npm --version

# 如果没有，安装 Node.js
curl -fsSL https://nodejs.org/dist/v22.14.0/node-v22.14.0-darwin-arm64.tar.xz | tar -xJf - -C ~/.hermes/
mv ~/.hermes/node-v22.14.0-darwin-arm64 ~/.hermes/node

# 重新启动 Dashboard，会自动构建前端
hermes dashboard
```

### Q3: Chat Tab 无法使用

需要 `pty` extra：

```bash
uv pip install -e ".[pty]"
```

macOS 上 `pty` 依赖 `ptyprocess`，会自动安装。

### Q4: 端口 9119 被占用

```bash
# 换个端口
hermes dashboard --port 8080
```

### Q5: 如何开机自启 Dashboard

macOS 用 LaunchAgent：

```bash
# 创建 plist 文件
cat > ~/Library/LaunchAgents/com.hermes.dashboard.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.hermes.dashboard</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/.local/bin/hermes</string>
        <string>dashboard</string>
        <string>--no-open</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/.hermes/logs/dashboard.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/.hermes/logs/dashboard-error.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>
</dict>
</plist>
EOF

# 替换 YOUR_USERNAME 为你的用户名
sed -i '' "s/YOUR_USERNAME/$(whoami)/g" ~/Library/LaunchAgents/com.hermes.dashboard.plist

# 加载
launchctl load ~/Library/LaunchAgents/com.hermes.dashboard.plist

# 查看状态
launchctl list | grep hermes
```

### Q6: Dashboard 和 CLI 同时用会冲突吗？

不会。Dashboard 是只读/管理界面，CLI 是交互界面。两者共享同一个 Hermes 后端：
- CLI 里开始的会话，Dashboard 里能看到
- Dashboard 里修改的配置，CLI 立即生效
- 可以同时开着 Dashboard 和 CLI

### Q7: 手机能访问 Dashboard 吗？

**局域网内**可以（如果手机和电脑同 Wi-Fi）：

```bash
# 绑定到局域网 IP（找到你的内网 IP）
ifconfig | grep "inet "

# 假设内网 IP 是 192.168.1.100
hermes dashboard --host 192.168.1.100 --no-open
```

然后在手机浏览器访问 `http://192.168.1.100:9119`。

> ⚠️ 只在受信任的局域网内这样做，不要公开到互联网。

---

## 相关链接

- **Hermes 官方 Dashboard 文档**: https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard
- **本仓库**: https://github.com/superGYC/Hermes
- **macOS 部署手册**: [MACOS-INSTALL.md](./MACOS-INSTALL.md)

---

*本手册基于 Hermes Agent v0.14.0 官方文档编写。*  
*Dashboard 是可选组件，默认不安装，需手动添加 `[web,pty]` extra。*
