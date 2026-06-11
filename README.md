# AssistGen 智能客服 Agent

AssistGen 是一个基于 FastAPI、Vue 静态前端和大模型 API 的智能客服/知识增强 Agent 项目。当前项目已经可以在本地运行，支持登录注册、会话管理、普通聊天、商城页面右下角在线客服、文件/图片入口、LangGraph Agent 编排入口，以及 Redis、Neo4j、Ollama、GraphRAG 的扩展配置。

> 重要：仓库只提交代码和示例配置。真实 `.env`、API Key、数据库密码、上传文件、日志、虚拟环境、GraphRAG 缓存都不会提交。

## 当前可用状态

已在本机跑通过：

- FastAPI 后端服务：`http://127.0.0.1:9000`
- 静态前端页面：`http://127.0.0.1:9000`
- 商城演示页：`http://127.0.0.1:9000/ecommerce`
- API 文档：`http://127.0.0.1:9000/docs`
- MySQL 用户/会话/消息表
- 登录注册与 JWT
- `/api/chat` 流式聊天接口
- 商城右下角“在线客服”接入 `/api/chat`
- Kimi/Moonshot OpenAI-compatible 文本回复

仍需本机安装服务后才能完整启用：

- Redis：语义缓存、重复问题加速、省 API 成本
- Ollama：本地模型和本地 embedding，例如 `bge-m3`
- Neo4j：知识图谱、Text2Cypher、关系查询
- 完整 GraphRAG：图谱增强知识库问答和索引导入

## 技术栈

- 后端：Python、FastAPI、SQLAlchemy async、LangChain、LangGraph
- 数据库：MySQL
- 大模型：OpenAI-compatible API，当前可配置 Kimi/Moonshot 或 DeepSeek
- 视觉模型：OpenAI-compatible Vision API
- 缓存：Redis 语义缓存，可选
- 本地模型：Ollama，可选
- 图数据库：Neo4j，可选
- 知识增强：GraphRAG，可选
- 前端：Vue 构建后的静态资源，已内置在 `llm_backend/static/dist`

## 目录结构

```text
deepseek_agent/
├─ llm_backend/                 # FastAPI 后端和静态前端入口
│  ├─ app/
│  │  ├─ api/                   # 登录注册等 API
│  │  ├─ core/                  # 配置、数据库、日志、安全
│  │  ├─ lg_agent/              # LangGraph Agent 编排和图谱工具
│  │  ├─ models/                # SQLAlchemy 模型
│  │  ├─ services/              # LLM、Redis、Ollama、GraphRAG 等服务
│  │  ├─ graphrag/              # GraphRAG 源码/配置/示例
│  │  └─ tools/                 # 工具函数
│  ├─ static/dist/              # 前端构建产物
│  └─ main.py                   # FastAPI 应用入口
├─ requirements.txt             # Python 依赖
├─ .env.example                 # 环境变量模板
└─ README.md
```

## 本地启动

### 1. 创建虚拟环境

```powershell
cd E:\agnet\智能客服Agent\code\deepseek_agent
python -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
```

### 2. 配置环境变量

复制模板：

```powershell
Copy-Item .env.example llm_backend\.env
```

然后编辑 `llm_backend/.env`，至少配置：

```env
DEEPSEEK_API_KEY=你的模型API密钥
DEEPSEEK_BASE_URL=https://api.moonshot.cn/v1
DEEPSEEK_MODEL=moonshot-v1-8k
CHAT_SERVICE=deepseek
REASON_SERVICE=deepseek
AGENT_SERVICE=deepseek

DB_HOST=localhost
DB_PORT=3306
DB_USER=shopcare_user
DB_PASSWORD=你的MySQL密码
DB_NAME=shopcare_agent

SECRET_KEY=本地开发密钥
```

### 3. 准备 MySQL

进入 MySQL 后执行：

```sql
CREATE DATABASE IF NOT EXISTS shopcare_agent
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER IF NOT EXISTS 'shopcare_user'@'localhost'
  IDENTIFIED BY 'ShopCare_2026_local';

GRANT ALL PRIVILEGES ON shopcare_agent.* TO 'shopcare_user'@'localhost';
FLUSH PRIVILEGES;
```

如果 `.env` 中使用了别的密码，记得同步修改。

### 4. 启动后端

```powershell
cd E:\agnet\智能客服Agent\code\deepseek_agent\llm_backend
..\.venv\Scripts\python.exe -m uvicorn main:app --host 127.0.0.1 --port 9000
```

打开：

- 首页：`http://127.0.0.1:9000/`
- 商城页：`http://127.0.0.1:9000/ecommerce`
- API 文档：`http://127.0.0.1:9000/docs`

## 测试账号

可以在前端注册账号。注意前端要求密码包含大小写字母和数字，至少 8 位。

本地测试时也可以使用：

```text
邮箱：localui2@example.com
密码：Shopcare123
```

如果数据库重新初始化，这个账号需要重新注册。

## 常用接口

### 健康检查

```powershell
curl http://127.0.0.1:9000/health
```

### 流式聊天

```powershell
$body = @{
  user_id = 1
  conversation_id = 1
  messages = @(@{ role = 'user'; content = '你好，请用一句中文回答' })
} | ConvertTo-Json -Depth 5

Invoke-WebRequest -UseBasicParsing `
  -Uri 'http://127.0.0.1:9000/api/chat' `
  -Method Post `
  -ContentType 'application/json; charset=utf-8' `
  -Body $body
```

### LangGraph Agent 入口

```powershell
curl.exe -N -X POST http://127.0.0.1:9000/api/langgraph/query `
  -F "query=帮我分析这个售后问题" `
  -F "user_id=1"
```

## 可选服务说明

### Redis 缓存

用途：缓存相似问题的回答，减少重复模型调用，提高速度并节省 API 成本。

默认配置：

```env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0
```

没有 Redis 时，聊天仍可用，但日志中会出现连接 Redis 失败的错误。

### Ollama embedding

用途：本地生成文本向量，支撑 Redis 语义缓存、相似度匹配、RAG 检索。

默认配置：

```env
OLLAMA_BASE_URL=http://127.0.0.1:11434
OLLAMA_EMBEDDING_MODEL=bge-m3
```

没有 Ollama 时，在线模型聊天仍可用，但本地 embedding 和语义缓存不可用。

### Neo4j 图谱

用途：存储实体关系，支持 Text2Cypher、图谱查询、多跳关系分析。

默认配置：

```env
NEO4J_URL=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=你的Neo4j密码
NEO4J_DATABASE=neo4j
```

没有 Neo4j 时，普通聊天和商城客服仍可用，但图谱 Agent 工具不可用。

### GraphRAG

用途：把文档、实体、关系、社区报告和向量索引结合起来，提供更强的知识库问答能力。

默认配置：

```env
GRAPHRAG_PROJECT_DIR=llm_backend/app/graphrag
GRAPHRAG_DATA_DIR=data
GRAPHRAG_QUERY_TYPE=local
```

完整 GraphRAG 需要先完成索引构建、数据导入和图谱服务配置。

## 已知注意事项

- 本项目当前主要面向本地运行，不建议直接把 `.env` 上传到任何公开仓库。
- `llm_backend/.env` 中的 API Key、数据库密码不会被 Git 跟踪。
- `.venv/` 约 2GB，不应提交。
- `uploads/`、`logs/`、GraphRAG 运行缓存不应提交。
- 如果 `/ecommerce` 刷新 404，确认后端使用的是当前版本 `main.py`，其中已包含 SPA fallback。
- 如果聊天很慢，优先检查 Redis/Ollama 是否未启动导致 embedding 超时。

## GitHub 上传流程

```powershell
git init
git add .
git status
git commit -m "feat: initialize AssistGen agent project"
gh repo create deepseek-agent --private --source . --remote origin --push
```

如果远程仓库已经存在：

```powershell
git remote add origin https://github.com/<your-user>/<repo>.git
git push -u origin main
```