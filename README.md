# AgentSociety 使用指南

本文档介绍如何在 WSL 环境下从零搭建并启动 AgentSociety 项目。

---

## 1. 系统环境要求

* 操作系统：WSL（Ubuntu）
* Python：3.12
* Node.js：v18+（推荐使用 `nvm` 管理）
* npm：对应 Node.js 版本
* Git：版本控制

## 2. 克隆仓库

```bash
git clone https://github.com/YourOrg/AgentSociety.git
cd AgentSociety
```

## 3. Python 环境配置

1. 创建并激活虚拟环境：

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

2. 安装后端依赖：

   ```bash
   pip install uv
   uv sync
   ```

## 4. 前端构建

> 确保 `npm` 是 Linux 本地版本，已剔除 `/mnt/c` 路径干扰。

1. 进入前端目录：

   ```bash
   cd frontend
   ```

2. 安装依赖并构建：

   ```bash
   npm install
   npm run build
   ```

3. 将打包产物复制到后端静态目录：

   ```bash
   mkdir -p packages/agentsociety/agentsociety/_dist
   cp -r frontend/dist/* packages/agentsociety/agentsociety/_dist/
   ```

4. 回到项目根目录：

   ```bash
   cd ..
   ```

## 5. 启动项目

> 确保当前仍在 `.venv` 虚拟环境内。

```bash
cd ~/AgentSociety
source .venv/bin/activate
agentsociety ui
```

* 默认监听：`http://localhost:8080`
* 若需局域网访问：

  ```bash
  agentsociety ui --addr 0.0.0.0:8080
  ```

## 6. 常见问题

* **npm 报错找不到**：请检查 `$PATH` 中是否包含 `/mnt/c`，并在 `~/.bashrc` 顶部添加：

  ```bash
  export PATH=$(echo "$PATH" | tr ':' '\n' | grep -v '^/mnt/c/' | paste -sd':' -)
  ```
* **前端界面无法加载**：确认已执行 `npm run build` 并复制到 `_dist`。
* **命令不存在**：请激活虚拟环境并安装后端依赖。

---

## 7. 联系方式

如有问题，请在项目 Issue 中提交或联系维护者。
