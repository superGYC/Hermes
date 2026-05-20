# Hermes Agent 完全离线部署手册

**版本**: v1.0  
**日期**: 2026-05-20  
**适用场景**: 服务器无外网访问，需提前准备所有依赖  
**推荐方案**: Docker 离线镜像（比逐包安装 Python 依赖更可靠）

---

## 一、方案概述

Hermes Agent 的 `uv.lock` 锁定了 **218 个 Python 依赖包**，从 `aiohttp` 到 `uvloop`，涵盖 HTTP 客户端、数据库驱动、云平台 SDK、消息网关等。在完全离线的服务器上逐个安装这些包容易出错且不可维护。

**推荐方案**：**Docker 离线镜像部署**

| 方案 | 复杂度 | 可靠性 | 维护性 | 推荐度 |
|------|--------|--------|--------|--------|
| **Docker 镜像离线导入** | 低 | 高 | 高 | ⭐⭐⭐ 首选 |
| uv cache 离线同步 | 中 | 中 | 中 | ⭐⭐ 备选 |
| pip download 逐包安装 | 高 | 低 | 低 | ⭐ 不推荐 |

**核心思路**：
1. **有网络机器**：下载官方 Docker 镜像 → `docker save` 导出
2. **传输到离线服务器**：USB / 内网文件服务器 / 堡垒机中转
3. **离线服务器**：`docker load` 导入 → 直接运行

---

## 二、依赖分析（来自 uv.lock）

### 2.1 依赖规模

```
总包数: 218 个
Python 版本要求: >=3.11
包管理器: uv (基于 Rust 的高性能 pip 替代)
```

### 2.2 关键依赖分类

| 类别 | 代表包 | 说明 |
|------|--------|------|
| **HTTP/网络** | `aiohttp`, `httpx`, `httpx-sse`, `python-socks`, `h2` | Agent 的核心通信 |
| **LLM 客户端** | `anthropic`, `openai`, `jiter`, `tiktoken` | 多模型提供商接入 |
| **数据库** | `asyncpg`, `aiosqlite`, `sqlalchemy` | 持久化存储 |
| **云平台** | `alibabacloud-*`, `azure-*`, `boto3`, `google-auth` | 阿里云/腾讯云/AWS/GCP 集成 |
| **消息网关** | `discord-py`, `dingtalk-stream`, `lark-oapi` | Discord/钉钉/飞书 |
| **Web 框架** | `fastapi`, `uvicorn`, `starlette` | API Server |
| **媒体处理** | `av`, `edge-tts`, `elevenlabs`, `faster-whisper` | 音视频/TTS/ASR |
| **工具/爬虫** | `firecrawl-py`, `exa-py`, `playwright` | 网页爬取 |
| **系统级** | `cryptography`, `cffi`, `pydantic` | 加密/校验 |

### 2.3 系统级依赖（非 Python）

Hermes Agent 的某些功能需要系统库：

| 功能 | 系统依赖 |
|------|---------|
| `cryptography` 包 | `libssl-dev`, `libffi-dev` |
| `asyncpg` (PostgreSQL) | `libpq-dev` |
| `av` (音视频) | `ffmpeg`, `libavformat-dev` |
| `playwright` (浏览器) | `chromium` 或系统浏览器 |
| `faster-whisper` (语音) | 可选 CUDA |
| 编译 C 扩展 | `gcc`, `python3-dev` |

---

## 三、方案 A：Docker 离线镜像部署（推荐）

### 3.1 有网络机器 —— 准备阶段

```bash
# 1. 拉取官方镜像
# 官方镜像已包含所有 218 个 Python 包和系统依赖
docker pull nousresearch/hermes-agent:latest

# 2. 导出为 tar 包（约 1-2GB）
docker save -o hermes-agent-offline.tar nousresearch/hermes-agent:latest

# 3. 同时拉取 Dashboard 镜像（可选）
docker pull nousresearch/hermes-dashboard:latest
docker save -o hermes-dashboard-offline.tar nousresearch/hermes-dashboard:latest

# 4. 准备 docker-compose.yml（见下方）
# 5. 准备环境变量模板 .env.example
# 6. 准备启动脚本

# 7. 所有文件打包
tar -czvf hermes-offline-deploy-v1.tar.gz \
  hermes-agent-offline.tar \
  hermes-dashboard-offline.tar \
  docker-compose.yml \
  .env.example \
  start.sh \
  offline-install.md
```

**`docker-compose.yml`**：

```yaml
version: '3.8'

services:
  hermes:
    image: nousresearch/hermes-agent:latest
    container_name: hermes-gateway
    restart: unless-stopped
    ports:
      - "8642:8642"
    volumes:
      - ./hermes-data:/opt/data
    env_file:
      - .env
    environment:
      - API_SERVER_ENABLED=true
      - API_SERVER_KEY=${API_SERVER_KEY}
    # 安全加固
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    # 如果需要访问本地模型（同一内网）
    extra_hosts:
      - "ollama.local:192.168.1.100"
      - "vllm.local:192.168.1.101"

  dashboard:
    image: nousresearch/hermes-dashboard:latest
    container_name: hermes-dashboard
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - ./hermes-data:/opt/data:ro
    depends_on:
      - hermes

volumes:
  hermes-data:
    driver: local
```

**`.env` 离线环境配置**：

```bash
# ===== 本地模型配置（完全离线） =====
# 方案 1: Ollama（内网另一台机器）
OPENAI_BASE_URL=http://ollama.local:11434/v1
OPENAI_API_KEY=ollama
OPENAI_MODEL=llama3.1:70b

# 方案 2: vLLM（内网推理服务器）
# OPENAI_BASE_URL=http://vllm.local:8000/v1
# OPENAI_API_KEY=EMPTY
# OPENAI_MODEL=Qwen/Qwen3-72B

# ===== API Server 配置 =====
API_SERVER_ENABLED=true
API_SERVER_KEY=your-offline-production-key

# ===== 可选：消息网关（如果内网有 Telegram 代理） =====
# TELEGRAM_BOT_TOKEN=...
# TELEGRAM_API_URL=http://telegram-proxy.local:8081

# ===== 完全禁用云端功能 =====
WEB_SEARCH_ENABLED=false
BROWSER_ENABLED=false
# 所有需要外网的功能在配置中禁用
```

### 3.2 传输到离线服务器

```bash
# 方法 1: USB / 移动硬盘
cp hermes-offline-deploy-v1.tar.gz /mnt/usb/

# 方法 2: 内网文件服务器（SCP）
scp hermes-offline-deploy-v1.tar.gz admin@offline-server:/opt/packages/

# 方法 3: 堡垒机中转（如果有）
scp -J bastion@jump-server hermes-offline-deploy-v1.tar.gz admin@offline-server:/opt/
```

### 3.3 离线服务器 —— 安装阶段

**前提条件**：
- 离线服务器已安装 Docker 24+ 和 Docker Compose
- 如未安装，需提前准备 Docker 的离线安装包

```bash
# ===== 步骤 1: 解压 =====
cd /opt
tar -xzvf hermes-offline-deploy-v1.tar.gz

# ===== 步骤 2: 导入 Docker 镜像 =====
docker load -i hermes-agent-offline.tar
docker load -i hermes-dashboard-offline.tar

# 验证导入
 docker images | grep hermes

# ===== 步骤 3: 准备数据目录 =====
mkdir -p /opt/hermes-data
chmod 700 /opt/hermes-data

# ===== 步骤 4: 配置环境变量 =====
cp .env.example /opt/hermes-data/.env
# 编辑 .env，填入本地模型地址和 API Key
nano /opt/hermes-data/.env

# ===== 步骤 5: 启动 =====
cd /opt
docker compose up -d

# ===== 步骤 6: 验证 =====
docker ps | grep hermes
curl -s http://localhost:8642/health | jq .
```

---

## 四、方案 B：Python 虚拟环境离线部署（备选）

如果服务器无法运行 Docker，需手动处理 218 个 Python 包。

### 4.1 有网络机器 —— 下载所有依赖

```bash
# 1. 克隆源码
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# 2. 使用 uv 导出依赖列表
uv pip compile pyproject.toml -o requirements.txt

# 3. 使用 pip 下载所有 whl（只下载，不安装）
mkdir -p /tmp/hermes-packages
pip download \
  --destination-directory /tmp/hermes-packages \
  -r requirements.txt \
  --only-binary :all: \
  --platform manylinux2014_x86_64 \
  --python-version 3.11

# 4. 下载 uv 二进制（离线服务器需要）
curl -Lo /tmp/uv https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-unknown-linux-gnu.tar.gz

# 5. 打包
tar -czvf hermes-python-packages.tar.gz \
  hermes-agent/ \
  /tmp/hermes-packages/ \
  /tmp/uv \
  install-offline.sh
```

### 4.2 离线服务器 —— 安装

```bash
# 1. 解压
tar -xzvf hermes-python-packages.tar.gz

# 2. 安装 uv
cd /tmp/uv && tar -xzf uv-x86_64-unknown-linux-gnu.tar.gz
mv uv /usr/local/bin/

# 3. 创建虚拟环境
uv venv /opt/hermes-venv --python 3.11

# 4. 从本地包安装（关键：--find-links 指向本地目录）
uv pip install \
  --find-links /tmp/hermes-packages \
  --no-index \
  -r hermes-agent/requirements.txt

# 5. 启动 Agent
source /opt/hermes-venv/bin/activate
cd hermes-agent
python -m hermes run
```

**⚠️ 注意**：此方案容易因缺少系统库（`libssl`, `libpq`, `ffmpeg`）导致编译失败，需提前在离线服务器上安装系统依赖。

---

## 五、本地模型离线部署（核心）

Hermes Agent 要完全离线运行，**必须搭配本地 LLM**。否则 Agent 启动后会尝试连接云端 API，导致报错。

### 5.1 本地模型方案对比

| 方案 | 硬件要求 | 适用场景 | 部署复杂度 |
|------|---------|---------|-----------|
| **Ollama** | GPU 16GB+ 或 CPU 32GB+ | 中小模型（7B-70B） | 低 |
| **vLLM** | GPU 80GB+ | 大模型、高并发 | 中 |
| **llama.cpp** | CPU / 边缘设备 | 低资源环境 | 低 |
| **Unsloth** | 消费级 GPU | GGUF 模型 | 中 |

### 5.2 Ollama 离线部署（推荐）

**有网络机器准备**：

```bash
# 1. 拉取 Ollama 镜像
docker pull ollama/ollama:latest
docker save -o ollama-offline.tar ollama/ollama:latest

# 2. 下载模型（需要联网）
docker run -d -v ollama-models:/root/.ollama -p 11434:11434 --name ollama ollama/ollama:latest
docker exec -it ollama ollama pull llama3.1:70b

# 3. 导出模型卷
docker run --rm -v ollama-models:/data -v $(pwd):/backup alpine tar -czvf /backup/ollama-models.tar.gz /data

# 4. 打包
 tar -czvf llm-offline.tar.gz ollama-offline.tar ollama-models.tar
```

**离线服务器部署**：

```bash
# 1. 导入镜像
docker load -i ollama-offline.tar

# 2. 恢复模型
docker volume create ollama-models
docker run --rm -v ollama-models:/data -v /opt/packages:/backup alpine \
  sh -c "cd /data && tar -xzvf /backup/ollama-models.tar.gz --strip-components=1"

# 3. 启动 Ollama
docker run -d \
  --name ollama \
  --gpus all \
  -v ollama-models:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama:latest

# 4. 验证
curl http://localhost:11434/api/tags
```

### 5.3 Hermes 连接本地 Ollama

```yaml
# docker-compose.yml 中 Hermes 的环境变量
services:
  hermes:
    image: nousresearch/hermes-agent:latest
    environment:
      - OPENAI_BASE_URL=http://ollama:11434/v1
      - OPENAI_API_KEY=ollama
      - OPENAI_MODEL=llama3.1:70b
    # 同一 docker 网络内可直接通过服务名访问
    networks:
      - hermes-net

  ollama:
    image: ollama/ollama:latest
    volumes:
      - ollama-models:/root/.ollama
    networks:
      - hermes-net

networks:
  hermes-net:
    driver: bridge

volumes:
  ollama-models:
```

---

## 六、完全离线架构图

```
┌─────────────────────────────────────────────────────┐
│                   离线服务器（无外网）                  │
│                                                      │
│  ┌─────────────────────┐   ┌─────────────────────┐ │
│  │   Hermes Agent        │   │   Ollama / vLLM     │ │
│  │   (Docker)            │◄──│   (本地 LLM)        │ │
│  │   - API: :8642        │   │   - 完全内网        │ │
│  │   - 数据: /opt/data   │   │   - 不连外网        │ │
│  └─────────────────────┘   └─────────────────────┘ │
│           │                                          │
│  ┌────────┴────────────┐                           │
│  │   持久化存储         │                           │
│  │   - memory/          │                           │
│  │   - sessions/        │                           │
│  │   - skills/          │                           │
│  │   - config.yaml      │                           │
│  └─────────────────────┘                           │
│           │                                          │
│  ┌────────┴────────────┐                           │
│  │   可选: Dashboard    │                           │
│  │   (只读监控面板)     │                           │
│  └─────────────────────┘                           │
│                                                      │
│  外部网络: ❌ 无                                     │
│  LLM 推理: ✅ 本地                                   │
│  数据存储: ✅ 本地磁盘                               │
│  备份方式: 🔐 加密后 → 内网备份服务器                 │
└─────────────────────────────────────────────────────┘
```

---

## 七、离线部署检查清单

### 部署前准备

- [ ] 离线服务器已安装 Docker 24+ 和 Docker Compose
- [ ] 服务器硬件满足最低要求（1 vCPU / 2GB RAM / 2GB 磁盘）
- [ ] 本地模型服务器已就绪（Ollama / vLLM）
- [ ] 传输介质就绪（USB / 内网文件服务器 / 堡垒机）
- [ ] 数据目录已规划（`/opt/hermes-data`，独立磁盘或加密卷）

### 传输文件清单

| 文件 | 大小 | 来源 |
|------|------|------|
| `hermes-agent-offline.tar` | ~1-2GB | `docker pull` + `docker save` |
| `hermes-dashboard-offline.tar` | ~500MB | 同上 |
| `ollama-offline.tar` | ~1GB | 同上 |
| `ollama-models.tar.gz` | 10-40GB+ | 模型大小决定 |
| `docker-compose.yml` | <1KB | 手动编写 |
| `.env` | <1KB | 根据环境配置 |

### 启动后验证

```bash
# 1. 容器状态
docker ps | grep hermes

# 2. 健康检查
curl -s http://localhost:8642/health

# 3. API 可用性
curl http://localhost:8642/v1/models \
  -H "Authorization: Bearer your-key"

# 4. 本地模型连通性
curl http://localhost:8642/v1/chat/completions \
  -H "Authorization: Bearer your-key" \
  -H "Content-Type: application/json" \
  -d '{"model": "llama3.1:70b", "messages": [{"role": "user", "content": "测试"}]}'

# 5. 数据目录检查
ls -la /opt/hermes-data/
# 应包含: memory/ sessions/ skills/ config.yaml
```

---

## 八、常见问题

### Q1: Docker 镜像拉取失败 / 网络不稳定

```bash
# 使用国内镜像源加速
docker pull registry.cn-hangzhou.aliyuncs.com/nousresearch/hermes-agent:latest
docker tag registry.cn-hangzhou.aliyuncs.com/nousresearch/hermes-agent:latest \
  nousresearch/hermes-agent:latest
```

### Q2: 模型太大，传输困难

**方案 1**: 只传输小模型（8B），离线运行后再按需加载  
**方案 2**: 在离线服务器直接挂载 NFS / SAN 存储，模型文件放在共享存储  
**方案 3**: 使用量化模型（GGUF / Q4），体积减半

### Q3: 需要 HTTPS 但没有公网证书

```bash
# 内网自签名证书
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout hermes.key -out hermes.crt \
  -subj "/CN=hermes.local" \
  -addext "subjectAltName=DNS:hermes.local,IP:192.168.1.100"

# Nginx 配置
server {
    listen 443 ssl;
    ssl_certificate /opt/hermes/hermes.crt;
    ssl_certificate_key /opt/hermes/hermes.key;
    proxy_pass http://127.0.0.1:8642;
}
```

### Q4: 怎么更新版本

离线环境更新 = 重新走一遍流程：
1. 有网络机器拉取新版本镜像 → `docker save`
2. 传输到离线服务器
3. `docker compose down && docker load -i new.tar && docker compose up -d`
4. 数据卷保持不变，配置自动继承

---

## 九、参考资源

- Hermes Agent 官方仓库: https://github.com/NousResearch/hermes-agent
- Ollama 官方文档: https://github.com/ollama/ollama
- uv 包管理器: https://github.com/astral-sh/uv
- Docker 离线安装: https://docs.docker.com/engine/install/binaries/

---

*本手册基于 Hermes Agent 开源版本和公开文档编写，具体实现以官方最新版本为准。*
