

> 目标：
> 
> - 形成一份 **人工可阅读的技术笔记**
>     
> - 同时可作为 **Codex 规划开发任务时的输入上下文**
>     

---

# 1. 总体结论（Executive Summary）

当前最优的 AI 开发工具组合与任务分工：

|阶段|工具|
|---|---|
|架构设计 / 复杂问题|Codex|
|编码 / 调试循环|Copilot|
|高 token 批处理任务|Claude Code + DeepSeek|
|素材处理 / 知识库 / RAG|本地 5090 + 开源模型|

核心原则：

- **高价值推理 → 云端强模型**
    
- **高 token 任务 → 本地 GPU**
    
- **日常 coding → Copilot**
    
- **设计与规划 → Codex**
    

这种组合可以：

- 最小化 API token 成本
    
- 最大化开发吞吐量
    
- 减少 premium request 消耗
    
- 提高自动化程度
    

---

# 2. Copilot Premium Request 机制

## 2.1 什么是 Premium Request

Copilot 有两类请求：

|类型|模型|是否消耗 premium|
|---|---|---|
|代码补全|小模型|否|
|简单 chat|小模型|否|
|复杂推理|GPT-4 / Claude|是|

计费单位：

```
1 premium request = 1 次高级模型调用
```

**不是按 token 计费。**

---

## 2.2 Premium Request 实际消耗

典型 feature 开发循环：

```
写代码
生成测试
运行测试
分析错误
修复 bug
```

可能调用模型：

```
3-8 次
```

平均：

```
1 feature ≈ 1-5 premium requests
```

---

## 2.3 Copilot 每月可支持开发规模

Copilot Pro：

```
300 premium requests / month
```

假设：

```
平均 3 requests / feature
```

则：

```
≈ 100 features / month
```

---

# 3. Codex 与 Copilot 的最佳分工

## 3.1 Codex（设计层）

适合：

- 架构设计
    
- 复杂 debugging
    
- 代码库理解
    
- refactor 规划
    
- 技术方案比较
    

输出应结构化为：

```
plan.md
tasks.md
acceptance.md
risk.md
```

---

## 3.2 Copilot（执行层）

适合：

- 编写代码
    
- 小规模 refactor
    
- 单元测试
    
- 局部 debugging
    
- IDE 迭代
    

优化原则：

```
bounded task
单模块修改
明确 spec
```

避免：

```
开放式设计问题
全局重构
大规模 reasoning
```

---

# 4. Claude Code + DeepSeek 的角色

最适合：

```
高 token
低复杂度
批处理
```

例如：

- 文档处理
    
- 日志分析
    
- 数据清洗
    
- 知识库生成
    
- metadata 提取
    
- 批量 markdown / json 处理
    

原因：

```
DeepSeek token 成本极低
```

相比 Claude Sonnet：

```
≈ 10-30x cheaper
```

---

# 5. 本地 5090 GPU 的经济性

RTX 5090 规格：

- 32GB VRAM
    
- 575W TGP
    
- ≈2000 USD 价格
    

---

## 5.1 本地推理成本模型

每日成本估算：

```
电费 + 折旧 ≈ $2.5 / day
```

8小时推理能力：

```
≈ 3M tokens
```

一个月：

```
≈ 86M tokens
```

单位成本：

```
≈ $0.8 / 1M tokens
```

---

## 5.2 与 API 成本对比

|方案|成本 / 1M tokens|
|---|---|
|本地 5090|≈ $0.8|
|DeepSeek API|≈ $0.3-1|
|Claude Sonnet|≈ $3-15|

结论：

```
本地 ≈ DeepSeek
远低于 Claude
```

---

# 6. 什么任务适合本地 GPU

最适合：

```
高输入
低输出
大量样本
批处理
```

例如：

- 原始素材整理
    
- 文档结构化
    
- 知识库构建
    
- 日志归纳
    
- RAG preprocessing
    

原因：

```
API 按 input token 收费
本地只消耗电费
```

---

# 7. Batch 推理的优势

在高输入场景：

```
batch inference > 单请求推理
```

优化技术：

- continuous batching
    
- prefix caching
    
- chunked prefill
    
- length bucketing
    

可大幅提升 GPU 利用率。

---

# 8. 推荐的本地模型组合（20-40B）

适合 RTX 5090：

### 主力模型

```
Qwen2.5-32B-Instruct
```

用途：

- 文档理解
    
- 知识库
    
- agent
    

---

### reasoning

```
QwQ-32B
```

用途：

- 推理任务
    
- 复杂分析
    

---

### 轻量模型

```
7B-14B
```

用途：

- 预处理
    
- 分类
    
- 简单 summarization
    

---

# 9. 本地推理架构

推荐三种方案：

## 9.1 简单方案

```
Ollama
Open WebUI
Qwen32B
```

优点：

- 易部署
    
- 适合个人工作站
    

---

## 9.2 高吞吐方案

```
vLLM
continuous batching
prefix caching
```

适合：

- 批处理 pipeline
    
- RAG
    

---

## 9.3 NVIDIA 性能优先

```
TensorRT-LLM
```

适合：

- 最大吞吐
    
- GPU 深度优化
    

---

# 10. 推荐整体 AI 架构

```
          Codex
            │
     设计 / 规划
            │
            ▼
        Copilot
            │
        coding / test
            │
            ▼
  Claude Code + DeepSeek
            │
      automation / ops
            │
            ▼
        本地 5090
            │
   高 token 数据处理
```

优势：

- 成本最低
    
- 吞吐最高
    
- 任务分工清晰
    

---

# 11. AI Coding Pipeline（推荐流程）

## Step 1

Codex 输出：

```
plan.md
tasks.md
acceptance.md
risk.md
```

---

## Step 2

Copilot 执行：

```
task by task
bounded scope
single module
```

---

## Step 3

Claude Code 处理：

```
logs
docs
data
automation
```

---

## Step 4

5090 批处理：

```
素材
知识库
RAG
```

---

# 12. 最关键的优化原则

真正影响成本的不是模型，而是：

### 1. 减少循环次数

```
LLM calls ↓
```

### 2. 控制上下文大小

```
context tokens ↓
```

### 3. 避免重复解释问题

```
共享 plan.md
```

---

# 13. 关键经验法则

### 本地 GPU 更值

当：

```
>30M tokens / day
```

---

### API 更值

当：

```
<2M tokens / day
```

---

# 14. 最终推荐

对于你的环境：

```
5090 GPU
Codex
Copilot Pro
Claude Code
DeepSeek
```

最佳组合：

```
Codex → design
Copilot → coding
DeepSeek → automation
5090 → high-token pipelines
Claude Sonnet → rare expert reasoning
```

这种架构：

```
成本最低
效率最高
可扩展性最好
```