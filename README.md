# AI私人厨师智能推荐系统

基于 LangGraph 构建的多模态 AI 私人厨师助手，用户上传冰箱食材照片或手动输入食材清单，系统自动识别食材、智能检索食谱、多维度评估排序，最终输出结构化的食谱推荐方案。

## 🌟 功能特点

- **多模态食材识别**：支持上传冰箱食材图片自动识别，也支持手动输入食材清单
- **智能食谱检索**：基于 Tavily 搜索工具检索真实网页菜谱
- **多维度评估排序**：从营养价值和制作难度两个维度量化打分，优先推荐"制作简单且营养丰富"的食谱
- **结构化方案输出**：包含食谱信息、得分、推荐理由、参考图片的完整报告
- **会话状态持久化**：基于 LangGraph Checkpointer 实现对话历史保存
- **流式响应输出**：SSE 流式输出，首字延迟控制在 1~2 秒内

## 🛠️ 技术栈

| 类别 | 技术选型 |
|------|---------|
| 后端框架 | FastAPI + Uvicorn |
| AI 框架 | LangChain + LangGraph |
| 大语言模型 | 通义千问 Qwen-3.5-Plus（阿里云 DashScope） |
| 搜索工具 | Tavily Search |
| 状态持久化 | SQLite + LangGraph SqliteSaver |
| 对象存储 | 阿里云 OSS |
| 前端框架 | Next.js（React + TypeScript） |
| 包管理 | uv + Python 3.12 |
| 可观测性 | LangSmith Tracing |

## 🏗️ 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                        前端 (Next.js)                        │
│  食材上传 | 对话交互 | 食谱展示 | 流式输出渲染                │
└─────────────────────────────┬───────────────────────────────┘
                              │ HTTP + SSE
┌─────────────────────────────▼───────────────────────────────┐
│                      FastAPI 后端服务                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  chat.py │  │  oss.py  │  │ 静态资源 │                   │
│  │ 对话接口 │  │上传签名  │  │ 前端托管 │                   │
│  └────┬─────┘  └────┬─────┘  └──────────┘                   │
└───────┼──────────────┼───────────────────────────────────────┘
        │              │
┌───────▼──────────────▼───────────────────────────────────────┐
│                    Agent 核心层 (LangGraph)                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              create_agent 构建的 Agent Graph         │    │
│  │  [Model Node] ←→ [Tool Node(Tavily搜索)]            │    │
│  │         ↓                                           │    │
│  │  SqliteSaver (Checkpointer / 状态持久化)            │    │
│  └─────────────────────────────────────────────────────┘    │
└───────┬─────────────────────────────────┬───────────────────┘
        │                                 │
   ┌────▼─────┐                    ┌──────▼──────┐
   │ 通义千问  │                    │  Tavily 搜索 │
   │ (DashScope)│                   │   (Web API) │
   └────┬─────┘                    └─────────────┘
        │
   ┌────▼─────┐
   │ 阿里云OSS │
   │ (图片存储)│
   └──────────┘
```

## 📁 项目结构

```
AIAgent/
├── app/                    # 应用主目录
│   ├── __init__.py
│   ├── main.py             # FastAPI 入口文件
│   ├── agents/             # Agent 核心逻辑
│   │   └── personal_chief.py    # 私人厨师 Agent 实现
│   ├── api/                # API 路由
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── chat.py     # 对话接口
│   │       └── oss.py      # OSS 上传签名接口
│   ├── common/             # 公共模块
│   │   ├── __init__.py
│   │   └── logger.py       # 日志配置
│   ├── db/                 # 数据库文件
│   │   └── personal_chief.db
│   ├── models/             # 数据模型
│   │   ├── __init__.py
│   │   └── schemas.py      # Pydantic 模型定义
│   └── static/             # 前端静态资源
├── .env                    # 环境变量（不上传）
├── .env.example            # 环境变量模板
├── .gitignore
├── pyproject.toml          # 项目依赖配置
└── README.md
```

## 🚀 快速开始

### 环境要求

- Python >= 3.12
- uv（Python 包管理器）

### 安装步骤

2. **创建虚拟环境并安装依赖**

```bash
uv venv
uv pip install -e .
```

3. **配置环境变量**

复制 `.env.example` 文件为 `.env`，并填入相关 API 密钥：

```bash
cp .env.example .env
```

编辑 `.env` 文件：

```bash
# DashScope API Key（必需）
DASHSCOPE_API_KEY=your_api_key_here

# Tavily API Key（必需）
TAVILY_API_KEY=your_api_key_here

# 其他可选配置...
```

4. **启动服务**

```bash
uv run python -m app.main
```

服务将在 `http://127.0.0.1:8001` 启动。

## 📡 API 接口

### 对话接口

**POST** `/api/v1/chat`

请求体：
```json
{
  "prompt": "我有鸡蛋和番茄，推荐个菜",
  "image": "https://example.com/ingredients.jpg",
  "thread_id": "user_123"
}
```

**GET** `/api/v1/chat/history/{thread_id}` - 获取会话历史

**DELETE** `/api/v1/chat/history/{thread_id}` - 清空会话

### OSS 上传签名

**POST** `/api/v1/oss/sign`

请求体：
```json
{
  "filename": "ingredients.jpg"
}
```

## 🧪 核心技术

### LangGraph Agent 工作流

使用 `create_agent` 构建具备 Tool Calling 能力的 Agent：

```python
agent = create_agent(
    model=model,
    tools=[web_search],
    checkpointer=checkpointer,
    system_prompt=system_prompt,
)
```

### 多模态输入处理

支持纯文本和"图片+文本"两种输入格式：

```python
if image:
    message = HumanMessage(content=[
        {"type": "image", "url": image},
        {"type": "text", "text": prompt}
    ])
else:
    message = HumanMessage(content=prompt)
```

### 流式响应

通过 `stream_mode="messages"` 实现消息级流式输出：

```python
for chunk, metadata in agent.stream(
    {"messages": [message]},
    {"configurable": {"thread_id": thread_id}},
    stream_mode="messages"
):
    if isinstance(chunk, AIMessageChunk) and chunk.content:
        yield chunk.content
```

