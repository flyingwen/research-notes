
# AI Agent Runtime / ContextOS 研究笔记

## 1. 研究背景

当前 AI Agent 框架大多围绕 **workflow orchestration**（例如 LangChain pipelines），但这种模式逐渐暴露出问题：

* LLM 只是 workflow 中的一个节点
* 推理逻辑被框架控制
* 上下文管理不透明
* Agent 行为难以调试

因此出现一种新的架构方向：

```
LLM autonomy
+ ContextOS
+ Observability
+ Local-first development
+ Minimal orchestration
```

代表理念：

* LLM 保持 **主要决策权**
* Runtime 负责 **上下文生命周期管理**
* 提供 **完整可观测性与调试能力**
* 支持 **本地 agent 开发环境**

类似：

```
Claude Code
Continue
Aider
```

---

# 2. Agent 系统的两种核心图结构

理解 Agent Runtime 必须区分两种结构。

---

# 2.1 Execution Graph（执行图）

记录 **系统执行路径**。

```
User Query
   │
LLM step
   │
Tool call
   │
Retrieval
   │
LLM step
   │
Result
```

这种结构可以被 **tracing 系统**捕获。

典型工具：

* OpenTelemetry
* Langfuse
* LangSmith
* AgentOps

这些工具回答的问题：

* 哪个 tool 被调用
* 哪个 LLM step latency 最大
* 哪一步消耗最多 tokens
* execution trace

---

# 2.2 Context Graph（上下文图）

真正影响 LLM 推理的是 **context composition**。

```
Prompt
 ├ system prompt
 ├ conversation history
 ├ memory nodes
 ├ retrieved documents
 ├ tool outputs
 └ skill instructions
```

每个节点都会贡献：

```
tokens
semantic influence
```

重要问题：

* 哪些 context 被使用
* 哪些 context 被忽略
* 哪些 context 占据 token
* context 的来源

这就是：

```
Context Graph
```

---

# 3. Observability 工具能力分析

---

# 3.1 OpenTelemetry

OpenTelemetry 是 CNCF 的 **分布式可观测性标准**。

核心数据模型：

```
Trace
 └ Span
     └ Attributes
```

记录：

* latency
* token usage
* tool calls
* API calls

能回答：

```
LLM latency
tool invocation
token usage
execution timeline
```

不能回答：

```
skill 是否触发
prompt composition
memory attribution
context graph
```

定位：

```
Execution Observability
```

---

# 3.2 Langfuse

Langfuse 是 **LLM Observability 平台**。

数据模型：

```
Trace
 └ Observation
     ├ Span
     ├ Generation
     └ Event
```

记录：

* LLM prompt
* tokens
* cost
* latency
* tool calls

能回答：

```
哪个 step 消耗最多 tokens
LLM prompt 内容
tool invocation
execution trace
```

局限：

```
不理解 context graph
不理解 memory
不理解 retrieval composition
```

定位：

```
LLM-aware tracing
```

---

# 4. Agent Observability 的核心缺口

当前工具无法回答：

```
哪些 skill 被加载
哪些 skill 被触发
哪些 memory node 被使用
哪个 retrieval chunk 占 context
哪些 tokens 实际影响输出
```

原因：

当前 prompt 本质是：

```
string
```

缺少：

```
context schema
context graph
token attribution
```

---

# 5. Context Graph Debugger 概念

未来 Agent Debugger 需要提供：

```
Agent Run
   │
Context Graph
   │
 ├ system prompt
 ├ memory nodes
 ├ retrieval docs
 ├ skill instructions
 ├ tool outputs
 └ conversation history
```

并提供：

### Context Composition

```
system: 200 tokens
memory: 600 tokens
retrieval: 3000 tokens
history: 2000 tokens
```

---

### Context Lineage

```
retrieval chunk
   ↑
vector search
   ↑
embedding
```

---

### Skill Triggering

```
skill loaded
skill considered
skill triggered
```

---

### Token Attribution

```
哪些 context tokens 影响 output
```

---

# 6. 接近 ContextOS 的开源项目

以下项目按类别整理。

---

# 6.1 Agent Runtime

## Continue

[https://github.com/continuedev/continue](https://github.com/continuedev/continue)

定位：

```
open-source AI coding assistant runtime
```

功能：

* IDE agent
* tool invocation
* codebase context
* local development

架构：

```
context providers
tool registry
agent loop
```

局限：

```
context graph 不可见
observability 不完善
```

---

## Aider

[https://github.com/Aider-AI/aider](https://github.com/Aider-AI/aider)

定位：

```
AI pair programming CLI
```

功能：

* git-aware context
* repo map
* automatic code edits
* multi-file reasoning

优势：

```
context selection 非常先进
```

局限：

```
没有 observability
没有 context graph
```

---

## Open Interpreter

[https://github.com/OpenInterpreter/open-interpreter](https://github.com/OpenInterpreter/open-interpreter)

定位：

```
computer-use agent
```

能力：

```
terminal control
file system operations
code execution
browser automation
```

架构：

```
LLM loop
tool runtime
```

局限：

```
observability 弱
context管理较简单
```

---

## OpenDevin

[https://github.com/OpenDevin/OpenDevin](https://github.com/OpenDevin/OpenDevin)

目标：

```
open-source Devin
```

组件：

```
planner
agent
workspace
tools
```

能力：

```
autonomous software engineer
```

局限：

```
runtime复杂
observability不足
```

---

# 6.2 Multi-Agent Runtime

## AutoGen

[https://github.com/microsoft/autogen](https://github.com/microsoft/autogen)

定位：

```
multi-agent conversation framework
```

功能：

```
agents
tools
conversation loop
```

局限：

```
偏 orchestration
context management 不透明
```

---

## CrewAI

[https://github.com/crewAIInc/crewAI](https://github.com/crewAIInc/crewAI)

定位：

```
multi-agent orchestration
```

能力：

```
role-based agents
tasks
tools
```

局限：

```
workflow-heavy
```

---

## LangGraph

[https://github.com/langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)

定位：

```
stateful agent graph runtime
```

能力：

```
state machine
agent loops
persistent state
```

局限：

```
graph orchestration
context graph 不可见
```

---

# 6.3 Observability

## OpenLLMetry

[https://github.com/traceloop/openllmetry](https://github.com/traceloop/openllmetry)

定位：

```
OpenTelemetry for LLM applications
```

功能：

```
LLM tracing
vector DB tracing
tool tracing
```

优点：

```
OpenTelemetry生态兼容
```

局限：

```
不理解 context graph
```

---

## Langtrace

[https://github.com/Scale3-Labs/langtrace](https://github.com/Scale3-Labs/langtrace)

定位：

```
LLM observability tool
```

功能：

```
execution traces
LLM monitoring
tool tracing
```

局限：

```
context attribution 不存在
```

---

## OpenLIT

[https://github.com/openlit/openlit](https://github.com/openlit/openlit)

定位：

```
zero-code LLM observability
```

功能：

```
auto tracing
token monitoring
cost monitoring
```

局限：

```
runtime context 不透明
```

---

# 6.4 Context / Knowledge Systems

## GraphRAG

[https://github.com/microsoft/graphrag](https://github.com/microsoft/graphrag)

定位：

```
graph-based retrieval
```

能力：

```
knowledge graph
entity relationships
context generation
```

优势：

```
context 更结构化
```

局限：

```
不是 agent runtime
```

---

## LlamaIndex

[https://github.com/run-llama/llama_index](https://github.com/run-llama/llama_index)

定位：

```
data framework for LLM apps
```

能力：

```
data connectors
retrieval
memory
indexes
```

局限：

```
不是完整 runtime
```

---

# 7. 当前生态结构

当前 Agent 工具生态：

```
Agent Runtime
    Continue
    Aider
    OpenDevin

Multi-Agent Framework
    AutoGen
    CrewAI
    LangGraph

Observability
    OpenLLMetry
    Langtrace
    OpenLIT

Context Systems
    GraphRAG
    LlamaIndex
```

但缺少：

```
Context Runtime
+
Context Graph
+
Context Debugger
```

---

# 8. 未来趋势

很可能出现一种新工具：

```
Agent Development Environment
```

类似：

```
VSCode
+
Chrome DevTools
```

但用于：

```
Agent Runtime
```

核心组件：

```
ContextOS
Context Graph
Agent Runtime
Observability
Debugger
```

提供能力：

```
context visualization
token attribution
skill debugging
execution tracing
```

---

# 9. 关键结论

AI Agent 生态正在从：

```
workflow orchestration
```

转向：

```
LLM autonomy
+
context runtime
+
observability
```

而真正缺失的核心组件是：

```
ContextOS
```

负责：

```
context assembly
context pruning
memory graph
prompt composition
context debugging
```

