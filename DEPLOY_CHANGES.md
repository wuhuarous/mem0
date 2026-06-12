# mem0 开箱即用部署版

基于 [mem0ai/mem0](https://github.com/mem0ai/mem0) 上游项目，踩坑后整理的**可直接部署版本**。支持 OpenAI 兼容 API（Grok、通义千问等），使用 Qdrant 作为向量数据库，国内网络友好。

---

## 变更概览

### 1. 向量数据库：pgvector → Qdrant

默认向量存储从 pgvector 切换为 Qdrant：

- `server/main.py`：`vector_store.provider` 改为 `qdrant`，连接 `qdrant:6333`，集合名 `mem0-qdrant`，嵌入维度 1536
- `server/docker-compose.yaml`：新增 `qdrant` 服务（`qdrant/qdrant:latest`）及持久化卷 `qdrant_data`

### 2. Dashboard 构建优化

| 文件 | 变更 |
|---|---|
| `Dockerfile` | Node 镜像 `20-alpine` → `22-alpine` |
| `Dockerfile` | pnpm 显式指定 `10.18.3`，使用 `--no-frozen-lockfile` 避免锁文件不一致导致构建失败 |
| `package.json` | 新增 `pnpm.onlyBuiltDependencies`（sharp、unrs-resolver），跳过不必要的原生编译 |

### 3. Cookie 安全策略

`server/dashboard/src/app/api/auth/refresh/route.ts`：

- `secure` 标志从 `NODE_ENV === "production"` 改为 `SECURE_COOKIES === "true"`
- 允许在生产环境中灵活控制是否启用 HTTPS-only cookie

### 4. Docker Compose 调整

- **Postgres**：移除宿主机端口映射 `8432:5432`（仅内部网络访问，更安全）
- **Dashboard**：新增 `NEXT_PUBLIC_BASE_PATH=/mem0`，支持子路径部署
- 各服务添加 `container_name`，便于管理
- `.env.example` 删除，环境变量统一在 `docker-compose.yaml` 中声明

### 5. 开发环境优化（国内网络）

`server/dev.Dockerfile`：

- pip 源替换为清华镜像 `pypi.tuna.tsinghua.edu.cn`
- 增加超时（600s）和重试（10 次），提升国内网络环境下的依赖安装稳定性

---

## 快速启动

### 1. 克隆并启动

```bash
git clone https://github.com/wuhuarous/mem0.git
cd mem0/server
docker compose up -d
```

### 2. 服务列表

| 服务 | 地址 |
|---|---|
| mem0 API | `http://localhost:8888` |
| mem0 Dashboard | `http://localhost:3000/mem0` |
| Qdrant | 内部网络 `mem0-qdrant:6333` |
| Postgres | 内部网络 `mem0-pgvector:5432` |

---

## 自定义配置

mem0 支持任何 OpenAI 兼容的 LLM 和 Embedder。只需在服务端修改配置即可切换模型。

### 配置示例（使用 Grok + 通义千问 Embedding）

在 `server/main.py` 的 `DEFAULT_CONFIG` 中修改，或通过 API 传入以下配置：

```json
{
  "version": "v1.1",
  "vector_store": {
    "provider": "qdrant",
    "config": {
      "host": "qdrant",
      "port": 6333,
      "collection_name": "mem0_memories",
      "embedding_model_dims": 1024
    }
  },
  "llm": {
    "provider": "openai",
    "config": {
      "api_key": "your-api-key",
      "temperature": 0.2,
      "model": "grok-4.20-0309-non-reasoning",
      "openai_base_url": "https://api-proxy.ai789.top/v1",
      "max_tokens": 2000
    }
  },
  "embedder": {
    "provider": "openai",
    "config": {
      "api_key": "your-api-key",
      "model": "text-embedding-v4",
      "openai_base_url": "https://dashscope-intl.aliyuncs.com/api/v1",
      "embedding_dims": 1024
    }
  },
  "history_db_path": "/app/history/history.db"
}
```

**配置要点：**
- `llm` 和 `embedder` 都支持 `openai_base_url`，可替换为任何 OpenAI 兼容 API
- `embedding_model_dims` 必须与 embedder 模型输出维度一致（如 `text-embedding-v4` 为 1024）
- 向量数据库的 `embedding_model_dims` 也需要与此保持一致

---

## 与上游的区别

上游 `main` 分支以 pgvector 为默认向量库，本仓库改为 Qdrant，并优化了国内部署体验（镜像源、构建配置等）。其余功能与上游保持同步。