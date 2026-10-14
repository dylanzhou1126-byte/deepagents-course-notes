# 2026-10-14 Task00 · 环境准备与 AgentSeek 主模板（Windows 原生环境）

> 预备阶段要求：安装 Python 3.12/3.13、uv、node，准备模型平台（SiliconFlow）API Key，跑通 notebook 最小实验；Task0 要求装 AgentSeek、生成主模板并查看 info / task / doctor。
> 本机为 Windows 原生环境（未用 WSL2），过程记录如下。

## 1. 本机已有依赖

| 组件 | 版本 | 说明 |
|---|---|---|
| Python | 3.13.13 / 3.12.8 | py launcher 多版本共存，位于 D:\python |
| Node.js | v22.23.2 | 前端工具链需要 |
| npm | 10.9.8 | 随 Node 安装 |
| git | 2.54.0 | 已配置 user.name=dylanzhou1126-byte |

缺失项：`uv` 包管理器、WSL2（未装，直接使用原生 Windows）。

## 2. 安装 uv

- `winget install uv` 失败：报 create_symlink 权限问题（winget 安装器在用户目录创建符号链接被拒）。
- 改用 Python 自带 pip 安装，成功：
  ```
  py -m pip install uv
  ```
- 安装结果：uv 0.13.0，位于 `D:\python\Scripts\uv.exe`，已在 PATH。

## 3. 克隆课程仓库

- 首次 `git clone https://github.com/datawhalechina/deepagents-in-action` 失败。
- 原因：git 全局代理配置 `http.proxy http://127.0.0.1:32081`，但该代理端口当时未运行。
- 解决：命令行临时绕过失效代理（不改全局配置）：
  ```
  git -c http.proxy= -c https.proxy= clone https://github.com/datawhalechina/deepagents-in-action
  ```
- 仓库落在 `D:\DOU\deepagents-in-action`，最新提交 f93fe0a（2026-10-12）。

## 4. Notebook 实验环境

在仓库根目录执行：

```
uv lock --project notebooks              # 首次 uv.lock 为旧格式，先更新锁文件
uv sync --project notebooks --locked     # 同步依赖，共 110 个包
uv run --project notebooks --locked python -m course_notebooks.run template
```

结果：`template: passed (offline)` ✅

- 核心版本：deepagents 0.7.22、langchain 1.4.3、langchain-core 1.6.6、langgraph 1.2.13
- 已注册 Jupyter 内核 `deepagents-course`（显示名 "Deep Agents course"）
- 复制了 `.env` 模板，等待填入模型平台 Key

## 5. 模型平台

- 平台：SiliconFlow（硅基流动），OpenAI 兼容接口
- 配置项（写入 `.env`）：
  - `OPENAI_API_KEY=sk-...`
  - `OPENAI_API_BASE=https://api.siliconflow.cn/v1`
  - `AGENTSEEK_MODEL=deepseek-ai/DeepSeek-V3.2`

## 6. 安装 AgentSeek

```
uv tool install --upgrade agentseek
```

- 版本：agentseek 0.1.6
- 安装后 PATH 未生效，执行 `uv tool update-shell` 将 `C:\Users\Zhou Quan\.local\bin` 加入 PATH。

## 7. 生成主模板

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

## 8. 配置与依赖安装

- 在 `.env` 填入 SiliconFlow Key（DeepSeek-V3.2 模型）。
- `agentseek task sync`：成功，安装 121 个包。
- `agentseek task frontend`：模板命令 `npm install --prefix frontend` 在此 npm 版本报 ENOENT（找不到 package.json）。
  解决：进入 `frontend` 目录后直接 `npm install`，成功安装 274 个包。
- `agentseek doctor`：16 项全绿 ✅
- `agentseek task course-check`：14 passed ✅

## 9. 关键坑：Windows 不支持嵌入式 SeekDB

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

## 10. 持久化验证

- 后端重启后，通过会话链接 `http://127.0.0.1:5174/?thread=<id>` 可恢复同一对话。
- 证明 SQLite checkpointer 正常工作（跨重启保留会话状态）。

## 引用来源

- 课程仓库：https://github.com/datawhalechina/deepagents-in-action
- Notebook 实验说明：https://github.com/datawhalechina/deepagents-in-action/blob/main/notebooks/README.md
- AgentSeek 准备篇：课程仓库内 `content/pre01-agentseek-create.md`
- AgentSeek 模板仓库：https://github.com/agentseek-ai/agentseek-templates
