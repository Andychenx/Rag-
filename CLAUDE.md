# CLAUDE.md

本文档为 Claude Code（claude.ai/code）在此仓库中编写代码时提供指导。

## 项目概述

基于 ReAct（推理+行动）模式与 RAG 构建的扫地机器人/扫拖一体机器人 AI 客服智能体。使用 LangChain（自定义智能体框架）、LangGraph、通义千问/DashScope LLM（`qwen3-max`）和 ChromaDB 向量存储。整个代码库使用中文。

## 项目结构

```
├── agent/
│   ├── react_agent.py          # ReAct 智能体入口（流式输出）
│   └── tools/
│       ├── agent_tools.py      # 7 个 LangChain 工具定义
│       └── middleware.py       # 3 个中间件钩子（工具监控、提示词切换）
├── config/
│   ├── agent.yml               # 外部数据路径配置
│   ├── chroma.yml              # ChromaDB 集合、分块大小、检索器 k 值
│   ├── prompts.yml             # 提示词 txt 文件路径
│   └── rag.yml                 # 模型名称、嵌入模型、API 密钥
├── data/
│   ├── external/records.csv    # 模拟用户使用记录
│   └── *.txt / *.pdf           # 知识库（常见问题、故障排查、指南）
├── model/
│   └── factory.py              # 工厂模式：ChatTongyi + DashScopeEmbeddings
├── prompts/
│   ├── main_prompt.txt         # 主 ReAct 系统提示词（客服）
│   ├── rag_summarize.txt       # RAG 总结提示词
│   └── report_prompt.txt       # 报告生成提示词
├── rag/
│   ├── rag_service.py          # RAG 链：检索器 → 模型 → 输出解析器
│   └── vector_store.py         # ChromaDB 带 MD5 去重的文档加载
├── utils/
│   ├── config_handler.py       # 模块级别 YAML 配置单例
│   ├── file_handler.py         # MD5、PDF/TXT 加载器、目录列表
│   ├── logger_handler.py       # 日志处理（控制台 + 文件处理器）
│   ├── path_tool.py            # 基于项目根目录的路径解析
│   └── prompt_loader.py        # 从 YAML 配置的路径加载提示词文本
└── .vscode/settings.json
```

## 架构

### 动态提示词切换

智能体通过中间件在两个模式间切换：当智能体调用 `fill_context_for_report` 时，`monitor_tool` 中间件将 `context["report"]` 设为 `True`，随后 `report_prompt_switch` 中间件（带 `@dynamic_prompt` 装饰器）返回报告生成提示词，替代原先的主客服提示词。两个模式共享同一套工具，但使用不同的系统指令。

### ReAct 智能体流程

1. 用户查询进入 `ReactAgent.execute_stream()` → 以 `context={"report": False}` 调用 `agent.stream()`
2. 模型自行决定调用工具还是直接回复（ReAct 循环）
3. 每次工具调用由 `monitor_tool` 包装（日志记录 + 上下文标志检测）
4. 每次模型调用前，`report_prompt_switch` 选择合适的提示词
5. 报告生成需遵循固定工具调用顺序：`get_user_id` → `get_current_month` → `fill_context_for_report` → `fetch_external_data`

### RAG 流水线

向量存储（ChromaDB）→ 检索器（`k=3`）→ 提示词模板（从上下文中获取参考资料）→ 模型（`qwen3-max`）→ `StrOutputParser`。文档加载时使用 MD5 去重以避免重复嵌入。

### 工具

`agent/tools/agent_tools.py` 中定义了 7 个工具：

- `rag_summarize(query)` — 从向量存储进行 RAG 检索
- `get_weather(city)` — 通过 wttr.in 获取实时天气（中文）
- `get_user_location()` — 随机选择城市
- `get_user_id()` — 从 10 个模拟用户中随机获取用户 ID
- `get_current_month()` — 随机获取月份
- `fetch_external_data(user_id, month)` — 从 CSV 获取用户使用记录
- `fill_context_for_report()` — 触发提示词切换的开关（否则无操作）

### 模型配置

所有模型通过 `model/factory.py` 中的通义千问/DashScope（`ChatTongyi`、`DashScopeEmbeddings`）进行访问。配置在 `config/rag.yml` 中：模型名称、嵌入模型名称、API 密钥。

## 关键依赖（根据导入推断）

- `langchain` / `langchain-core` / `langchain-community` — 智能体框架、模型、工具
- `langgraph` — 智能体运行时和状态管理
- `langchain-chroma` — ChromaDB 向量存储集成
- `langchain-text-splitters` — `RecursiveCharacterTextSplitter`
- `dashscope` / `langchain-community.chat_models.tongyi` — 通义千问 LLM + 嵌入
- `pyyaml` — YAML 配置加载

## 运行项目

```bash
# 将知识库文档加载到 ChromaDB
python -m rag.vector_store

# 运行智能体（使用硬编码查询进行交互测试）
python -m agent.react_agent
```

## 运行各组件

```bash
# 测试 RAG 总结
python -m rag.rag_service

# 测试 RAG 向量存储检索
python -m rag.vector_store

# 测试配置加载
python -m utils.config_handler

# 测试提示词加载
python -m utils.prompt_loader

# 测试路径解析
python -m utils.path_tool

# 测试日志
python -m utils.logger_handler
```

## 配置

所有配置位于 `config/*.yml` 中——每个子系统一个 YAML 文件。它们在 `utils/config_handler.py` 中作为模块级单例加载。要更改 LLM 模型、嵌入模型或 API 密钥，编辑 `config/rag.yml`。要更改分块大小或检索器 k 值，编辑 `config/chroma.yml`。

## API 密钥

DashScope/通义千问 API 密钥存储在 `config/rag.yml` 的 `api_key` 字段中（该文件已被 gitignore）。复制 `config/rag.yml.template` 为 `config/rag.yml` 并填入你的密钥。
