

> 目标：  
> 为 AI Agent 提供一个 **相对安全、低打扰、可恢复** 的执行环境。  
> 重点不是防御恶意越狱，而是**避免模型犯傻**：误删文件、误改宿主机、误装软件、误 push、乱写目录、批处理污染环境。

---

# 1. 设计目标

本方案追求的是：

- Agent 可以少确认、连续完成任务
    
- 日常开发体验自然，不明显削弱能力
    
- 出错后容易恢复
    
- 宿主机和关键资产不容易被误伤
    

**不追求：**

- 对抗恶意提示注入
    
- 防御高强度逃逸
    
- 达到虚拟机级别的强隔离
    

因此，这是一个：

**“工程上实用的护栏系统”**，不是严格意义上的安全沙箱。

---

# 2. 总体原则

隔离设计的核心不是“检查 agent 有没有乱来”，而是：

**从一开始就限制它只能在受控范围内活动。**

建议遵循四条原则：

## 2.1 执行入口统一

不要让 agent 直接拿到宿主机 shell。  
所有命令统一通过一个包装入口，例如：

```bash
agent-run "<command>"
```

这样才能：

- 强制命令进容器
    
- 在执行前做规则检查
    
- 在执行后记录日志
    
- 给高风险命令加确认
    

---

## 2.2 工作区隔离

Agent 默认只能操作：

- 当前项目目录
    
- 少量缓存目录
    
- 专用输出目录
    
- 临时目录
    

不要直接暴露：

- 整个 home
    
- `/etc`
    
- `/var`
    
- 宿主机系统配置
    
- Docker socket
    
- SSH/GPG 敏感目录
    

---

## 2.3 快照回滚

不要奢望 agent 永不犯错。  
更现实的办法是：

**让错误发生在可恢复的地方。**

所以工作目录、数据目录、知识库目录最好放在：

- Btrfs 子卷
    
- 或其他支持快照/版本回滚的文件系统组织中
    

大任务前自动 snapshot。

---

## 2.4 高风险动作单独拦截

不需要审查所有命令。  
重点拦这些：

- `sudo`
    
- `git push`
    
- `rm -rf`
    
- 宿主机敏感路径写入
    
- 容器提权
    
- `docker.sock`
    
- 安装/卸载宿主机软件
    
- 访问生产环境
    
- secrets / key / token 修改
    

---

# 3. 建议的隔离层次

最实用的是三层护栏。

## 第一层：软规则

通过：

- `CLAUDE.md`
    
- `AGENTS.md`
    
- Copilot instructions
    
- Codex 的项目说明文件
    

告诉模型：

- 所有命令必须通过 `agent-run`
    
- 不要直接操作宿主机
    
- 不要 `sudo`
    
- 不要自动 push
    
- 只允许写指定目录
    
- 只用固定测试命令 / 构建命令
    

这层的作用是：

**减少模型犯傻概率。**

但它不是硬约束。

---

## 第二层：工具层 / Hook 层

这是最关键的一层。  
通过工具配置、hook、wrapper，把命令执行统一收口。

可实现：

- 命令透明改写成 `agent-run`
    
- 危险命令 deny / ask
    
- 限定允许访问的路径
    
- 强制使用容器内工作目录
    
- 记录审计日志
    

这层决定 agent 是否真正受控。

---

## 第三层：运行时隔离

真正执行命令时，放进：

- Docker
    
- Podman
    
- Dev Container
    
- 或更轻量的受控容器环境
    

限制：

- 非 root 用户
    
- 固定工作目录
    
- 固定挂载
    
- 不挂 `docker.sock`
    
- 不启用 `--privileged`
    
- 不使用 host network / host pid
    

这层负责把“误操作半径”缩小。

---

# 4. 推荐的最小实现方案

如果追求实用而不是复杂，可以只做下面这套。

## 4.1 基础结构

宿主机：

- 一个统一执行器：`agent-run`
    
- 项目目录：`/workspaces/project`
    
- snapshot 目录：`/snapshots/project`
    
- agent 专用工作目录：`/agent-work`
    

容器：

- 开发容器 `devbox`
    
- 用户 `dev`
    
- 工作目录 `/workspace/project`
    

---

## 4.2 目录建议

```text
/workspaces/project
/agent-work/input
/agent-work/output
/agent-work/tmp
/agent-work/logs
/snapshots/project
```

建议语义：

- `/workspaces/project`：正式项目
    
- `/agent-work/input`：素材输入
    
- `/agent-work/output`：agent 产物
    
- `/agent-work/tmp`：临时文件
    
- `/agent-work/logs`：运行日志
    
- `/snapshots/project`：快照
    

---

## 4.3 容器挂载策略

建议只挂载：

- 项目目录
    
- 输出目录
    
- 临时目录
    
- 必要缓存目录（pip / npm / cargo 等）
    

尽量不要挂载：

- `/`
    
- 整个 home
    
- `~/.ssh`
    
- `~/.gnupg`
    
- `/var/run/docker.sock`
    

如确需凭据：

- 尽量只挂只读文件
    
- 或使用临时 token
    
- 不暴露完整私钥目录
    

---

# 5. `agent-run` 的职责

`agent-run` 不是提示词，而是执行层包装器。

它应该负责：

## 5.1 强制容器执行

例如：

```bash
docker exec -u dev -w /workspace/project devbox bash -lc "$CMD"
```

这意味着：

- agent 表面上在跑 shell
    
- 实际都被强制送进容器
    

---

## 5.2 风险命令拦截

示例规则：

- 命中 `sudo` → 拒绝
    
- 命中 `git push` → 需要确认
    
- 命中 `rm -rf` 且范围大 → 确认或拒绝
    
- 命中 `/var/run/docker.sock` → 拒绝
    
- 命中宿主机敏感路径 → 拒绝
    

---

## 5.3 日志记录

记录内容建议包括：

- 时间
    
- 原始命令
    
- 实际执行命令
    
- 工作目录
    
- 退出码
    
- 修改文件列表摘要
    

这有助于：

- 排查 agent 行为
    
- 分析失败原因
    
- 回放关键操作
    

---

## 5.4 与快照联动

在以下场景前自动 snapshot：

- 大规模 refactor
    
- 批量素材处理
    
- 知识库重建
    
- 依赖升级
    
- 自动修复循环开始前
    

---

# 6. Btrfs 在隔离中的作用

Btrfs 不是安全边界，它是**恢复机制**。

最适合做三件事：

## 6.1 项目工作区快照

任务前：

```bash
btrfs subvolume snapshot /workspaces/project /snapshots/project/pre-task-xxx
```

任务失败后可：

- 对比变更
    
- 回滚整个目录
    
- 恢复部分文件
    

---

## 6.2 缓存隔离

把这些目录单独做子卷：

- pip cache
    
- npm cache
    
- cargo registry
    
- model cache
    
- vector store
    

优点：

- 污染后可单独清理
    
- 不影响主项目
    

---

## 6.3 数据处理 checkpoint

例如知识库或素材处理：

- `pre-import`
    
- `pre-clean`
    
- `pre-rewrite`
    
- `post-validated`
    

便于批处理失败时快速恢复。

---

# 7. 不同工具的隔离接法

---

## 7.1 Claude Code

最适合做：

**软规则 + hook + sandbox**

建议：

- `CLAUDE.md` 写执行规则
    
- `PreToolUse` hook 拦命令
    
- 可透明改写 Bash 为 `agent-run`
    
- 用 permissions / sandbox 限定 Bash 的边界
    

推荐程度：**最高**

原因：

- 配置自然
    
- 低打扰
    
- 最适合“少确认但不乱来”
    

---

## 7.2 Codex

最适合做：

**容器内运行 + 原生 sandbox/rules**

建议：

- 不强求外层 `agent-run`
    
- 最自然的是直接在 dev container 里运行 Codex
    
- 再叠加 Codex 自己的：
    
    - sandbox
        
    - approvals
        
    - rules
        
    - writable_roots
        

推荐思路：

**让 Codex 自己就在隔离环境里运行。**

---

## 7.3 Copilot CLI / Copilot Agent

最适合做：

**instructions + hooks + 限定工具权限**

建议：

- 用 `AGENTS.md` / copilot instructions 写规则
    
- 用 hooks 做 pre-check / post-check
    
- 限定 allowed tools / URLs / paths
    
- 如果需要更强控制，可做 plugin 或 MCP 工具封装
    

注意：

Copilot 更适合做执行层，不太适合承担复杂隔离逻辑本身。

---

# 8. 风险分级建议

最简单好用的是把命令分三类。

## A 类：自动放行

例如：

- `ls`
    
- `cat`
    
- `grep`
    
- `find`
    
- `git status`
    
- `git diff`
    
- `pytest`
    
- `npm test`
    
- `make build`
    
- `ruff`
    
- `black`
    

特点：

- 低风险
    
- 日常频繁使用
    
- 不值得打断 agent
    

---

## B 类：规则审查后放行

例如：

- 安装项目依赖
    
- 修改 CI 配置
    
- 批量生成输出
    
- 访问允许的网络资源
    
- 写缓存目录
    

特点：

- 有一定风险
    
- 但可在容器内自动做
    

---

## C 类：必须确认或拒绝

例如：

- `git push`
    
- `sudo`
    
- `rm -rf`
    
- 改宿主机文件
    
- 写 secrets
    
- 动系统服务
    
- 容器提权
    
- 改生产配置
    

特点：

- 风险高
    
- 必须单独处理
    

---

# 9. 最适合你的隔离强度

结合你的诉求：

- 不防恶意模型
    
- 主要防犯傻
    
- 希望 agent 少确认
    
- 不想明显损失能力
    

推荐强度是：

## Level 1：够用

- 容器开发环境
    
- 项目目录独立挂载
    
- `git push` 单独确认
    
- 禁止 `sudo`
    
- 禁止 `docker.sock`
    

## Level 2：推荐

- 在 Level 1 基础上加：
    
- `agent-run`
    
- Btrfs snapshot
    
- 风险命令拦截
    
- 日志记录
    
- 输出目录隔离
    

## Level 3：增强

- 任务级 checkpoint
    
- 文件变更摘要
    
- 更细粒度 path allowlist
    
- 网络域名 allowlist
    
- 更复杂 hooks
    

对你来说，**Level 2 最合适**。

---

# 10. 推荐的整体结构

```text
                ┌─────────────┐
                │  Codex      │
                │  规划/设计   │
                └─────┬───────┘
                      │
                      ▼
                ┌─────────────┐
                │ Copilot     │
                │ 编码/测试    │
                └─────┬───────┘
                      │
                      ▼
                ┌─────────────┐
                │ Claude Code │
                │ 运维/批处理  │
                └─────┬───────┘
                      │
                      ▼
                ┌─────────────┐
                │ agent-run   │
                │ 统一执行入口 │
                └─────┬───────┘
                      │
                      ▼
                ┌─────────────┐
                │ Docker/Podman│
                │ devbox容器   │
                └─────┬───────┘
                      │
                      ▼
                ┌─────────────┐
                │ Btrfs       │
                │ 快照/回滚    │
                └─────────────┘
```

---

# 11. 推荐实施顺序

不要一次做太复杂，建议分阶段落地。

## 第一阶段

先完成：

- 开发容器
    
- 固定挂载目录
    
- 非 root 用户
    
- 禁止危险挂载
    

## 第二阶段

加入：

- `agent-run`
    
- 风险命令规则
    
- 统一日志记录
    

## 第三阶段

加入：

- Btrfs 子卷
    
- 自动 snapshot
    
- 任务级 checkpoint
    

## 第四阶段

针对具体工具做：

- Claude Code hooks
    
- Codex rules / sandbox
    
- Copilot instructions / hooks
    

---

# 12. 最终结论

隔离环境的最佳思路不是：

**“让模型记住自己要小心”**

而是：

**“让模型只能在受控范围内活动，并且出错后容易恢复”**

对当前需求，最合适的方案是：

- **容器隔离执行**
    
- **`agent-run` 统一入口**
    
- **Btrfs 快照恢复**
    
- **少量高风险命令拦截**
    
- **提示文件作为软规则补充**
    

一句话总结：

> 对防“模型犯傻”而言，最有效的不是重型安全系统，而是  
> **统一执行入口 + 受控工作区 + 快照回滚 + 高风险动作拦截。**
