# 02 · Task0：AgentSeek 与主模板（2026-10-14）

> Task0 要求：安装 AgentSeek，跑通 offline 最小实验，生成主模板并查看 info / task / doctor。

## 1. 安装 AgentSeek

```
uv tool install --upgrade agentseek
```

- 版本：agentseek 0.1.6
- 安装后 PATH 未生效，执行 `uv tool update-shell` 将 `C:\Users\Zhou Quan\.local\bin` 加入 PATH。

## 2. 生成主模板

```
agentseek create --list-templates
```

- 模板列表含 `deepagents/content-builder`（bundled 模板，本地可用）。
- 在 `D:\DOU` 下生成项目：`content_builder_deepagent`。

```
agentseek info          # 项目信息
agentseek task --list   # sync / frontend / course-check
agentseek doctor        # 环境诊断
```

首查 doctor：3 项 fail（frontend/node_modules 未装、.env 未填、OPENAI_API_KEY 缺失）。

## 3. 配置与依赖安装

- 在 `.env` 填入 SiliconFlow Key（DeepSeek-V3.2 模型）。
- `agentseek task sync`：成功，安装 121 个包。
- `agentseek task frontend`：模板命令 `npm install --prefix frontend` 在此 npm 版本报 ENOENT（找不到 package.json）。
  解决：进入 `frontend` 目录后直接 `npm install`，成功安装 274 个包。
- `agentseek doctor`：16 项全绿 ✅
- `agentseek task course-check`：14 passed ✅

## 4. 关键坑：Windows 不支持嵌入式 SeekDB

`agentseek dev` 启动后端报错：

```
pylibseekdb not available (Linux only)
```

嵌入式 SeekDB（pylibseekdb）仅支持 Linux。通读 agentseek_api 源码后发现 SQLite 分支（SqliteCheckpointSaver + InMemorySaver + SqliteStore），在 `.env` 增加：

```
SEEKDB_EMBED=false
METADATA_DB_BACKEND=sqlite
METADATA_DB_URL=sqlite+aiosqlite:///./agentseek_metadata.db
```

同时复制 `frontend/.env`（模板提供）：

```
VITE_LANGGRAPH_API_URL=http://127.0.0.1:2024
FRONTEND_PORT=5174
```

重启后前后端均正常：

- 后端：http://127.0.0.1:2024（health 返回 `{"status":"healthy"}`）
- 前端：http://127.0.0.1:5174（200）

## 5. 持久化验证

- 后端重启后，通过会话链接 `http://127.0.0.1:5174/?thread=<id>` 可恢复同一对话。
- 证明 SQLite checkpointer 正常工作（跨重启保留会话状态）。

## 引用来源

- 课程仓库：https://github.com/datawhalechina/deepagents-in-action
- AgentSeek 准备篇：课程仓库内 `content/pre01-agentseek-create.md`
- AgentSeek 模板仓库：https://github.com/agentseek-ai/agentseek-templates
