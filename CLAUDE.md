# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在本仓库中工作时提供指导。

## 项目概述

Langchain-Chatchat（原名 langchain-ChatGLM）是一个基于 LangChain 的开源、可离线部署的 RAG 与 Agent 问答应用，面向中文场景的知识库问答，支持开源模型。它提供 FastAPI 服务（含 OpenAI 兼容接口）和内置的 Streamlit WebUI。原独立 React 前端已删除，WebUI 位于服务端包内。

代码注释、docstring、生成的配置注释及项目文档大部分为**中文**。

## 仓库结构

Monorepo 结构。根目录的 `pyproject.toml` 仅用于文档工具——真正的项目在 `libs/` 下：

- `libs/chatchat-server/` — 主服务端项目（Poetry 包名 `langchain-chatchat`），包含两个 Python 包：
  - `chatchat/` — 应用层：CLI、基于 YAML 的配置系统、FastAPI 服务、Streamlit WebUI、知识库管理、SQLAlchemy 数据层。
  - `langchain_chatchat/` — LangChain 集成层：自定义聊天模型、Embedding、Agent 执行器/工厂、流式回调、MCP 工具。对外入口：`ChatPlatformAI`、`PlatformToolsRunnable`。
- `libs/python-sdk/` — 客户端 SDK（Poetry 项目名 `open_langchain_chatchat`）；导入包名为 `open_chatcaht`（拼写错误属历史遗留）。
- `docs/` — 安装与贡献指南（`docs/contributing/` 涵盖开发部署、API 使用、Agent、配置说明）；`docs/learning_guide.md` 是面向新人的项目学习路径。
- `markdown_docs/` — 自动生成的各模块函数文档。
- `docker/` — Dockerfile + docker-compose 部署；`tools/` — 辅助脚本（如 Xinference 模型加载 UI）。

## 开发环境搭建

依赖使用 Poetry 按包管理。IDE 源代码根目录设为 `libs/chatchat-server`。

```bash
cd libs/chatchat-server
poetry install --with lint,test -E xinference   # 或：pip install -e .
```

chatchat 的所有数据和配置位于 `CHATCHAT_ROOT`（环境变量；未设置时默认为当前目录）：

```bash
chatchat init        # 创建数据目录 + samples 知识库，建数据库表，生成 YAML 配置模板
chatchat kb -r       # 重建知识库（需要 embedding 模型已启动）
chatchat start -a    # 以独立进程启动 API 服务（端口 7861）+ WebUI（端口 8501）
```

不安装也可直接运行：`python chatchat/cli.py init|kb|start`。Swagger 文档位于 `http://<host>:7861/docs`。

chatchat 自身不加载模型——它对接 OpenAI 兼容的模型平台（Xinference、Ollama、LocalAI、FastChat、OneAPI），在 `model_settings.yaml`（`MODEL_PLATFORMS`）中配置；这些服务需先启动。避免在 chatchat 环境中导入模型推理框架，以防依赖冲突。

## 测试

在 `libs/chatchat-server` 下运行：

```bash
make test                                        # 单元测试（pytest 禁用 socket）
make test TEST_FILE=tests/unit_tests/test_x.py   # 运行单个测试文件
make extended_tests                              # 仅运行标记 `requires` 的测试
make integration_tests                           # tests/integration_tests
make scheduled_tests                             # 标记 `scheduled` 的集成测试
make test_watch                                  # 文件变更自动重跑（ptw）
make coverage                                    # 单元测试 + 覆盖率报告
```

- 自定义标记/选项在 `tests/conftest.py`：`requires`（依赖包未安装则跳过）、`scheduled`；`--only-extended` / `--only-core` 过滤选项。
- CI 在 Ubuntu/Windows/macOS × Python 3.8–3.11 上运行 `make test`，且要求测试后工作树干净（无未跟踪文件）——运行测试后请保持 git 干净。
- `tests/api/` 和 `tests/kb_vector_db/` 需要真实服务/模型；单元测试必须能离线运行。

## 代码检查

```bash
make format        # ruff format + isort
make format_diff   # 仅格式化相对 master 变更的文件
make lint          # check_pydantic.sh + lint_imports.sh + ruff + mypy
make spell_check   # codespell（忽略词表在 pyproject.toml）
```

ruff 规则：E、F、I、T201（禁止 `print`）。`scripts/check_pydantic.sh` 禁止直接 `import pydantic`——请改用兼容模块 `chatchat/server/pydantic_v1.py` / `pydantic_v2.py`（langchain 固定为 0.1.x，使用 pydantic v1；应用本身用 v2）。写 import 时优先用 `chatchat.server.pydantic_v2`（或 `langchain_chatchat.utils` 中的 `PYDANTIC_V2`）。

## 架构

### 配置系统（Settings）

`chatchat/settings.py` 定义了 `Settings` 容器，包含五组基于 YAML 的配置，模板生成在 `CHATCHAT_ROOT` 下：`basic_settings.yaml`、`kb_settings.yaml`、`model_settings.yaml`、`tool_settings.yaml`、`prompt_settings.yaml`。

- 每组配置是 pydantic `BaseSettings` 子类，基于 `chatchat/pydantic_settings_file.py`（`BaseFileSettings` + 自定义 YAML 数据源）；字段的中文 docstring 会生成为 YAML 模板中的注释。
- 配置文件修改后自动重载（基于 mtime 的缓存）。访问配置务必用 `Settings.<组名>.字段` 链式写法——把组赋给变量会使其失去自动重载。
- 新增配置项：在 `settings.py` 对应类中添加字段（带 docstring 和默认值），然后执行 `chatchat init` 重新生成模板。

### 启动与 API 服务

`chatchat/cli.py`（click）提供 `init` / `kb` / `start` 命令。`chatchat/startup.py` 将 API 服务和 WebUI 作为独立的多进程子进程（spawn 方式）启动。

`chatchat/server/api_server/server_app.py` 构建 FastAPI 应用，挂载各路由：`chat_routes`（对话/Agent）、`kb_routes`（知识库）、`tool_routes`（`/tools` 工具注册表）、`openai_routes`（OpenAI 兼容的 `/chat/completions` 等）、`server_routes`、`mcp_routes`（MCP 服务管理），另有 `/other/completion` 和静态资源 `/media`、`/img`。

### 对话编排（`chatchat/server/chat/`）

- `chat.py` — 主要的 Agent 对话接口。根据 `LLM_MODEL_CONFIG`（`llm_model` + `action_model`）构建 LLM 实例，通过 `PlatformToolsRunnable.create_agent_executor` 实例化 Agent，以 SSE 流式输出类型化状态事件（`PlatformToolsAction`、`PlatformToolsFinish`、`PlatformToolsActionToolStart/End`、`PlatformToolsLLMStatus`）。对话及序列化的 `intermediate_steps` 持久化到数据库，支持多轮 Agent 对话续接。
- `kb_chat.py` — RAG 对话：本地知识库 / 临时知识库 / 搜索引擎三种模式。
- `file_chat.py` — 文件对话：上传 → 解析 → 临时向量库（含 BM25）→ 回答。
- `completion.py`、`feedback.py` — LLM 补全和用户反馈。

### Agent 与工具

- 注册表：`chatchat/server/agents_registry/agents_registry.py` 将 `agent_type`（如 `platform-knowledge-mode`）映射到 LCEL 流水线，流水线由 `langchain_chatchat/agents/` 下的工厂函数构建（`create_structured_glm3_chat_agent`、`create_qwen_chat_agent`、`create_chat_agent`、`create_platform_tools_agent`、`create_platform_knowledge_agent`）；提示词模板在 `agents/react/create_prompt_template.py`；各模型的输出解析器在 `agents/` 下。
- 运行时：`PlatformToolsAgentExecutor`（`agents/all_tools_agent.py`）是自定义的 `AgentExecutor` 循环，处理内置适配工具、MCP 工具和终结型工具；`PlatformToolsRunnable`（`agents/platform_tools/base.py`）将其封装为类型化事件的异步迭代器。流式回调：`callbacks/agent_callback_handler.py`。
- 内置工具：`chatchat/server/agent/tools_factory/` — 每个文件通过 `@regist_tool` 装饰器注册工具（来自 `tools_registry.py`），返回 `BaseToolOutput`。包括 `search_local_knowledgebase`、`search_internet`、`text2sql`、`text2promql`、`wolfram`、`arxiv`、`calculate`、`text2image`、高德天气/POI、`url_reader`、`search_youtube`、`shell`、`wikipedia_search`。
- MCP：`langchain_chatchat/agent_toolkits/mcp_kit/`（stdio/SSE 的 `MultiServerMCPClient`、`MCPStructuredTool`）；连接信息存于数据库（`mcp_connection_repository`），可通过 WebUI 的 MCP 页面管理。

### 知识库（`chatchat/server/knowledge_base/`）

- `kb_service/` — `KBService` 基类及每种后端一个实现（faiss、milvus、zilliz、pg、es、relyt、chromadb），通过 `KBServiceFactory.get_service_by_name` 获取；由 `kb_settings.yaml` 的 `DEFAULT_VS_TYPE` 选择。
- `kb_doc_api.py` — 文档入库与检索的 API 辅助函数。
- `kb_cache/` — FAISS 实例池，含文件对话临时知识库使用的 `memo_faiss_pool`。
- `migrate.py` — 建表及目录 ↔ 向量库同步（`folder2db` 的 `recreate_vs` / `update_in_db` / `increment` 模式，及 prune 命令）。
- `chatchat/server/db/` — SQLAlchemy 2.0 模型（知识库/文件、会话、消息、MCP 连接）和 repository 层；默认 sqlite 数据库在 `data/knowledge_base/info.db`。

### 文件 RAG（`chatchat/server/file_rag/`）

自定义文档加载器（带 OCR 阈值控制的 PDF、图片、PPT、CSV）、检索器（`vectorstore`、`milvus_vectorstore`、`ensemble`——BM25 + 向量混合）、自定义文本分割器。

### WebUI

Streamlit 应用在 `chatchat/webui.py` + `chatchat/webui_pages/`（对话、RAG 对话、知识库管理、MCP 管理、模型配置）。它纯粹是 API 服务的客户端，通过 `ApiRequest`（`webui_pages/utils.py`）调用——UI 中无后端逻辑。

### 模型层

`langchain_chatchat/chat_models/` 提供 `ChatPlatformAI`（对接平台/智谱 GLM API 的 OpenAI 协议聊天模型；环境变量 `CHATCHAT_API_KEY` / `CHATCHAT_API_BASE` / `CHATCHAT_PROXY`），`embeddings/zhipuai.py` 提供 `ZhipuAIEmbeddings`。其余均使用 `langchain_openai.ChatOpenAI` 并设置 `local_wrap=True`（见 `chatchat/server/utils.py`）。

注意：langchain 固定为 0.1.x，请使用旧的导入路径（`langchain.agents`、`langchain.chains`、`langchain.prompts.chat`），不要使用新版 `langchain_core` / `langgraph` 的写法。

## 发布

仓库根目录的 `release.py` 用于递增 git 标签版本号（`vX.Y.Z`）并推送；CI 工作流（`_release.yml`、`docker-build.yaml`）根据标签构建并发布。
