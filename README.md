# AgentSociety：部署与配置指南

本文档为 AgentSociety 项目提供了一份详尽的部署与配置指南，涵盖了从零开始的本地手动部署和使用 Docker Compose 的一键部署两种模式。

---

## 目录
1.  [两种部署模式对比](#两种部署模式对比)
2.  [核心配置：`.env` 文件详解](#核心配置-env-文件详解)
    -   [数据库 (PostgreSQL / SQLite)](#数据库-postgresql--sqlite)
    -   [对象存储 (MinIO / S3)](#对象存储-minio--s3)
    -   [大语言模型 (LLM)](#大语言模型-llm)
3.  [部署方式一：使用 Docker Compose (推荐)](#部署方式一-使用-docker-compose-推荐)
    -   [先决条件](#先决条件)
    -   [步骤1：创建 `docker-compose.yml`](#步骤1-创建-docker-composeyml)
    -   [步骤2：创建并配置 `.env` 文件](#步骤2-创建并配置-env-文件)
    -   [步骤3：启动环境](#步骤3-启动环境)
    -   [步骤4：访问服务与管理](#步骤4-访问服务与管理)
4.  [部署方式二：本地手动部署 (适用于开发)](#部署方式二-本地手动部署-适用于开发)
    -   [系统环境要求](#系统环境要求)
    -   [步骤1：克隆仓库](#步骤1-克隆仓库)
    -   [步骤2：配置 Python 环境](#步骤2-配置-python-环境)
    -   [步骤3：配置前端环境与构建](#步骤3-配置前端环境与构建)
    -   [步骤4：配置 `.env` 文件](#步骤4-配置-env-文件)
    -   [步骤5：启动项目](#步骤5-启动项目)
5.  [常见问题](#常见问题)

---

## 两种部署模式对比

| 特性 | Docker Compose (推荐) | 本地手动部署 |
| :--- | :--- | :--- |
| **易用性** | **高** (一键启动所有服务) | 低 (需手动配置每个依赖) |
| **环境隔离** | **强** (所有依赖在容器内) | 弱 (依赖本地环境) |
| **一致性** | **高** (开发、测试、生产环境一致) | 低 (易受本地环境差异影响) |
| **适用场景** | 生产、团队协作、快速开始 | 深度开发、调试特定模块 |

---

## 核心配置：`.env` 文件详解

无论是 Docker 部署还是本地部署，项目都通过根目录下的 `.env` 文件来管理所有敏感和环境相关的配置。请在项目根目录创建此文件。

### 数据库 (PostgreSQL / SQLite)

用于持久化存储模拟数据。

-   **PostgreSQL (推荐)**:
    ```env
    # 格式: postgresql+asyncpg://<user>:<password>@<host>:<port>/<db_name>
    AGENTSOCIETY_DB_DSN="postgresql+asyncpg://user:password@localhost:5432/agentsociety_db"
    ```
    *在 Docker 中，`host` 应为服务名，如 `db`。*

-   **SQLite (用于快速测试)**:
    ```env
    AGENTSOCIETY_DB_DSN="sqlite+aiosqlite:///./agentsociety_data/database.db"
    ```

### 对象存储 (MinIO / S3)

用于存储地图、日志等大文件。

-   **MinIO / S3 兼容服务**:
    ```env
    AGENTSOCIETY_S3_ENDPOINT_URL="http://localhost:9000"
    AGENTSOCIETY_S3_ACCESS_KEY_ID="minioadmin"
    AGENTSOCIETY_S3_SECRET_ACCESS_KEY="your_password"
    AGENTSOCIETY_S3_BUCKET_NAME="agentsociety"
    AGENTSOCIETY_S3_FORCE_PATH_STYLE="true" # MinIO 必须为 true
    ```
    *在 Docker 中，`AGENTSOCIETY_S3_ENDPOINT_URL` 的 `host` 应为服务名，如 `http://minio:9000`。*

### 大语言模型 (LLM)

配置驱动智能体思考的语言模型。

-   **OpenAI**:
    ```env
    OPENAI_API_KEY="sk-..."
    AGENT_DEFAULT_MODEL_NAME="gpt-4o"
    # OPENAI_API_BASE="https://your-proxy/v1" # (可选代理)
    ```

-   **DeepSeek (通过阿里云 DashScope)**:
    ```env
    # 1. 设置 Provider
    AGENT_DEFAULT_LLM_PROVIDER=deepseek
    # 2. 填入阿里云 DashScope 的 API Key
    OPENAI_API_KEY="sk-..." # <-- 填阿里云的 Key
    # 3. 指定 DeepSeek 模型
    AGENT_DEFAULT_MODEL_NAME="deepseek-r1"
    ```

---

## 部署方式一：使用 Docker Compose (推荐)

此方法将应用、数据库和对象存储打包，一键启动。

### 先决条件
-   [Docker](https://www.docker.com/)
-   [Docker Compose](https://docs.docker.com/compose/install/)

### 步骤1：创建 `docker-compose.yml`
在项目根目录创建 `docker-compose.yml` 文件：
```yaml

services:
  # AgentSociety 应用服务
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: agentsociety_app
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
      minio:
        condition: service_healthy
    command: agentsociety ui --config-base64 eyJhZGRyIjogIjAuMC4wLjA6ODA4MCJ9

  # PostgreSQL 数据库服务
  db:
    image: postgres:16
    container_name: agentsociety_db
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB" ]
      interval: 10s
      timeout: 5s
      retries: 5

  # MinIO 对象存储服务 (S3 兼容)
  minio:
    # 把不可用的 RELEASE 标签改为 latest，或者指定一个存在的 Release
    image: minio/minio:latest
    container_name: agentsociety_minio
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    command: server /data --console-address ":9001"
    healthcheck:
      # 如果你安装了 mc 客户端，可保留；否则也可以改成 curl
      test: [ "CMD", "mc", "alias", "set", "local", "http://localhost:9000", "${MINIO_ROOT_USER}", "${MINIO_ROOT_PASSWORD}" ]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  minio_data:
```

### 步骤2：创建并配置 `.env` 文件
根据您的需求（如使用 DeepSeek），创建一个完整的 `.env` 文件。

**`.env` 示例 (用于 Docker Compose + DeepSeek):**
```env
# --- 数据库 (PostgreSQL) ---
POSTGRES_USER=agentsociety
POSTGRES_PASSWORD=your_strong_password
POSTGRES_DB=agentsociety
AGENTSOCIETY_DB_DSN="postgresql+asyncpg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}"

# --- 对象存储 (MinIO) ---
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=your_strong_password
AGENTSOCIETY_S3_ENDPOINT_URL="http://minio:9000"
AGENTSOCIETY_S3_ACCESS_KEY_ID=${MINIO_ROOT_USER}
AGENTSOCIETY_S3_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}
AGENTSOCIETY_S3_BUCKET_NAME=agentsociety
AGENTSOCIETY_S3_FORCE_PATH_STYLE=true

# --- 大语言模型 (DeepSeek) ---
AGENT_DEFAULT_LLM_PROVIDER=deepseek
OPENAI_API_KEY="sk-..." # <-- 填阿里云 DashScope 的 API Key
AGENT_DEFAULT_MODEL_NAME="deepseek-v2"
```

### 步骤3：启动环境
在项目根目录运行：
```bash
docker-compose up --build -d
```
- `--build`：首次启动或代码更新后使用。
- `-d`：后台运行。

### 步骤4：访问服务与管理
-   **AgentSociety Web UI**: `http://localhost:8000`
-   **MinIO Console**: `http://localhost:9001`
-   **查看日志**: `docker-compose logs -f app`
-   **停止服务**: `docker-compose down`
-   **停止并清除数据**: `docker-compose down -v`
-   **正常打开**: `docker-compose up -d` (如果代码有更新，用 `docker-compose up --build -d`)。

---

## 部署方式二：本地手动部署 (适用于开发)

### 系统环境要求
-   OS: Linux (推荐 WSL-Ubuntu)
-   Python: 3.12+
-   Node.js: 18+
-   Git

### 步骤1：克隆仓库
```bash
git clone https://github.com/YourOrg/AgentSociety.git
cd AgentSociety
```

### 步骤2：配置 Python 环境
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e "packages/agentsociety[dev]" 或者 pip install uv & uv sync
```

### 步骤3：配置前端环境与构建
> 仅在修改了 `frontend/` 目录下的代码后需要执行。
```bash
cd frontend
npm install
npm run build
cd ..
```
*构建产物将由 Vite 自动放置到后端所需的位置。*

### 步骤4：配置 `.env` 文件
参考 [核心配置](#核心配置-env-文件详解) 章节，创建并配置您的 `.env` 文件。确保数据库和 S3 的 `host` 指向 `localhost`。

### 步骤5：启动项目
确保您已激活 Python 虚拟环境，并已安装所有依赖。
> 确保当前仍在 `.venv` 虚拟环境内。

```bash
# 运行数据库迁移 (首次启动或数据库模型变更后)
agentsociety db upgrade

cd ~/AgentSociety
source .venv/bin/activate
agentsociety ui
```

* 默认监听：`http://localhost:8080`
* 若需局域网访问：

  ```bash
  agentsociety ui --addr 0.0.0.0:8080
  ```

---

## 常见问题
-   **`command not found: agentsociety`**: 请确保您已激活项目的 Python 虚拟环境 (`source .venv/bin/activate`)。
-   **前端界面 404 或无法加载**: 如果是本地部署，请确保已成功执行前端构建 (`npm run build`)。
-   **数据库连接失败**: 检查 `.env` 文件中的 `AGENTSOCIETY_DB_DSN` 配置是否正确，并确保数据库服务正在运行且网络可达。
-   **WSL 中 npm 命令问题**: 如遇 `npm` 路径问题，请确保您的 `$PATH` 变量优先使用 Linux 的路径，而非 Windows 的 `/mnt/c/...` 路径。
* **npm 报错找不到**：请检查 `$PATH` 中是否包含 `/mnt/c`，并在 `~/.bashrc` 顶部添加：
  ```bash
  export PATH=$(echo "$PATH" | tr ':' '\n' | grep -v '^/mnt/c/' | paste -sd':' -)
  ```
* **前端界面无法加载**：确认已执行 `npm run build` 并复制到 `_dist`。
* **命令不存在**：请激活虚拟环境并安装后端依赖。
