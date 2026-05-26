# Hermes Agent API Server 部署手册

**版本**: v1.0  
**适用**: macOS / Linux（已部署 Hermes Agent）  
**场景**: 把 Hermes 暴露为 OpenAI 兼容 API，供前端/他人调用  
**日期**: 2026-05-26

---

## 目录

1. [API Server 是什么](#api-server-是什么)
2. [快速开启（3 步）](#快速开启3-步)
3. [核心端点一览](#核心端点一览)
4. [认证方式](#认证方式)
5. [暴露给局域网/公网](#暴露给局域网公网)
6. [前端接入示例](#前端接入示例)
7. [多用户隔离（Profile）](#多用户隔离profile)
8. [常见问题](#常见问题)

---

## API Server 是什么

Hermes 内置了一个 **OpenAI 兼容的 HTTP API**。启动后，任何支持 OpenAI 格式的前端（Open WebUI、LobeChat、ChatBox、NextChat 等）都可以连上来，把 Hermes 当后端用。

**关键特点**：
- 兼容 `/v1/chat/completions` 和 `/v1/responses`
- Hermes 的工具（终端、文件操作、搜索、记忆）仍然可用
- 流式输出（SSE）带工具进度提示
- 支持图片输入（URL 或 Base64）

---

## 快速开启（3 步）

### Step 1 — 编辑 `.env` 配置

```bash
nano ~/.hermes/.env
```

添加：

```bash
# 开启 API Server
API_SERVER_ENABLED=true

# 认证密钥（必须设置，调用时带在 Header 里）
API_SERVER_KEY=sk-your-secret-key-here

# 可选：CORS（浏览器前端直连时需要）
# API_SERVER_CORS_ORIGINS=http://localhost:3000,http://127.0.0.1:3000
```

### Step 2 — 启动 Gateway

```bash
hermes gateway
```

终端会显示：

```
[API Server] API server listening on http://127.0.0.1:8642
```

### Step 3 — 测试

```bash
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer sk-your-secret-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hermes-agent",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

有回复 → API 就绪。

---

## 核心端点一览

| 方法 | 端点 | 作用 |
|------|------|------|
| `POST` | `/v1/chat/completions` | 标准 OpenAI 聊天（无状态，每次带完整 messages） |
| `POST` | `/v1/responses` | OpenAI Responses API（支持 `previous_response_id` 保持上下文） |
| `GET` | `/v1/responses/{id}` | 获取之前存储的 Response |
| `DELETE` | `/v1/responses/{id}` | 删除存储的 Response |
| `GET` | `/v1/models` | 列出可用模型（前端需要这个做模型发现） |
| `GET` | `/v1/capabilities` | 返回 API 能力清单（供 UI 自动探测） |
| `GET` | `/health` | 健康检查（返回 `{"status": "ok"}`） |
| `POST` | `/v1/runs` | 创建 Agent Run（长会话，返回 run_id） |
| `GET` | `/v1/runs/{id}` | 查询 Run 状态 |
| `GET` | `/v1/runs/{id}/events` | SSE 订阅 Run 的实时事件 |
| `POST` | `/v1/runs/{id}/stop` | 中断正在执行的 Run |
| `GET` | `/api/jobs` | 列出定时任务 |
| `POST` | `/api/jobs` | 创建定时任务 |
| `GET` | `/api/jobs/{id}` | 查看任务详情 |
| `PATCH` | `/api/jobs/{id}` | 更新任务 |
| `DELETE` | `/api/jobs/{id}` | 删除任务 |
| `POST` | `/api/jobs/{id}/run` | 手动触发任务 |

---

## 认证方式

唯一认证：**Bearer Token**

```bash
Authorization: Bearer sk-your-secret-key-here
```

如果没有 `API_SERVER_KEY`，Hermes 会拒绝所有请求。

### curl 示例

```bash
curl http://localhost:8642/v1/models \
  -H "Authorization: Bearer sk-your-secret-key-here"
```

### Python 示例

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8642/v1",
    api_key="sk-your-secret-key-here"  # 这里的 key 就是 API_SERVER_KEY
)

response = client.chat.completions.create(
    model="hermes-agent",
    messages=[{"role": "user", "content": "List files in current directory"}]
)
print(response.choices[0].message.content)
```

---

## 暴露给局域网/公网

**默认绑定 `127.0.0.1`，只有本机能访问。** 如果要给局域网或公网调用：

### 方案 A：局域网（同 Wi-Fi）

```bash
# 编辑 .env
nano ~/.hermes/.env
```

```bash
API_SERVER_ENABLED=true
API_SERVER_KEY=sk-your-secret-key-here
API_SERVER_HOST=0.0.0.0       # 监听所有网卡
API_SERVER_PORT=8642
```

重启 Gateway：

```bash
hermes gateway
```

局域网内其他设备访问：

```
http://你的内网IP:8642/v1
# 例如 http://192.168.1.100:8642/v1
```

> ⚠️ **必须设置强密钥**，局域网内所有人都能访问。

### 方案 B：公网（云服务器/VPS）

**不要直接暴露裸端口。** 正确做法：

```
用户 → HTTPS (443) → Nginx/Caddy → http://127.0.0.1:8642 → Hermes
```

#### Nginx 配置示例

```nginx
server {
    listen 443 ssl;
    server_name hermes.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:8642;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Authorization $http_authorization;

        # SSE 流式输出需要
        proxy_buffering off;
        proxy_cache off;
    }
}
```

#### Caddy 配置（更简单）

```
hermes.yourdomain.com {
    reverse_proxy localhost:8642
}
```

#### 防火墙

```bash
# 只开放 443，关闭 8642 对外
sudo ufw allow 443/tcp
sudo ufw deny 8642/tcp
```

### 方案 C：SSH 隧道（临时给特定人）

在你的 Mac 上执行（假设 Hermes 跑在服务器上）：

```bash
ssh -L 8642:localhost:8642 user@server-ip
```

对方在本地访问 `http://localhost:8642/v1`，流量走 SSH 加密。

---

## 前端接入示例

### Open WebUI

1. 进入 **Settings → Connections**
2. 添加 OpenAI API 连接：
   - **API Base URL**: `http://localhost:8642/v1`（或你的公网地址）
   - **API Key**: `sk-your-secret-key-here`
3. 保存，刷新页面，模型列表里会出现 `hermes-agent`

### LobeChat

1. 左下角 **设置 → 语言模型**
2. 选择 **OpenAI**
3. **API 代理地址**: `http://localhost:8642/v1`
4. **API Key**: `sk-your-secret-key-here`
5. **模型列表**留空（自动发现）

### ChatBox

1. 设置 → 模型提供方 → 自定义
2. **API Host**: `http://localhost:8642`
3. **API Key**: `sk-your-secret-key-here`
4. **模型**: `hermes-agent`

### NextChat

环境变量或设置里：

```
BASE_URL=http://localhost:8642/v1
OPENAI_API_KEY=sk-your-secret-key-here
```

### AnythingLLM

1. 工作区设置 → LLM 偏好
2. **LLM 提供商**: Generic OpenAI
3. **Base URL**: `http://localhost:8642/v1`
4. **API Key**: `sk-your-secret-key-here`
5. **模型**: `hermes-agent`

---

## 多用户隔离（Profile）

如果要给多个人用，每人独立配置、独立记忆、独立工具：

```bash
# 创建两个 Profile
hermes profile create alice
hermes profile create bob

# 配置 Alice
mkdir -p ~/.hermes/profiles/alice
cat >> ~/.hermes/profiles/alice/.env <<EOF
API_SERVER_ENABLED=true
API_SERVER_PORT=8643
API_SERVER_KEY=alice-secret-key
EOF

# 配置 Bob
mkdir -p ~/.hermes/profiles/bob
cat >> ~/.hermes/profiles/bob/.env <<EOF
API_SERVER_ENABLED=true
API_SERVER_PORT=8644
API_SERVER_KEY=bob-secret-key
EOF

# 启动两个 Gateway（后台运行）
hermes -p alice gateway &
hermes -p bob gateway &
```

前端连接：
- Alice → `http://localhost:8643/v1`，Key = `alice-secret-key`，模型名 = `alice`
- Bob → `http://localhost:8644/v1`，Key = `bob-secret-key`，模型名 = `bob`

---

## 常见问题

### Q1: `hermes gateway` 后没看到 API Server 启动

```bash
# 确认 .env 里写了 API_SERVER_ENABLED=true
cat ~/.hermes/.env | grep API_SERVER

# 确认没有语法错误（别留空格）
# 错误: API_SERVER_ENABLED = true
# 正确: API_SERVER_ENABLED=true
```

### Q2: 调用返回 401 Unauthorized

Header 里的 Key 和 `API_SERVER_KEY` 不一致，或者没传 Header。

```bash
# 测试
 curl -v http://localhost:8642/v1/models \
   -H "Authorization: Bearer sk-your-secret-key-here"
```

### Q3: 流式输出（SSE）在前端不工作

Nginx 代理需要关闭 buffering：

```nginx
proxy_buffering off;
proxy_cache off;
```

### Q4: 局域网其他设备连不上

```bash
# 1. 确认 Hermes 绑定了 0.0.0.0
cat ~/.hermes/.env | grep API_SERVER_HOST

# 2. 确认防火墙没拦端口
sudo ufw status | grep 8642

# 3. 确认内网 IP
ifconfig | grep "inet "
```

### Q5: 前端说 "No models available"

前端启动时会调 `/v1/models`，确认这个端点通：

```bash
curl http://localhost:8642/v1/models \
  -H "Authorization: Bearer sk-your-secret-key-here"
```

### Q6: 传图片报错

只支持 `image_url`（URL 或 Base64 data URI），不支持上传文件（`file`、`input_file`）。

```json
{
  "messages": [{
    "role": "user",
    "content": [
      {"type": "text", "text": "Describe this"},
      {"type": "image_url", "image_url": {"url": "data:image/png;base64,iVBORw0K..."}}
    ]
  }]
}
```

### Q7: 怎么让 Hermes 在后台一直跑

```bash
# 用 nohup
nohup hermes gateway > ~/.hermes/logs/gateway.log 2>&1 &

# 或用 tmux
tmux new-session -d -s hermes-gateway "hermes gateway"

# 或用 systemd（Linux）
# 见 macOS 部署手册的 LaunchAgent 示例，类似思路
```

---

## 相关链接

- **Hermes API Server 官方文档**: https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server
- **Open WebUI 集成指南**: https://hermes-agent.nousresearch.com/docs/integrations/open-webui
- **本仓库**: https://github.com/superGYC/Hermes

---

*本手册基于 Hermes Agent v0.14.0 官方文档编写。*  
*API Server 绑定 `127.0.0.1:8642`，暴露到公网务必配 HTTPS + 防火墙。*
