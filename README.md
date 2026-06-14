# 🤖 AI 扫地机器人智能客服

基于 **ReAct（推理+行动）** 模式与 **RAG（检索增强生成）** 构建的扫地机器人/扫拖一体机器人 AI 客服智能体。使用 LangChain 自定义智能体框架，支持动态提示词切换、工具调用和知识库检索。

## ✨ 核心特性

- **🧠 ReAct 智能体** — 遵循"思考→行动→观察→再思考"的推理循环，自主决定调用工具或直接回答
- **🔄 动态提示词切换** — 通过中间件机制，在普通客服模式与报告生成模式之间自动切换，两种模式使用不同的系统提示词
- **📚 RAG 知识库** — 基于 ChromaDB 向量存储 + 通义千问嵌入模型，支持 PDF/TXT 文档检索，MD5 去重增量加载
- **🔧 7 个专用工具** — 知识检索、天气查询、用户定位、使用记录查询、报告触发等
- **🌐 中文全栈** — 代码、提示词、知识库均为中文，面向中国用户

## 🏗️ 项目架构

```
├── agent/                       # 智能体层
│   ├── react_agent.py           # ReAct 智能体入口（流式输出）
│   └── tools/
│       ├── agent_tools.py       # 7 个自定义工具
│       └── middleware.py        # 3 个中间件钩子
├── config/                      # 配置
│   ├── agent.yml                # 智能体配置
│   ├── chroma.yml               # ChromaDB 向量库配置
│   ├── prompts.yml              # 提示词路径配置
│   └── rag.yml.template         # 模型/密钥配置模板
├── data/                        # 数据
│   ├── external/records.csv     # 模拟用户使用记录
│   ├── 扫地机器人100问.pdf      # 知识库 PDF
│   └── *.txt                    # 知识库文本（故障排除、维护保养、选购指南等）
├── model/
│   └── factory.py               # 模型工厂（ChatTongyi + DashScopeEmbeddings）
├── prompts/                     # 提示词
│   ├── main_prompt.txt          # 客服模式系统提示词
│   ├── rag_summarize.txt        # RAG 总结提示词
│   └── report_prompt.txt        # 报告生成模式提示词
├── rag/                         # RAG 层
│   ├── rag_service.py           # RAG 总结链
│   └── vector_store.py          # ChromaDB 向量存储服务
└── utils/                       # 工具层
    ├── config_handler.py        # YAML 配置加载
    ├── file_handler.py          # 文件处理（MD5/PDF/TXT）
    ├── logger_handler.py        # 日志系统
    ├── path_tool.py             # 路径解析
    └── prompt_loader.py         # 提示词加载
```

## ⚙️ 工作原理

### 普通客服模式

用户提问 → 智能体使用 `main_prompt.txt`（客服提示词）→ ReAct 循环推理 → 调用 RAG 检索知识库 → 结合天气等工具信息 → 给出专业回答

### 报告生成模式（动态切换）

```
用户："生成我的使用报告"
    │
    ▼
① get_user_id()           → 获取用户 ID
② get_current_month()     → 获取当前月份
③ fill_context_for_report() → 触发中间件切换提示词
                              └── context["report"] = True
    │
    ▼  system prompt 切换到 report_prompt.txt
    │
④ fetch_external_data()   → 获取用户使用数据
⑤ 模型生成 MarkDown 格式使用报告
```

## 🚀 快速开始

### 前置条件

- Python 3.10+
- 通义千问 API 密钥（[DashScope 控制台](https://dashscope.aliyun.com/)）

### 安装

```bash
# 克隆仓库
git clone https://github.com/Andychenx/Rag-.git
cd Rag-

# 安装依赖
pip install langchain langchain-core langchain-community langgraph langchain-chroma langchain-text-splitters dashscope pyyaml

# 配置 API 密钥
cp config/rag.yml.template config/rag.yml
# 编辑 config/rag.yml，填入你的 api_key
```

### 加载知识库

```bash
python -m rag.vector_store
```

### 运行智能体

```bash
python -m agent.react_agent
```

## 📦 依赖

| 包 | 用途 |
|------|------|
| `langchain` / `langchain-core` / `langchain-community` | 智能体框架、模型、工具 |
| `langgraph` | 智能体运行时和状态管理 |
| `langchain-chroma` | ChromaDB 向量存储集成 |
| `langchain-text-splitters` | 文本分割器 |
| `dashscope` | 通义千问 LLM + 嵌入 API |
| `pyyaml` | YAML 配置加载 |

## 📂 知识库

| 文件 | 内容 |
|------|------|
| `data/扫地机器人100问.pdf` | 扫地机器人常见问题 100 问 |
| `data/扫地机器人100问2.txt` | 扫地机器人常见问题续 |
| `data/扫拖一体机器人100问.txt` | 扫拖一体机器人常见问答 |
| `data/故障排除.txt` | 故障排查知识 |
| `data/维护保养.txt` | 维护保养指南 |
| `data/选购指南.txt` | 产品选购建议 |
| `data/external/records.csv` | 模拟用户使用记录（10 个用户） |

## 🔧 配置

所有配置在 `config/*.yml` 中，每个子系统一个文件：

- **`config/rag.yml`** — LLM 模型、嵌入模型、API 密钥
- **`config/chroma.yml`** — 向量库参数（chunk_size=200, k=3 等）
- **`config/prompts.yml`** — 提示词文件路径
- **`config/agent.yml`** — 外部数据路径

## 📄 文档

- [代码文件介绍与流程说明](./代码文件介绍与流程说明.md) — 每个文件的详细说明和整体流程图
- [项目优化方案](./项目优化方案.md) — 从文本切割、召回精准率、模型选择等 10 个维度的优化建议

## 🛠️ 开发

```bash
# 测试各组件
python -m utils.prompt_loader    # 测试提示词加载
python -m utils.config_handler   # 测试配置加载
python -m utils.path_tool        # 测试路径解析
python -m rag.rag_service        # 测试 RAG 总结
python -m rag.vector_store       # 测试向量检索
```

## 📄 许可证

MIT
