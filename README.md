# 🔍 XDeepSearch — 多智能体深度搜索与文档生成系统

基于 **FastAPI + DeepAgents + LangGraph** 的多 Agent 协作深度搜索系统。采用 Orchestrator-SubAgent 主从架构，由一个主智能体统一编排三个专业子智能体，实现从多源信息获取到文档生成的全流程自动化。

---

## 🎯 功能概览

| 功能 | 说明 |
|------|------|
| 🌐 网络搜索 | 调用 Tavily 搜索公开互联网信息，支持多角度多轮检索 |
| 🗄️ 数据库查询 | 连接 MySQL 查询企业业务数据（表结构预览 + 自定义 SQL） |
| 📚 RAGFlow 知识库 | 接入企业内部知识库，实现基于私有文档的智能问答 |
| 📄 文件上传解析 | 支持上传 MD/TXT/DOCX/PDF/XLSX 文件并自动分析 |
| 📝 文档生成 | 自动生成 Markdown 报告，并可转换为 PDF 文档 |
| 🔌 WebSocket 实时推送 | 工具调用、子智能体执行、任务结果实时回传前端 |
| 🔒 会话隔离 | ContextVar 协程级隔离，多任务并行互不干扰 |

---

## 🏗️ 技术架构

```
用户 → FastAPI Web 服务 → DeepAgents/LangGraph 主智能体

主智能体编排:
  ┌─ 网络搜索子智能体 ───→ Tavily API
  ├─ 数据库查询子智能体 ──→ MySQL
  ├─ RAGFlow 子智能体 ───→ RAGFlow API
  ├─ Markdown 生成工具 ──→ output/session_{id}
  ├─ PDF 转换工具 ───────→ output/session_{id}
  └─ 文件读取工具 ───────→ updated/session_{id}

运行时支撑:
  ContextVar 上下文隔离 → ToolMonitor 进度推送 → WebSocket 前端
```

### 技术栈

- **后端框架**: Python / FastAPI / Uvicorn
- **Agent 框架**: DeepAgents / LangGraph / LangChain
- **LLM**: Qwen（DashScope OpenAI 兼容接口）
- **网络搜索**: Tavily API
- **数据库**: MySQL（SQLAlchemy）
- **知识库**: RAGFlow SDK
- **文档生成**: Markdown + python-docx + pywin32（Word 转 PDF）
- **实时通信**: WebSocket（Starlette）
- **上下文隔离**: Python ContextVar

---

## 🚀 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
```

### 2. 配置环境变量

```bash
# 编辑 .env，填入 API 密钥和服务连接信息
```

```env
# RAGFlow 知识库
RAGFLOW_API_URL=http://your-ragflow-host
RAGFLOW_API_KEY=your-ragflow-api-key

# LLM 配置（阿里云 DashScope）
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
OPENAI_API_KEY=your-openai-compatible-key
LLM_QWEN_MAX=qwen-max

# Tavily 网络搜索
TAVILY_API_KEY=your-tavily-key

# MySQL 数据库
MYSQL_USER=root
MYSQL_PASSWORD=root
MYSQL_DATABASE=pharma_db
MYSQL_HOST=localhost
MYSQL_PORT=3306
```

### 3. 启动服务

```bash
uvicorn api.server:app --host 0.0.0.0 --port 8000 --reload
```

### 4. 标准调用流程

```
1. 前端生成 thread_id
2. 连接 WebSocket → ws://host:port/ws/{thread_id}
3. (可选) POST /api/upload 上传附件
4. POST /api/task 提交问题
5. 监听 WebSocket 接收实时进度
6. GET /api/files 查看生成文件
7. GET /api/download 下载结果文件
```

---

## 📁 项目结构

```
XDeepSearch/
├── agent/                      # 主智能体与子智能体
│   ├── main_agent.py           # 主智能体编排入口（DeepAgents）
│   ├── llm.py                  # LLM 模型初始化
│   ├── prompts.py              # 提示词加载（YAML）
│   └── subagents/
│       ├── network_search_agent.py    # 🌐 网络搜索子智能体
│       ├── database_query_agent.py    # 🗄️ 数据库查询子智能体
│       └── knowledge_base_agent.py    # 📚 RAGFlow 知识库子智能体
├── api/                        # FastAPI 服务层
│   ├── server.py               # HTTP/WebSocket 接口
│   ├── context.py              # ContextVar 会话上下文隔离
│   └── monitor.py              # WebSocket 实时进度推送
├── tools/                      # 工具层
│   ├── tavily_tool.py          # Tavily 网络搜索工具
│   ├── db_tools.py             # MySQL 数据库工具
│   ├── ragflow_tools.py        # RAGFlow 知识库工具
│   ├── upload_file_read_tool.py # 多格式文件读取工具
│   ├── markdown_tools.py       # Markdown 文档生成工具
│   └── pdf_tools.py            # MD → PDF 转换工具
├── utils/                      # 工具函数
│   ├── path_utils.py           # 会话路径解析与隔离
│   └── word_converter.py       # Word 文档转换
├── prompt/                     # 提示词配置
│   └── prompts.yml             # 各智能体系统提示词
├── rawflow/                    # RAGFlow 集成示例
├── output/                     # 任务输出目录（session 隔离）
├── updated/                    # 用户上传文件目录（session 隔离）
├── .env                        # 外部服务配置
└── requirements.txt            # Python 依赖清单
```

---

## 💡 核心设计

### Orchestrator-SubAgent 架构

系统采用**主智能体（Orchestrator）统一编排、子智能体（SubAgent）专业执行**的模式：

| 层级 | 组件 | 职责 |
|------|------|------|
| 🧠 编排层 | `main_agent.py` | 接收用户问题，决策调用哪个子智能体/工具，流式执行 |
| 🌐 网络搜索 | `network_search_agent` | 多角度 Tavily 搜索，5 轮深度检索 |
| 🗄️ 数据库 | `database_query_agent` | 表结构探索、数据预览、自定义 SQL 执行 |
| 📚 知识库 | `knowledge_base_agent` | RAGFlow 助手列表获取与问答 |
| 📝 文档 | `generate_markdown` / `convert_md_to_pdf` | Markdown/PDF 报告生成 |
| 📂 文件 | `read_file_content` | 多格式文件内容读取 |

### ContextVar 会话隔离

使用 Python `ContextVar` 实现协程级上下文隔离，确保多任务并发时：

- 每个任务有独立的 `session_dir`（输出目录）
- 每个任务有独立的 `thread_id`（WebSocket 通道）
- 深层工具调用无需显式传参，自动获取当前会话上下文

### WebSocket 实时监控

所有工具调用、子智能体激活、结果生成均通过 `ToolMonitor` 实时推送给前端：

| 消息类型 | 触发时机 |
|---------|---------|
| `session_created` | 任务目录创建完成 |
| `tool_start` | 工具开始执行 |
| `assistant_call` | 子智能体被激活 |
| `task_result` | 主智能体生成最终回答 |
| `error` | 执行异常 |

---

## ⚙️ 工具层说明

### 检索类工具

| 工具 | 文件 | 外部依赖 | 功能 |
|------|------|----------|------|
| `internet_search` | `tools/tavily_tool.py` | `TAVILY_API_KEY` | 公开互联网信息搜索 |
| `list_sql_tables` | `tools/db_tools.py` | MySQL 连接 | 列出数据库所有表 |
| `get_table_data` | `tools/db_tools.py` | MySQL 连接 | 预览表前 100 条数据 |
| `execute_sql_query` | `tools/db_tools.py` | MySQL 连接 | 执行自定义 SQL 查询 |
| `get_assistant_list` | `tools/ragflow_tools.py` | `RAGFLOW_API_URL/KEY` | 获取 RAGFlow 助手列表 |
| `create_ask_delete` | `tools/ragflow_tools.py` | `RAGFLOW_API_URL/KEY` | 向 RAGFlow 助手提问 |

### 文件类工具

| 工具 | 文件 | 功能 |
|------|------|------|
| `read_file_content` | `tools/upload_file_read_tool.py` | 读取 MD/TXT/DOCX/PDF/XLSX 文件 |
| `generate_markdown` | `tools/markdown_tools.py` | 生成 Markdown 格式报告 |
| `convert_md_to_pdf` | `tools/pdf_tools.py` | Markdown → Word → PDF 转换 |

---

## 🔌 API 接口

### HTTP 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/task` | 提交搜索/生成任务 |
| POST | `/api/upload` | 上传附件文件 |
| GET | `/api/files` | 列出输出目录文件 |
| GET | `/api/download` | 下载生成文件 |
| WS | `/ws/{thread_id}` | WebSocket 实时通信 |

### WebSocket 示例

```javascript
const ws = new WebSocket("ws://localhost:8000/ws/{thread_id}");
ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  switch(msg.type) {
    case "session_created": /* 任务目录已创建 */ break;
    case "assistant_call":  /* 子智能体开始执行 */ break;
    case "tool_start":      /* 工具调用中 */ break;
    case "task_result":     /* 最终结果 */ break;
    case "error":           /* 异常信息 */ break;
  }
};
```

---

## 🔧 主要依赖

| 组件 | 用途 |
|------|------|
| deepagents | 多 Agent 工作流编排 |
| langchain + langgraph | LangGraph 流式执行与状态管理 |
| fastapi + uvicorn | Web 服务框架 |
| openai | LLM 调用（OpenAI 兼容接口） |
| tavily-python | Tavily 网络搜索 |
| ragflow-sdk | RAGFlow 知识库集成 |
| SQLAlchemy | MySQL 数据库连接与查询 |
| python-docx | Word 文档读写 |
| pywin32 | Windows Word COM 接口（PDF 转换） |
| pypdf | PDF 文件内容提取 |
| openpyxl | Excel 文件读取 |
| PyYAML | 提示词配置解析 |

---

## ⚠️ 注意事项

- PDF 转换依赖本地 Windows + Microsoft Word + `pywin32`，Linux 环境下需替换为纯 Python 方案（如 WeasyPrint）
- `execute_sql_query` 当前无只读限制，生产环境建议仅允许 `SELECT` 语句
- 项目默认使用阿里云 DashScope 的 Qwen 模型，可通过修改 `agent/llm.py` 切换为其他 OpenAI 兼容接口
- 建议为每个任务使用独立的 `thread_id`（UUID），避免会话冲突
