# Hermes Agent API Key 配置手册（云端模型）

**版本**: v1.0  
**适用**: macOS / Linux，有外网  
**场景**: 用 OpenAI / Anthropic / OpenRouter / Kimi 等云端 API，不用本地模型  
**日期**: 2026-05-26

---

## 两个配置文件

Hermes 把配置拆成两处：

| 文件 | 放什么 | 怎么改 |
|------|--------|--------|
| **`~/.hermes/.env`** | **Secrets** — API Key、密码 | 手动编辑或用 `hermes config set` |
| **`~/.hermes/config.yaml`** | **非 Secrets** — 模型名、开关、偏好 | 手动编辑 |

> ⚠️ `.env` 权限默认 `600`，别泄露。

---

## 快速配置（3 步）

### Step 1 — 选 Provider 和模型

```bash
hermes model
```

交互式选择：
1. 选 Provider（OpenAI / Anthropic / OpenRouter / Kimi / DeepSeek / ...）
2. 选具体模型（如 `gpt-4o`、`claude-opus-4.6`、`deepseek-chat`）
3. 输 API Key（会加密存到 `~/.hermes/.env`）

### Step 2 — 确认 `.env` 写对了

```bash
cat ~/.hermes/.env
```

根据你选的 Provider，会看到类似：

```bash
# ===== OpenAI =====
OPENAI_API_KEY=sk-你的-key

# ===== Anthropic =====
ANTHROPIC_API_KEY=sk-ant-你的-key

# ===== OpenRouter =====
OPENROUTER_API_KEY=sk-or-你的-key

# ===== Kimi / Moonshot =====
KIMI_API_KEY=你的-key

# ===== DeepSeek =====
DEEPSEEK_API_KEY=你的-key

# ===== 阿里通义 (DashScope) =====
DASHSCOPE_API_KEY=你的-key
```

如果没有，手动加：

```bash
nano ~/.hermes/.env
```

### Step 3 — 验证

```bash
hermes doctor        # 检查配置和连接
hermes status        # 看当前用的模型和 Provider
hermes               # 启动聊天，测试是否能回复
```

---

## 主流 Provider 配置对照

| Provider | 环境变量 | 获取 Key 地址 | 推荐模型 |
|----------|----------|-------------|----------|
| **OpenAI** | `OPENAI_API_KEY` | https://platform.openai.com/api-keys | `gpt-4o`, `gpt-4o-mini` |
| **Anthropic** | `ANTHROPIC_API_KEY` | https://console.anthropic.com/settings/keys | `claude-opus-4.6`, `claude-sonnet-4` |
| **OpenRouter** | `OPENROUTER_API_KEY` | https://openrouter.ai/keys | `openai/gpt-4o`, `anthropic/claude-3.5-sonnet` |
| **Kimi** | `KIMI_API_KEY` | https://platform.moonshot.cn/console/api-keys | `kimi-k2`, `kimi-k1.5` |
| **DeepSeek** | `DEEPSEEK_API_KEY` | https://platform.deepseek.com/api_keys | `deepseek-chat`, `deepseek-reasoner` |
| **阿里通义** | `DASHSCOPE_API_KEY` | https://dashscope.console.aliyun.com/apiKey | `qwen-max`, `qwen-coder-plus` |
| **MiniMax** | `MINIMAX_API_KEY` | MiniMax 开放平台 | `MiniMax-M2.7` |
| **Z.AI (智谱)** | `GLM_API_KEY` | https://open.bigmodel.cn/usercenter/apikeys | `glm-4-plus` |

---

## 直接改配置文件（不用交互）

### 只改 `.env`

```bash
nano ~/.hermes/.env
```

示例（OpenAI）：

```bash
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxx
```

示例（OpenRouter，多模型路由）：

```bash
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxx
```

### 改 `config.yaml` 指定默认模型

```bash
nano ~/.hermes/config.yaml
```

```yaml
# 默认模型（CLI 启动时自动用这个）
model: openai/gpt-4o

# 或者 Anthropic
# model: anthropic/claude-opus-4.6

# 温度
temperature: 0.7

# 最大 token
max_tokens: 4096
```

---

## 开启 API Server（给别人/前端调用）

如果你还要把 Hermes 暴露成 API：

```bash
nano ~/.hermes/.env
```

在 Provider Key 下面加：

```bash
# ===== Provider Key（上面已有）=====
OPENAI_API_KEY=sk-xxx

# ===== API Server 配置 =====
API_SERVER_ENABLED=true
API_SERVER_KEY=sk-your-server-secret-key
```

启动：

```bash
hermes gateway
```

调用：

```bash
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer sk-your-server-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

---

## 常用命令速查

| 命令 | 作用 |
|------|------|
| `hermes model` | 交互式选择 Provider 和模型 |
| `hermes config set OPENAI_API_KEY sk-xxx` | 直接设 Key（自动写入 `.env`） |
| `hermes config set model openai/gpt-4o` | 改默认模型 |
| `hermes doctor` | 诊断配置和连接 |
| `hermes status` | 看当前配置快照 |
| `hermes` | 启动 CLI 聊天 |
| `hermes --tui` | 启动 TUI 聊天 |

---

## 验证清单

```bash
# 1. 确认 Key 已存
hermes config get OPENAI_API_KEY
# 输出: sk-xxx（脱敏显示）

# 2. 确认模型已配
hermes config get model
# 输出: openai/gpt-4o

# 3. 诊断网络
hermes doctor
# 应该全绿 ✓

# 4. 发一条测试消息
hermes chat -q "Say hello and tell me what model you are"
```

---

*本手册聚焦云端 API 调用，不含本地模型内容。*
