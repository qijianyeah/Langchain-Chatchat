# 学习指南

本文介绍如何从零开始学习 Langchain-Chatchat 项目，按"先跑通、再读主线、后深入"的顺序进行。

## 阶段 1：先把项目跑起来（体验驱动）

1. 按 README 快速上手：`chatchat init` → `chatchat kb -r` → `chatchat start -a`
2. 在 WebUI 上体验四类功能，每类对应一条代码链路，后面阅读代码时一一对应：

| 功能 | 对应代码 |
|------|----------|
| 纯 LLM 对话 | `chatchat/server/chat/chat.py` |
| RAG 知识库对话 | `chatchat/server/chat/kb_chat.py` |
| Agent 工具调用 | `chatchat/server/chat/chat.py` + `chatchat/server/agent/tools_factory/` |
| 文件对话 | `chatchat/server/chat/file_chat.py` |

3. 打开 `http://<host>:7861/docs` 浏览 Swagger，了解 API 全貌

## 阶段 2：理解入口和配置（先读薄的文件）

按以下顺序阅读，都不长：

1. `libs/chatchat-server/chatchat/cli.py` — 三个命令的入口，一眼看清项目能做什么
2. `libs/chatchat-server/chatchat/startup.py` — 如何拉起 API + WebUI 两个进程
3. `libs/chatchat-server/chatchat/settings.py` — 五组 YAML 配置的定义处。**重点理解**：字段的 docstring 会变成生成的 YAML 注释，配置通过 `Settings.<组>.字段` 链式访问才有自动重载（详见 [Settings 使用说明](contributing/settings.md)）
4. `libs/chatchat-server/chatchat/server/api_server/server_app.py` — 全部路由的总装图

## 阶段 3：跟住 RAG 主链路（项目核心）

这个项目的本质是 RAG，理解这条数据流就理解了骨架：

```
文档 → 解析切分 → 向量化 → 存入向量库 → 检索 top-k → 拼 prompt → LLM 回答
```

### 入库链路（从 `chatchat kb -r` 出发）

- `chatchat/init_database.py` → `chatchat/server/knowledge_base/migrate.py` 的 `folder2db`
- `chatchat/server/knowledge_base/kb_service/base.py` 的 `KBServiceFactory` + `faiss_kb_service.py`（默认后端）
- 文件解析切分在 `chatchat/server/utils.py` 的 `KnowledgeFile` 和 `chatchat/server/file_rag/`

### 问答链路（从 `/chat/kb_chat` 接口出发）

- `chatchat/server/chat/kb_chat.py` → `chatchat/server/knowledge_base/kb_doc_api.py` 的 `search_docs` → 拼接 prompt → LLM 流式返回

### Agent 链路（进阶，从 `/chat/chat` 接口出发）

- `chatchat/server/chat/chat.py` 构建模型和工具 → `chatchat/server/agents_registry/agents_registry.py` 选择 Agent 类型 → `langchain_chatchat/agents/` 里的执行器与解析器

## 阶段 4：动手练习（建议按序）

1. **加一个简单工具**：仿照 `chatchat/server/agent/tools_factory/calculate.py`，用 `@regist_tool` 写一个自己的工具，在 WebUI 中看到并调用它——这能串起工具注册、配置、Agent 调用的全流程（参考 [Agent 与 Function Call 说明](contributing/agent.md)）
2. **改一个 prompt 模板**：修改 `prompt_settings.yaml` 中的模板，观察热重载生效
3. **加一个配置项**：在 `settings.py` 添加字段 → `chatchat init` 重新生成模板

## 学习时的注意事项

- 项目固定 langchain 0.1.x，网上查资料时注意旧版 API（`langchain.agents`、`langchain.chains`），新版 `langgraph` 的写法不适用
- 项目有 pydantic v1/v2 混用（langchain 用 v1，应用用 v2），通过 `chatchat/server/pydantic_v1.py` / `pydantic_v2.py` 区分
- `chatchat`（应用层）与 `langchain_chatchat`（LangChain 集成层）的分工是理解代码组织的关键，读文件时留意 import 来源

以阶段 3 的 RAG 主链路为核心目标——把"一个 PDF 从上传到被回答"的完整路径在代码里走一遍，这个项目就基本入门了。
