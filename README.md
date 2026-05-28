# AI Agent Demo

> 基于 Spring AI 的生产级 AI Agent 应用Demo，集成了 RAG、Function Calling、MCP 协议、SubAgent 子代理、记忆管理、Skill 技能系统等核心能力。

[![Java](https://img.shields.io/badge/Java-21+-blue.svg)](https://www.java.com/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.0-green.svg)](https://spring.io/projects/spring-boot)
[![Spring AI](https://img.shields.io/badge/Spring%20AI-2.0-orange.svg)](https://spring.io/projects/spring-ai)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 为什么选择这个项目？

这是一个**功能完整的 AI Agent 参考实现**，适合学习或作为起点开发自己的 Agent 应用。

### 核心特性

| 特性 | 说明 |
|------|------|
| **Agent 编排引擎** | 意图识别 → RAG 检索 → 记忆管理 → 工具调用 → 响应生成 |
| **RAG 检索增强** | 多路召回（语义 + BM25 + 查询改写）+ RRF 融合 + LLM Rerank |
| **Function Calling** | 可插拔工具注册，LLM 自主决策调用工具 |
| **对话记忆管理** | 三层上下文压缩（摘要压缩 → Assistant 裁剪 → 滑动窗口）|
| **SubAgent 子代理** | 独立记忆的子代理，支持多任务并行处理 |
| **MCP 协议** | 双向支持：MCP Server 对外暴露能力 + MCP Client 连接外部服务 |
| **Command & Skill** | 两种 Prompt 模板机制，Command 用户主动调用，Skill LLM 自主决策 |
| **多模型支持** | 支持智谱 GLM、OpenAI、DeepSeek、通义千问等 OpenAI 兼容接口 |

### 技术栈

```
Spring Boot 4.0 + Spring AI 2.0-M4 + Java 21
```

---

## 快速开始

### 环境要求

- Java 21+
- Maven 3.9+

### 配置

编辑 `src/main/resources/application.properties`：

```properties
spring.ai.openai.base-url=https://open.bigmodel.cn/api/paas/v4
spring.ai.openai.api-key=你的API密钥
spring.ai.openai.chat.options.model=glm-4
spring.ai.openai.embedding.options.model=embedding-3
```

> 支持任意 OpenAI 兼容接口：智谱 GLM、OpenAI、DeepSeek、通义千问等，只需修改 base-url 和 model。

### 启动

```bash
./mvnw spring-boot:run
```

### 访问

- **Web 界面**：启动后打开 http://localhost:8080
- **API 调用**：

```bash
# 非流式对话
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "你好，介绍一下你的能力", "sessionId": "test-001"}'

# 流式对话（SSE）
curl -X POST http://localhost:8080/api/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"message": "你好", "sessionId": "test-001"}'
```

---

## 项目架构

```
┌─────────────────────────────────────────────────────────────┐
│                        AgentCore                            │
│                     （核心编排器）                            │
├─────────────────────────────────────────────────────────────┤
│  IntentRecognizer  │  ChatMemory  │  RagService  │ Tools   │
│    （意图识别）     │   （记忆管理） │  （RAG检索）  │ （工具） │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │    ChatClient   │
                    │  （大模型调用）  │
                    └─────────────────┘
```

### 对话流程

```
用户输入
    │
    ▼
意图识别（IntentRecognizer）
    │ 判断：知识问答 or 通用对话？
    ▼
RAG 注入（RagService）
    │ 检索知识库，拼入上下文
    ▼
记忆管理（ChatMemory）
    │ 自动摘要压缩 → 构建消息列表
    ▼
模型调用（ChatClient + ToolCallbacks）
    │ LLM 决策：直接回答 or 调用工具
    │ 工具执行 → 结果返回 LLM → 继续决策（ReAct 循环）
    ▼
返回最终回复
```

---

## 核心模块详解

### 1. AgentCore - 核心编排器

负责编排对话完整流程：意图识别 → RAG 注入 → 记忆管理 → 模型调用 → 工具执行。

### 2. ChatMemory - 对话记忆

三层递进的上下文压缩策略：

| 层级 | 策略 | 说明 |
|------|------|------|
| 第一层 | 摘要压缩 | 超过 16 条消息时，LLM 总结为 300 字摘要 |
| 第二层 | Assistant 裁剪 | 只保留最近 3 条 Assistant 回复 |
| 第三层 | 滑动窗口 | 超过 maxRounds×4 时，丢弃最早消息 |

### 3. RAG - 检索增强生成

完整流水线：

```
文档加载 → 文档分块 → 向量化 → 向量存储
    │
    ▼
多路召回（语义 + BM25 + 查询改写）→ RRF 融合 → Rerank 重排
    │
    ▼
LLM → 内容生成
```

**分块策略**：

| 策略 | 类型 | 说明 |
|------|------|------|
| TextSplitter | 确定规则 | 递归语义分块（默认） |
| FixedSizeSplitter | 确定规则 | 固定字符数切分 |
| SemanticChunkSplitter | 智能 | 基于语义相似度判断切分点 |
| AgenticSplitter | 智能 | LLM 判断最佳切分方式 |

### 4. Tool - 工具调用

内置工具：

| 工具名 | 功能 |
|--------|------|
| `knowledge_search` | 知识库检索 |
| `create_sub_agent` | 创建子代理 |
| `chat_with_sub_agent` | 与子代理对话 |
| `destroy_sub_agent` | 销毁子代理 |
| `{mcp_tool_name}` | 外部 MCP 工具 |
| `get_weather` | 天气查询（示例）|
| `get_stock_price` | 股票价格查询（示例）|

### 5. MCP - Model Context Protocol

- **MCP Server**：对外暴露知识库检索能力，其他 AI 应用可通过 MCP 协议调用
- **MCP Client**：动态连接外部 MCP 服务，自动发现并注册工具

### 6. Command & Skill

| 类型 | 触发方式 | 说明 |
|------|----------|------|
| Command | 用户主动 | 快捷指令，REST API 调用 |
| Skill | LLM 决策 | 注册为 ToolCallback，LLM 自主判断调用 |

---

## Web 界面功能

项目内置完整的 Web 聊天界面：

- **流式对话**：实时逐字输出 AI 回复（SSE）
- **Markdown 渲染**：代码块、表格、列表
- **命令面板**：输入 `/` 唤起快捷命令
- **会话管理**：支持清空对话历史

---

## API 接口

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/chat` | POST | 非流式对话 |
| `/api/chat/stream` | POST | 流式对话（SSE）|
| `/api/command/execute` | POST | 执行 Command |
| `/api/manage/mcp/connect` | POST | 连接 MCP 服务 |
| `/api/manage/mcp/disconnect` | POST | 断开 MCP 服务 |
| `/api/manage/mcp/list` | GET | 列出 MCP 服务 |

---

## 目录结构

```
src/main/java/com/zoujuexian/aiagentdemo/
├── AgentApplication.java          # 启动类
├── core/
│   ├── AgentCore.java            # 核心编排器
│   ├── ChatMemory.java           # 记忆管理
│   ├── IntentRecognizer.java     # 意图识别
│   ├── SubAgent.java             # 子代理
│   └── SubAgentManager.java      # 子代理管理
├── api/
│   ├── controller/               # REST API
│   │   ├── ChatController.java   # 对话接口
│   │   ├── CommandController.java # 命令接口
│   │   └── ManagerController.java # 管理接口
│   └── mcpserver/
│       └── SimpleMcpServer.java  # MCP 服务器
└── service/
    ├── rag/                      # RAG 服务
    │   ├── RagService.java
    │   ├── VectorStore.java
    │   ├── chunk/                # 分块策略
    │   ├── retrieve/             # 召回策略
    │   └── rerank/               # 重排策略
    ├── tool/                     # 工具
    │   ├── InnerTool.java
    │   └── impl/                 # 工具实现
    ├── skill/                    # 技能系统
    │   └── SkillManager.java
    └── command/                  # 命令系统
        └── CommandManager.java
```

---

## License

MIT License
