# Claude Code 源码架构深度分析

> 基于 claude-code/src 目录的完整源码分析

---

## 目录

1. [项目概览](#1-项目概览)
2. [技术栈](#2-技术栈)
3. [整体架构图](#3-整体架构图)
4. [核心模块详解](#4-核心模块详解)
5. [核心循环机制 — Agent Loop](#5-核心循环机制--agent-loop)
6. [工具系统（Harness 的核心）](#6-工具系统harness-的核心)
7. [多代理协作系统](#7-多代理协作系统)
8. [上下文管理与压缩系统](#8-上下文管理与压缩系统)
9. [权限与安全系统](#9-权限与安全系统)
10. [为什么比 LangChain 等显式 Agent 框架更强](#10-为什么比-langchain-等显式-agent-框架更强)
11. [Harness 能力的全面体现](#11-harness-能力的全面体现)
12. [总结](#12-总结)

---

## 1. 项目概览

Claude Code 是一个**基于终端的 AI 编程助手**，它不是一个"调用 LLM 的应用"，而是一个**为 LLM 构建的完整操作系统级 harness**。用户通过对话让 AI 直接操作文件系统、执行命令、编辑代码、管理 Git 等——所有这些能力都通过精心设计的工具系统和 Agent Loop 实现。

**核心设计哲学**：不定义显式的 Agent 工作流图，而是让 LLM 自主决策工具调用序列，由 harness 提供丰富的工具和智能的环境反馈。

---

## 2. 技术栈

| 层面 | 技术 |
|------|------|
| 运行时 | **Bun** (高性能 JS/TS 运行时) |
| 语言 | **TypeScript** |
| 终端 UI | **Ink** (React 的终端渲染引擎) + 深度定制 |
| 状态管理 | 自研 Store (类 Zustand) + React Context |
| API 通信 | Anthropic SDK (支持 Direct/Bedrock/Vertex/Foundry 多 Provider) |
| 协议 | **MCP** (Model Context Protocol) |
| 构建 | Bun bundler + `feature()` 编译时死代码消除 |
| A/B 测试 | GrowthBook |
| 监控 | Datadog + OpenTelemetry |

---

## 3. 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Entry Points 入口层                          │
│  cli.tsx ──→ init.ts ──→ main.tsx ──→ replLauncher.tsx             │
│                          ↓ (非交互模式)    ↓ (交互 REPL 模式)       │
│                     QueryEngine.ts    App.tsx → REPL.tsx            │
│                                            ↓                       │
│                                      QueryEngine.ts                │
├─────────────────────────────────────────────────────────────────────┤
│                       Query Engine 查询引擎层                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  QueryEngine.ts (会话管理/系统提示构建/消息处理/状态通知)     │  │
│  │       ↓                                                      │  │
│  │  query.ts (核心 Agent Loop — 流式 API 调用 + 工具执行循环)   │  │
│  │       ↓                                                      │  │
│  │  services/api/claude.ts (API 调用/流处理/beta管理/重试)      │  │
│  └──────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│                       Tool System 工具系统层                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  StreamingToolExecutor (流式工具执行 + 并发控制)              │  │
│  │  toolOrchestration.ts (工具编排: 并行/串行智能切换)          │  │
│  │       ↓                                                      │  │
│  │  30+ 内置工具:                                               │  │
│  │  ┌─────────────┬──────────────┬──────────────┐              │  │
│  │  │ BashTool    │ FileEditTool │ FileReadTool  │              │  │
│  │  │ FileWriteTool│ GrepTool    │ GlobTool      │              │  │
│  │  │ AgentTool   │ WebSearchTool│ WebFetchTool  │              │  │
│  │  │ MCPTool     │ LSPTool     │ NotebookEdit  │              │  │
│  │  │ TeamCreate  │ SendMessage  │ TodoWriteTool │              │  │
│  │  │ ToolSearch  │ SkillTool   │ ConfigTool    │              │  │
│  │  │ ...         │ ...         │ ...           │              │  │
│  │  └─────────────┴──────────────┴──────────────┘              │  │
│  └──────────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│                    Services 服务层                                   │
│  ┌────────────┬────────────┬──────────────┬─────────────────────┐  │
│  │ compact/   │ mcp/       │ analytics/   │ lsp/                │  │
│  │ 上下文压缩  │ MCP 协议   │ 遥测分析     │ 语言服务器          │  │
│  ├────────────┼────────────┼──────────────┼─────────────────────┤  │
│  │ oauth/     │ voice/     │ plugins/     │ SessionMemory/      │  │
│  │ 认证       │ 语音输入    │ 插件系统     │ 会话记忆            │  │
│  └────────────┴────────────┴──────────────┴─────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│                    Infrastructure 基础设施层                         │
│  ┌────────────┬────────────┬──────────────┬─────────────────────┐  │
│  │ bootstrap/ │ bridge/    │ state/       │ utils/              │  │
│  │ 全局状态   │ IDE 桥接   │ 应用状态     │ 工具函数            │  │
│  ├────────────┼────────────┼──────────────┼─────────────────────┤  │
│  │ constants/ │ types/     │ migrations/  │ ink/ (定制UI框架)   │  │
│  │ 常量/提示词 │ 类型定义   │ 数据迁移     │ 终端渲染引擎        │  │
│  └────────────┴────────────┴──────────────┴─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. 核心模块详解

### 4.1 入口系统 (`entrypoints/`)

| 文件 | 职责 |
|------|------|
| `cli.tsx` | 程序真正入口。实现快速路径（`--version` 零加载延迟）、多种启动模式（REPL/MCP Server/daemon worker）|
| `init.ts` | 初始化系统：加载配置、TLS 证书、遥测、LSP、OAuth、策略限制 |
| `mcp.ts` | MCP 服务器模式——将 Claude Code 自身作为 MCP Server 暴露所有工具 |
| `agentSdkTypes.ts` | Agent SDK 公共 API 类型，供外部集成使用 |
| `sandboxTypes.ts` | 沙箱网络/文件系统权限的 Zod Schema 定义 |

**启动流程**: `cli.tsx` → `init.ts`(初始化) → `main.tsx`(CLI 参数解析 + Commander.js) → `replLauncher.tsx`(交互模式) 或 `QueryEngine.ts`(非交互模式)

### 4.2 全局状态 (`bootstrap/state.ts`)

这是一个 **1759 行的全局单例状态文件**，管理：
- 会话 ID、CWD、工作目录
- Token 计数、模型使用量
- 遥测 meter
- CLAUDE.md 配置缓存
- 权限标志、Feature Flag
- 代理颜色分配

**设计原则**：作为"叶子节点"，不导入任何业务模块，避免循环依赖。这是整个应用的状态真相来源。

### 4.3 查询引擎 (`QueryEngine.ts` + `query.ts`)

这是**整个系统的心脏**。

**QueryEngine.ts** (1296 行)：
- 管理完整对话状态
- 构建系统提示（System Prompt）
- 处理用户输入
- 会话恢复
- SDK 状态通知

**query.ts** (1730 行) — 核心 Agent Loop：
- 向 Claude API 发送流式请求
- 处理流式响应中的工具调用
- 执行工具 → 将结果反馈给模型 → 继续对话
- 自动压缩（auto-compact）触发
- Token 预算管理
- 中断与恢复处理
- max_output_tokens 恢复循环

### 4.4 API 客户端 (`services/api/`)

| 文件 | 行数 | 职责 |
|------|------|------|
| `claude.ts` | 3420 | API 调用核心——消息流处理、工具 Schema 组装、beta 管理、系统提示缓存分割 |
| `client.ts` | 390 | SDK 客户端创建——支持 Direct/Bedrock/Vertex/Foundry 多 Provider |
| `withRetry.ts` | - | 重试逻辑，支持 fallback 模型 |
| `errors.ts` | - | 错误分类与处理 |
| `promptCacheBreakDetection.ts` | - | Prompt Cache 失效检测 |

### 4.5 命令系统 (`commands/` + `commands.ts`)

**100+ 个斜杠命令**，按功能分类：

| 类别 | 命令 |
|------|------|
| 会话管理 | `/resume`, `/session`, `/compact`, `/clear`, `/export`, `/share` |
| Git 操作 | `/commit`, `/branch`, `/review`, `/diff`, `/rewind` |
| 配置 | `/config`, `/model`, `/theme`, `/permissions`, `/effort`, `/fast` |
| 开发工具 | `/doctor`, `/mcp`, `/plugin`, `/skills`, `/hooks` |
| 高级功能 | `/agents`, `/tasks`, `/plan`, `/ultraplan`, `/bridge`, `/teleport` |

### 4.6 UI 层 (`components/` + `ink/`)

基于 **Ink (React 终端渲染器)** 构建，并做了深度定制（`ink/` 下 96 个文件）：

| 关键组件 | 行数 | 职责 |
|----------|------|------|
| `Messages.tsx` | 144KB | 消息列表渲染 |
| `Message.tsx` | 77KB | 单条消息渲染 |
| `ScrollKeybindingHandler.tsx` | 145KB | 滚动键绑定处理 |
| `PromptInput/` | - | 提示输入组件族 |
| `Markdown.tsx` | - | Markdown 渲染 |
| `ModelPicker.tsx` | - | 模型选择器 |

---

## 5. 核心循环机制 — Agent Loop

Claude Code 的灵魂是 `query.ts` 中的 **Agent Loop**，这是一个 `async generator` 实现的流式循环：

```
用户输入
   ↓
┌──────────────────────────────────────────────────────┐
│                   Agent Loop (query.ts)               │
│                                                       │
│  1. 构建请求 (系统提示 + 历史消息 + 用户上下文)       │
│  2. 调用 Claude API (流式)                            │
│  3. 处理流式响应:                                     │
│     ├── 文本块 → yield 给 UI 渲染                     │
│     ├── tool_use 块 → StreamingToolExecutor 执行       │
│     └── thinking 块 → 内部推理                        │
│  4. 收集工具结果                                      │
│  5. 判断是否继续:                                     │
│     ├── 有 tool_use → 将结果追加消息，回到步骤 2      │
│     ├── end_turn → 结束循环                           │
│     ├── max_output_tokens → 恢复循环 (最多 3 次)      │
│     └── prompt_too_long → 触发自动压缩，回到步骤 2    │
│  6. 自动压缩检查 (token 阈值触发)                     │
│  7. Token 预算检查                                    │
│  8. Turn 计数检查 (maxTurns 限制)                     │
│                                                       │
│  [持续循环直到模型决定停止或触发终止条件]              │
└──────────────────────────────────────────────────────┘
   ↓
输出结果
```

### 关键设计细节

1. **流式工具执行 (`StreamingToolExecutor`)**：工具在 API 流式返回时就开始执行，不等完整响应结束
2. **工具并发智能编排**：只读工具并行执行，写操作串行执行
3. **自动压缩**：当 token 接近上下文窗口限制时，自动触发压缩
4. **恢复循环**：当 `max_output_tokens` 被截断时，最多自动恢复 3 次
5. **Task Budget**：支持 API 级别的任务预算控制

---

## 6. 工具系统（Harness 的核心）

### 6.1 工具注册与发现

`tools.ts` 是工具注册中心，根据 Feature Flag 和用户类型条件加载 30+ 种工具：

```typescript
// 核心工具（始终加载）
BashTool, FileReadTool, FileWriteTool, FileEditTool,
GlobTool, GrepTool, AgentTool

// 条件加载工具
WebSearchTool      // feature('WEB_SEARCH')
WebFetchTool       // feature('WEB_FETCH')
MCPTool            // 按连接的 MCP 服务器动态生成
TeamCreateTool     // feature('BG_SESSIONS')
ToolSearchTool     // 工具数量 > 阈值时加载
```

### 6.2 工具执行编排 (`toolOrchestration.ts`)

```
Tool Calls 到达
      ↓
partitionToolCalls() — 将工具调用分区：
  ├── [ReadOnly, ReadOnly, ReadOnly]  → runToolsConcurrently() (最大并发 10)
  ├── [WriteOp]                       → runToolsSerially()
  ├── [ReadOnly, ReadOnly]            → runToolsConcurrently()
  └── [WriteOp]                       → runToolsSerially()
```

**关键机制**：每个工具定义一个 `isConcurrencySafe(input)` 方法，根据具体输入判断是否可以并发。例如，读取不同文件的 `FileReadTool` 可以并发，但 `BashTool` 执行写操作时必须串行。

### 6.3 核心工具分析

#### BashTool (1144+ 行)
- 执行 Bash 命令，支持超时控制
- 多层安全验证：路径验证、sed 解析、沙箱判断、破坏性命令检测
- 只读验证、权限管理

#### AgentTool — 子代理系统 (1398+ 行)
这是整个系统中**最复杂也最关键的工具**：

```
AgentTool 调用
      ↓
┌─────────────────────────────┐
│ 1. 解析参数                  │
│    - subagent_type (代理类型) │
│    - prompt (任务描述)        │
│    - isolation (隔离级别)     │
│    - run_in_background       │
│    - model (可选模型指定)     │
│                              │
│ 2. 创建子代理                │
│    - 前台同步 / 后台异步     │
│    - worktree 隔离           │
│    - remote 远程环境         │
│    - fork (继承父上下文)     │
│                              │
│ 3. 子代理运行                │
│    - 拥有独立的 query loop   │
│    - 受限的工具集            │
│    - 独立的 token 预算       │
│                              │
│ 4. 返回结果给父代理          │
└─────────────────────────────┘
```

**Fork 子代理**：继承父代理的完整对话上下文，共享 prompt cache，成本极低。这是一个革命性的设计——子代理不从零开始，而是"站在巨人肩膀上"。

#### ToolSearchTool (472 行)
当工具数量超过阈值时激活的**元工具**：
- 不将所有工具 Schema 都发送给模型
- 模型先调用 ToolSearchTool 描述需求
- 系统根据语义匹配返回相关工具
- 模型再调用匹配到的具体工具

这极大减少了 token 消耗，也是"延迟加载"思想在工具系统中的体现。

#### TeamCreateTool + SendMessageTool
多代理协作的基础设施，详见第 7 节。

### 6.4 工具类型定义 (`Tool.ts`)

每个工具包含：
- `name`: 工具名
- `description`: 给模型看的描述
- `inputSchema`: Zod Schema 验证
- `call()`: 执行逻辑（返回 AsyncGenerator）
- `isConcurrencySafe(input)`: 并发安全判断
- `isReadOnly()`: 是否只读
- `needsPermissions()`: 是否需要权限
- `userFacingName()`: 用户界面显示名
- `renderToolUseMessage()`: UI 渲染

---

## 7. 多代理协作系统

### 7.1 Coordinator 模式 (`coordinator/coordinatorMode.ts`)

当 `CLAUDE_CODE_COORDINATOR_MODE=1` 时，Claude Code 进入**协调器模式**：

```
┌──────────────────────────────────────────────────┐
│               Coordinator (主代理)                │
│                                                   │
│  职责: 理解用户意图 → 拆解任务 → 分配给 Worker    │
│  工具: Agent, SendMessage, TaskStop               │
│                                                   │
│  ┌───────────────────────────────────────────┐   │
│  │              Workers (子代理)               │   │
│  │                                            │   │
│  │  Worker A: 研究代码结构 ←──┐               │   │
│  │  Worker B: 分析测试覆盖 ←──┼── 并行执行    │   │
│  │  Worker C: 检查依赖关系 ←──┘               │   │
│  │                                            │   │
│  │  → Coordinator 综合分析 →                  │   │
│  │                                            │   │
│  │  Worker D: 实现修复 (串行)                 │   │
│  │  Worker E: 验证修复 (独立)                 │   │
│  └───────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

**核心工作流**：
1. **Research 阶段**：并行启动多个 Worker 进行调研
2. **Synthesis 阶段**：Coordinator **自己综合分析**研究结果（不委托给 Worker）
3. **Implementation 阶段**：基于综合分析，指派 Worker 执行
4. **Verification 阶段**：独立 Worker 验证（新鲜视角）

### 7.2 Team 系统

```typescript
// TeamCreateTool: 创建团队
TeamCreate({ team_name: "feature-auth", description: "Auth feature team" })

// 启动团队成员 (通过 AgentTool)
Agent({ name: "researcher", team_name: "feature-auth", prompt: "..." })
Agent({ name: "implementer", team_name: "feature-auth", prompt: "..." })

// 成员间通信
SendMessage({ to: "researcher", message: "请调查 session 过期逻辑" })
```

### 7.3 消息路由机制 (`SendMessageTool`)

918 行的消息路由系统支持：
- **直接消息**：发送给特定代理
- **广播**：发送给所有团队成员
- **关闭请求/响应**：优雅关闭
- **计划审批**：工作计划的审批流程

---

## 8. 上下文管理与压缩系统

这是 Claude Code 最精妙的系统之一，解决了"无限对话"的核心挑战。

### 8.1 多层压缩策略

```
┌──────────────────────────────────────────────┐
│            Context Management                 │
│                                               │
│  Layer 1: Auto Compact (自动压缩)            │
│  - 监控 token 使用量                          │
│  - 达到阈值自动触发                           │
│  - 保留关键上下文，压缩历史对话               │
│                                               │
│  Layer 2: Micro Compact (微压缩)             │
│  - 针对单次工具调用结果的压缩                 │
│  - 缓存压缩结果避免重复计算                   │
│                                               │
│  Layer 3: Reactive Compact (响应式压缩)      │
│  - 当 prompt_too_long 错误触发时              │
│  - 紧急压缩以恢复对话                        │
│                                               │
│  Layer 4: Snip Compact (裁剪压缩)            │
│  - 对历史消息进行智能裁剪                     │
│                                               │
│  Layer 5: Session Memory (会话记忆)          │
│  - 跨压缩边界保持关键信息                     │
│  - 持久化到磁盘                              │
│                                               │
│  Layer 6: Context Collapse (上下文折叠)      │
│  - Feature-gated 实验性特性                   │
│  - 更激进的上下文压缩策略                     │
└──────────────────────────────────────────────┘
```

### 8.2 自动压缩触发机制 (`autoCompact.ts`)

```typescript
// Token 警告状态
type TokenWarningState = {
  shouldAutoCompact: boolean        // 是否应该触发压缩
  tokenWarningLevel: 'normal' | 'warning' | 'critical'
  contextTokensUsed: number
  contextTokensLimit: number
}
```

当 token 使用接近窗口限制时：
1. `warning` 级别 → 提醒用户
2. `critical` 级别 → 自动触发压缩
3. 压缩后，将历史消息替换为简洁摘要
4. **Session Memory 跨压缩边界保持关键上下文**

### 8.3 System Prompt 缓存优化

```typescript
// prompts.ts 中的关键设计
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

System Prompt 被分为两部分：
- **静态部分**（boundary 之前）：可全局缓存，跨会话复用
- **动态部分**（boundary 之后）：包含用户/会话特定内容

这种分割设计使得 **Prompt Cache 命中率最大化**，大幅降低了 API 成本。

---

## 9. 权限与安全系统

### 9.1 多层权限模型

```
┌─────────────────────────────────────────────┐
│           Permission Layers                  │
│                                              │
│  Layer 1: Permission Mode (权限模式)         │
│  ├── default: 每次询问用户                   │
│  ├── auto: 自动分类器判断                    │
│  └── bypass: 跳过所有权限检查                │
│                                              │
│  Layer 2: Tool Permission (工具权限)         │
│  ├── allow: 始终允许                         │
│  ├── deny: 始终拒绝                          │
│  └── ask: 每次询问                           │
│                                              │
│  Layer 3: Hooks (钩子)                       │
│  ├── pre-tool hooks: 工具执行前              │
│  ├── post-tool hooks: 工具执行后             │
│  └── user-prompt-submit hooks: 提交前        │
│                                              │
│  Layer 4: Sandbox (沙箱)                     │
│  ├── 网络白名单                              │
│  ├── 文件系统读写权限                        │
│  └── Unix socket 限制                        │
│                                              │
│  Layer 5: Auto Classifier (自动分类器)       │
│  └── 判断工具调用的风险级别                   │
└─────────────────────────────────────────────┘
```

### 9.2 BashTool 安全验证

BashTool 是安全风险最高的工具，有极其严格的多层验证：

1. **路径验证**：确保命令不操作危险路径
2. **sed 解析**：解析 sed 命令判断是否修改文件
3. **破坏性命令检测**：识别 `rm -rf`、`git push --force` 等
4. **只读验证**：在只读模式下拦截写操作
5. **沙箱判断**：在沙箱环境中限制命令执行

---

## 10. 为什么比 LangChain 等显式 Agent 框架更强

### 10.1 根本范式差异

| 维度 | LangChain/LangGraph | Claude Code |
|------|---------------------|-------------|
| **控制流** | 开发者预定义工作流图(DAG) | LLM 自主决策，harness 提供环境 |
| **工具调用** | 显式 chain/pipe 串联 | 模型在 agent loop 中自由选择 |
| **状态管理** | 开发者手动管理 state | 系统自动管理（压缩/恢复/持久化）|
| **错误恢复** | 需要预定义 fallback 路径 | 自动重试/恢复/降级 |
| **并发** | 需要显式定义并行节点 | 自动根据工具特性智能并发 |
| **扩展** | 需要修改 chain 定义 | 添加工具/MCP 服务器即可 |

### 10.2 具体优势分析

#### ① 隐式 vs 显式控制流

**LangChain 的问题**：
```python
# 开发者必须预定义工作流
chain = (
    prompt_template 
    | llm 
    | output_parser 
    | tool_selector 
    | tool_executor
)
```
这要求开发者**预先知道**所有可能的执行路径，任何新场景都需要修改 chain 定义。

**Claude Code 的方式**：
```
query.ts 的 Agent Loop:
while (模型未停止):
    response = callClaudeAPI(messages)
    if response 包含 tool_use:
        results = executeTools(tool_calls)  // 模型决定调什么
        messages.append(results)
        continue  // 回到循环——模型自己决定下一步
    else:
        break  // 模型认为任务完成
```
模型在每一轮都可以**自由选择任意组合的工具**，不受预定义流程限制。

#### ② 流式工具执行

LangChain 的工具执行是"请求-响应"模式——等完整响应后再执行。

Claude Code 的 `StreamingToolExecutor` 在 **API 流式返回时就开始执行工具**：
- 第一个 `tool_use` block 完成 → 立刻开始执行
- 后续 block 继续流式接收 → 继续排队/并行执行
- 整体延迟大幅降低

#### ③ 智能并发编排

```typescript
// toolOrchestration.ts — 自动识别并发安全性
partitionToolCalls(tools):
  [ReadFile, GrepTool, GlobTool]  → 并行执行 (max concurrency: 10)
  [FileEditTool]                   → 串行执行 (写操作)
  [ReadFile, ReadFile]             → 再次并行
```

LangChain 需要开发者手动标记哪些步骤可以并行，Claude Code 通过每个工具的 `isConcurrencySafe()` 方法**自动判断**。

#### ④ 无限上下文

LangChain 受限于固定上下文窗口。Claude Code 通过多层压缩策略实现**理论上无限的对话长度**：

```
系统提示: "The conversation has unlimited context through automatic summarization."
```

这不是空话——背后是 5 层压缩策略 + Session Memory 的精密系统。

#### ⑤ 子代理系统的深度

LangChain 的 Agent 是扁平的——每个 Agent 是独立的 chain。

Claude Code 的 AgentTool 支持：
- **递归代理**：子代理可以再启动子代理
- **Fork 代理**：继承父上下文，共享 prompt cache
- **后台异步代理**：不阻塞主对话
- **Worktree 隔离**：在独立 Git worktree 中执行
- **Coordinator 模式**：主代理编排多个 Worker 的完整工作流
- **Team 协作**：多代理间通信和协调

#### ⑥ 环境感知的深度

Claude Code 不只是"调用工具"，它**深度感知开发环境**：

| 能力 | 实现 |
|------|------|
| Git 状态 | `context.ts` 注入分支、recent commits、status |
| 项目配置 | CLAUDE.md 项目级/用户级配置 |
| LSP 集成 | 实时获取诊断信息、代码智能 |
| MCP 协议 | 动态发现和连接外部工具 |
| IDE 桥接 | 与 VS Code/JetBrains 双向通信 |
| 语音输入 | 语音转文字直接输入 |

---

## 11. Harness 能力的全面体现

"Harness" 指的是为 LLM 构建的**运行时环境和工具套件**——它不控制 LLM 做什么，而是**赋予 LLM 做任何事的能力**。以下是 Claude Code 中 harness 能力的全面体现：

### 11.1 工具层 Harness

| Harness 能力 | 代码位置 | 说明 |
|-------------|---------|------|
| **文件系统完全访问** | `FileReadTool`, `FileWriteTool`, `FileEditTool` | 读/写/精确编辑任意文件 |
| **Shell 执行** | `BashTool` (1144行 + 100K安全验证) | 执行任意 Shell 命令 |
| **代码搜索** | `GlobTool`, `GrepTool` | 文件名模式匹配 + 内容正则搜索 |
| **网络访问** | `WebSearchTool`, `WebFetchTool` | 搜索互联网 + 抓取网页 |
| **子代理** | `AgentTool` (1398行) | 递归启动子代理处理复杂任务 |
| **外部工具** | `MCPTool` | 通过 MCP 协议接入任意外部工具 |
| **代码智能** | `LSPTool` | 调用语言服务器获取诊断、定义跳转 |
| **笔记本** | `NotebookEditTool` | 编辑 Jupyter 笔记本 |
| **元工具** | `ToolSearchTool` | 工具的工具——按需发现和加载 |

### 11.2 上下文层 Harness

| Harness 能力 | 代码位置 | 说明 |
|-------------|---------|------|
| **系统提示工程** | `constants/prompts.ts` (915行) | 精心构造的提示词，指导 LLM 行为 |
| **上下文注入** | `context.ts` | 自动注入 Git 状态、项目配置等 |
| **自动压缩** | `services/compact/` (5个文件) | 多层上下文压缩，突破窗口限制 |
| **Session Memory** | `services/SessionMemory/` | 跨压缩边界的持久记忆 |
| **Prompt Cache** | `claude.ts` 中的缓存分割 | 静态/动态提示词分离，最大化缓存命中 |
| **Token 预算** | `query/tokenBudget.ts` | 精确的 token 使用追踪和预算控制 |

### 11.3 执行层 Harness

| Harness 能力 | 代码位置 | 说明 |
|-------------|---------|------|
| **流式执行** | `StreamingToolExecutor` (531行) | 流式接收 + 实时执行 |
| **并发编排** | `toolOrchestration.ts` (189行) | 智能判断并行/串行 |
| **错误恢复** | `query.ts` 中的恢复循环 | max_output_tokens 自动恢复、prompt_too_long 自动压缩 |
| **中断处理** | `AbortController` 体系 | 优雅的中断和资源释放 |
| **Cost Tracking** | `cost-tracker.ts` | 实时追踪 API 消耗 |

### 11.4 安全层 Harness

| Harness 能力 | 代码位置 | 说明 |
|-------------|---------|------|
| **权限模型** | `hooks/useCanUseTool.tsx`, `types/permissions.ts` | 多层权限控制 |
| **沙箱** | `entrypoints/sandboxTypes.ts` | 网络/文件系统隔离 |
| **破坏性检测** | `BashTool/` 安全模块 | 识别并拦截危险命令 |
| **Hooks 系统** | `utils/hooks/` | 工具执行前后的自定义校验 |

### 11.5 扩展层 Harness

| Harness 能力 | 代码位置 | 说明 |
|-------------|---------|------|
| **MCP 协议** | `services/mcp/` | 作为 Server 暴露能力 + 作为 Client 接入外部 |
| **插件系统** | `plugins/`, `services/plugins/` | 自定义插件加载 |
| **技能系统** | `skills/` | 领域特定知识和工作流 |
| **Feature Flag** | GrowthBook + `feature()` 编译时消除 | 灵活的功能门控 |
| **自定义代理** | `AgentTool/loadAgentsDir.ts` | 用户定义的代理类型 |

### 11.6 协作层 Harness

| Harness 能力 | 代码位置 | 说明 |
|-------------|---------|------|
| **Coordinator 模式** | `coordinator/coordinatorMode.ts` | 主代理编排多 Worker |
| **Team 系统** | `TeamCreateTool`, `TeamDeleteTool` | 创建和管理代理团队 |
| **消息路由** | `SendMessageTool` (918行) | 代理间通信 |
| **IDE 桥接** | `bridge/` (31个文件) | 与 IDE 扩展双向通信 |
| **Teleport** | `utils/teleport/` | 跨设备会话转移 |

---
## 12. Edge Case 处理大全 — 现实世界复杂度的处理

Claude Code 之所以"好用"，不只是因为模型能力强，更因为源码中藏着**数百个 edge case 处理**。以下按类别详细展示，每个 edge case 都附带**源码位置、具体处理方式和触发场景**。如此被广泛使用的产品，面对到各种各样的奇怪问题也少不了对于边界条件的处理，而claude code也并非用了什么外界认为的高雅方式，而是面向体验为主，对于常见的边界条件做了一个个的特殊处理。

---

### 12.1 会话恢复与中断检测（conversationRecovery.ts）

这是最复杂的 edge case 处理之一。用户随时可能断开、Ctrl+C、崩溃，恢复时必须让对话"看起来没断过"。

#### ① 遗留附件类型迁移
```
文件: utils/conversationRecovery.ts · migrateLegacyAttachmentTypes()
```
- **场景**: 老版本的 session 使用了 `new_file`/`new_directory` 类型的附件，新版本不认识
- **处理**: 透明地将 `new_file` → `file`、`new_directory` → `directory`，并补填 `displayPath`
- **没有这个**: 旧 session 恢复时附件全部丢失

#### ② 无效 permissionMode 清理
```
文件: utils/conversationRecovery.ts · deserializeMessagesWithInterruptDetection() 
```
- **场景**: 从磁盘反序列化的消息可能包含当前 build 不支持的 `permissionMode` 值（比如从内部版本降级到外部版本）
- **处理**: 遍历所有 user 消息，对不在 `PERMISSION_MODES` 集合中的 mode 设为 `undefined`
- **没有这个**: API 调用带着无效 mode → 400 错误，用户无法继续对话

#### ③ 未解决的 tool_use 过滤
```
文件: utils/conversationRecovery.ts · filterUnresolvedToolUses()
```
- **场景**: 用户在模型调用工具的过程中 Ctrl+C，此时会留下只有 `tool_use` 但没有对应 `tool_result` 的消息
- **处理**: 过滤掉所有没有匹配 `tool_result` 的 `tool_use` 块及其后续合成消息
- **没有这个**: API 报错 "tool_use ids were found without tool_result blocks"，对话彻底卡死

#### ④ 孤儿 thinking-only 消息过滤
```
文件: utils/conversationRecovery.ts · filterOrphanedThinkingOnlyMessages()
```
- **场景**: 流式传输时，每个 content block 会产生独立的 AssistantMessage。如果中间插入了 user 消息导致无法按 `message.id` 合并，就会留下只含 `thinking` 的 assistant 消息
- **处理**: 移除这些只有 thinking block 的孤儿 assistant 消息
- **没有这个**: 连续的 assistant 消息如果 thinking block 签名不匹配 → API 400 错误

#### ⑤ 纯空白 assistant 消息过滤
```
文件: utils/conversationRecovery.ts · filterWhitespaceOnlyAssistantMessages()
```
- **场景**: 模型先输出 `"\n\n"` 然后开始 thinking，用户此时取消 → 留下只有空白文本的 assistant 消息
- **处理**: 过滤掉这些消息
- **顺序依赖**: 必须先 strip trailing thinking，再 filter whitespace。反过来会导致 `[text("\n\n"), thinking("...")]` 幸存后被 strip 成 `[text("\n\n")]`，API 同样拒绝

#### ⑥ 中断类型的三态检测
```
文件: utils/conversationRecovery.ts · detectTurnInterruption()
```
- **三种状态**: `none`（正常完成）| `interrupted_prompt`（用户输入了但模型没回复）| `interrupted_turn`（模型回复到一半被打断）
- **特殊处理**:
  - 系统消息和进度消息被跳过（它们是簿记制品，不应掩盖真正的中断）
  - API 错误消息也被跳过（否则重试耗尽后的错误会被误判为"完成的轮次"）
  - Brief 模式下工具调用正常结束在 `tool_result`（不是中断！）

#### ⑦ Brief 模式的特殊终止检测
```
文件: utils/conversationRecovery.ts · isTerminalToolResult()
```
- **场景**: Brief 模式（#20467）会省略尾部的 assistant 文本块，所以合法的完成轮次会以 `SendUserMessage` 的 `tool_result` 结尾
- **没有这个**: 每次恢复 brief 模式的 session 都会被误分类为"中断"，注入一个多余的 "Continue from where you left off."

#### ⑧ 技能状态跨 compaction 保留
```
文件: utils/conversationRecovery.ts · restoreSkillStateFromMessages()
```
- **场景**: session 经过 compaction（上下文压缩）后再 resume，如果不恢复技能状态，下一次 compaction 会因为 `STATE.invokedSkills` 为空而丢失技能信息
- **处理**: 从消息中的 `invoked_skills` 附件重新注册技能，并抑制重复的技能列表公告（节省 ~600 tokens）

---

### 12.2 API 重试与错误恢复（withRetry.ts）

这个文件是**重试逻辑的百科全书**，处理了 20+ 种错误场景。

#### ① 529 过载的分级处理
```
文件: services/api/withRetry.ts
```
- **前台任务**: 用户正在等待的查询（repl_main_thread、agent 等）→ 最多重试 3 次，然后降级到 fallback 模型
- **后台任务**: 摘要、标题生成、分类器等 → **直接放弃不重试**，因为每次重试会造成 3-10 倍的网关放大效应
- **无人值守模式**: `CLAUDE_CODE_UNATTENDED_RETRY` 环境变量 → 429/529 无限重试，每 30 秒发送心跳防止宿主标记 session 为空闲

#### ② Fast Mode 的精细降级
```
文件: services/api/withRetry.ts · fast mode fallback 逻辑
```
- **短延迟 429**（< 20s）: 等待后保持 fast mode 重试（保留 prompt cache）
- **长延迟 429/529**: 进入冷却（最少 10 分钟），切换到标准速度模型
- **Overage 被禁**: API 返回 `overage-disabled-reason` 头 → 永久关闭 fast mode
- **Fast mode 未开通**: API 返回 "Fast mode is not enabled" → 永久关闭并重试

#### ③ 连接状态修复 (ECONNRESET/EPIPE)
```
文件: services/api/withRetry.ts · isStaleConnectionError()
```
- **场景**: HTTP keep-alive 连接池中的连接已被服务端关闭，但客户端不知道
- **处理**: 检测到 ECONNRESET/EPIPE → 调用 `disableKeepAlive()` 关闭连接池 → 用新连接重试
- **没有这个**: 每个请求都先失败一次才能成功

#### ④ OAuth Token 过期/撤销的透明刷新
```
文件: services/api/withRetry.ts
```
- **401 Token 过期**: 记录失败的 access token → 调用 `handleOAuth401Error` 刷新 → 获取新 client
- **403 Token 撤销**（另一个进程刷新了 token）: 同样触发刷新
- **Bedrock AWS 凭证过期**: 清除凭证缓存，让 SDK 重新获取
- **Vertex GCP 凭证失败**: 检测 google-auth-library 的各种错误消息（"Could not load default credentials" / "invalid_grant"）
- **CCR 远程模式**: 401/403 不是凭证问题（JWT 由基础设施提供），是暂时性网络抖动 → 无视 `x-should-retry: false` 直接重试

#### ⑤ Context Window 溢出的自动调整
```
文件: services/api/withRetry.ts · parseMaxTokensContextOverflowError()
```
- **场景**: `input_tokens + max_tokens > context_limit`
- **处理**: 解析错误消息中的具体数字 → 计算可用空间 → 自动调低 `max_tokens`（但不低于 3000）→ 考虑 thinking budget 的最小需求 → 重试
- **没有这个**: 需要用户手动调参数

#### ⑥ SDK 529 状态码丢失的兜底
```
文件: services/api/withRetry.ts · is529Error()
```
- **场景**: SDK 在流式传输时有时不能正确传递 529 状态码
- **处理**: 除了检查 `error.status === 529`，还检查错误消息中是否包含 `"type":"overloaded_error"`
- **没有这个**: 529 错误不会被正确重试

---

### 12.3 消息/对话完整性修复（errors.ts + messages.ts）

#### ① tool_use/tool_result 配对修复
```
文件: services/api/claude.ts L1298-1301 · ensureToolResultPairing()
```
- **场景**: resume/teleport session 时，tool_use 和 tool_result 可能不匹配
- **处理**: 
  - 为孤儿 `tool_use` 插入合成的错误 `tool_result`（内容: "[Tool result missing due to internal error]"）
  - 剥离引用不存在 `tool_use` 的孤儿 `tool_result`
- **安全措施**: 合成的 placeholder 被 HFI（人类反馈）系统识别并拒绝提交，避免污染训练数据

#### ② 重复 tool_use ID 处理
```
文件: services/api/errors.ts L719-733
```
- **场景**: CC-1212 bug，`ensureToolResultPairing` 本应清除重复，但新的腐蚀路径可能绕过
- **处理**: 记录日志用于根因分析 + 提示用户 `/rewind` 恢复
- **没有这个**: 对话进入死循环（每次重试都累积更多重复）

#### ③ 孤儿权限响应处理
```
文件: cli/structuredIO.ts + cli/print.ts · handleOrphanedPermissionResponse()
```
- **场景**: SDK host（VS Code）发送的权限响应到达时，原始请求已经被处理或中止了
- **处理**: 
  - 维护 `resolvedToolUseIds` 集合，重复的 control_response 直接忽略
  - WebSocket 重连后可能发送重复消息 → 用 `handledOrphanedToolUseIds` 去重
- **没有这个**: 同一个工具被执行两次 → 重复 tool_use ID → API 400 → 对话永久损坏

---

### 12.4 上下文压缩的防御性处理（autoCompact.ts）

#### ① 自动压缩的熔断器
```
文件: services/compact/autoCompact.ts · MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```
- **场景**: 上下文已经不可恢复地超过限制，autocompact 注定会失败
- **数据驱动**: BQ 数据显示 1,279 个 session 有 50+ 次连续失败（最高 3,272 次），全局浪费 ~250K API 调用/天
- **处理**: 连续失败 3 次后停止尝试

#### ② 递归防护
```
文件: services/compact/autoCompact.ts · shouldAutoCompact()
```
- **场景**: `session_memory` 和 `compact` 本身就是 forked agent，如果它们自己触发 autocompact 会死锁
- **处理**: 对这些 querySource 直接返回 false
- **Context Collapse 场景**: marble_origami（ctx-agent）如果触发 autocompact → `resetContextCollapse()` 会销毁**主线程**的 committed log（因为是模块级共享状态）

#### ③ 压缩自身超过 prompt-too-long 的处理
```
文件: services/compact/compact.ts · CC-1180
```
- **场景**: 对话太长，压缩请求本身就触发了 "prompt is too long" → 用户被彻底卡住
- **处理**: 从最老的 API-round 组开始截断 → 利用 `getPromptTooLongTokenGap()` 解析错误消息中的 token 差值 → 一次跳过多个组而不是逐个剥离 → 重试
- **兜底**: 截断后仍然太长 → 抛出明确错误而不是无限循环

#### ④ 图片剥离
```
文件: services/compact/compact.ts · stripImagesFromMessages()
```
- **场景**: CCD（Chrome DevTools）session 用户频繁附加图片，压缩 API 调用把图片也发过去就超限了
- **处理**: 压缩前把所有 user 消息中的 image block 替换为 "[image was shared]" 文本标记

---

### 12.5 30+ 种 API 错误的精确分类（errors.ts）

`getAssistantMessageFromError()` 函数长达 **600+ 行**，精确分类了每种 API 错误并给出**针对性的用户提示**：

| 错误场景 | 检测方式 | 用户提示 |
|---------|---------|---------|
| SDK 超时 | `APIConnectionTimeoutError` | "Request timed out" |
| 图片太大 | `ImageSizeError` / `ImageResizeError` | CLI: "esc esc to go back"; SDK: generic |
| 429 Rate Limit + 新头部 | `anthropic-ratelimit-unified-*` 头 | 根据 5h/7d/opus 类型和 overage 状态定制 |
| 429 无头部 + Long Context | 消息包含 "Extra usage is required" | "run /extra-usage or /model" |
| Prompt Too Long | 大小写不敏感匹配（Vertex 首字母大写） | 泛化内容 + errorDetails 存原始消息 |
| PDF 超页数 | `/maximum of \d+ PDF pages/` | 非交互: "try pdftotext"; CLI: "esc esc" |
| PDF 密码保护 | 消息包含特定字符串 | 区分交互/非交互模式 |
| PDF 无效 | "The PDF specified was not valid" | 避免无效 block 污染后续所有调用 |
| 图片超 5MB | 400 + "image exceeds" + "maximum" | 大小超限提示 |
| 多图分辨率超限 | 400 + "image dimensions exceed" + "many-image" | "Run /compact" or "start new session" |
| 413 请求体太大 | HTTP 413 | 大 PDF + 上下文超 32MB 限制 |
| tool_use/tool_result 不匹配 | 400 + 特定消息 | 内部: "/share + /rewind"; 外部: "/rewind" |
| 重复 tool_use ID | 400 + "ids must be unique" | "/rewind to recover" |
| 无效模型名 + Pro 用户 | 400 + "invalid model name" + Opus | "Opus not available on Pro plan, /logout + /login" |
| 无效模型名 + 内部用户 | 400 + "invalid model name" + 非自定义 | "Your org isn't gated, try ANTHROPIC_MODEL=..." |
| 余额不足 | "credit balance is too low" | 专门的计费错误提示 |
| Organization 被禁用 | 400 + "organization has been disabled" | 区分环境变量 vs OAuth 来源 |
| API Key 无效 | 消息包含 "x-api-key" | CCR: "temporary issue, try again"; 外部: "Fix API key"; 其他: "/login" |
| OAuth Token 被撤销 | 403 + "OAuth token has been revoked" | "/login" |
| OAuth 组织不允许 | 401/403 + 特定消息 | "/login" |
| Bedrock 模型无权限 | 消息包含 "model id" | 建议 fallback 模型链 |
| 404 模型不存在 | HTTP 404 | 3P 用户建议降级链 (opus-4-6→opus-4-1→...) |
| 连接错误 | `APIConnectionError` | 包含 SSL 证书检测 |
| AFK Mode 不可用 | 400 + beta header 被拒 | "Auto mode unavailable for your plan" |
| Refusal | stop_reason === 'refusal' | 建议切换模型 |

---

### 12.6 Git 操作的防御性处理（git.ts）

```
文件: utils/git.ts · preserveGitStateForIssue()
```
三种退化模式，每种都有独立的恢复路径：

1. **Detached HEAD**: 无法获取分支名 → 直接用 merge-base 与默认分支
2. **No Remote**: 没有远端仓库 → remote 字段返回 null，只用 HEAD-only 模式
3. **Shallow Clone**: 浅克隆没有完整历史 → 检测后 fallback 到 HEAD-only 模式
4. **Merge-base 失败**: 即使上面都过了，merge-base 可能还是失败 → 再次 fallback

---

### 12.7 第三方 SDK/API 的 Workaround

#### ① GrowthBook SDK 的 Remote Eval Bug
```
文件: services/analytics/growthbook.ts
```
**三层 workaround**：
1. API 返回 `{ "value": ... }` 但 SDK 期望 `{ "defaultValue": ... }` → 手动转换
2. SDK 的 `evalFeature()` 在 remoteEval 模式下忽略预计算的 value → 自己缓存评估结果
3. SDK 的 `setForcedFeatures` 在 remoteEval 模式也不可靠 → 在 `getFeatureValueInternal` 中优先查自己的缓存

#### ② Ink 不支持多个 Static 组件
```
文件: utils/staticRender.tsx
```
- **场景**: Ink 框架不允许在同一渲染树中使用多个 `<Static>` 组件
- **处理**: 将组件渲染为字符串后直接打印到 stdout，绕过 Ink 的限制

#### ③ Bun 的 FSWatcher 死锁
```
文件: utils/skills/skillChangeDetector.ts
```
- **场景**: chokidar 4+ 放弃了 fsevents，Bun 的 `fs.watch` fallback 使用 kqueue，每个监视目录需要一个 fd。git 操作触碰多个目录时会导致死锁
- **处理**: 在 Bun 下使用 `stat()` 轮询代替 FSWatcher
- **注释**: "The fix is pending upstream; remove this once the Bun PR lands"

#### ④ Axios 代理兼容性
```
文件: utils/proxy.ts
```
- 绕过 axios/axios#4531 bug：设置 `axios.defaults.proxy = false`，然后手动创建 proxy agent

---

### 12.8 WebSocket/网络层的鲁棒性

#### ① 重连时防止事件处理器泄漏
```
文件: remote/SessionsWebSocket.ts + cli/transports/WebSocketTransport.ts
```
- **场景**: 网络不稳定时频繁重连，每次重连都注册新的事件监听器，旧的 WebSocket + 闭包被孤儿化
- **处理**: 重连前显式清除旧 WS 的所有事件处理器，防止内存泄漏
- **注释**: "Without removal, each reconnect orphans the old WS object + its 5 closures until GC"

#### ② 文件监视器的 fd 限制
```
文件: services/teamMemorySync/watcher.ts
```
- **场景**: chokidar 4+ 放弃 fsevents 后，Bun 用 kqueue 实现 `fs.watch`，每个文件需要一个 fd。500+ team memory 文件 = 500+ 永久占用的 fd
- **处理**: 使用 `fs.watch({recursive: true})` 而不是 chokidar
- **验证**: "confirmed via lsof + repro"

#### ③ Keep-alive 心跳防空闲超时
```
文件: cli/remoteIO.ts
```
- Bridge 拓扑下 Envoy idle timeout 问题（#21931）→ 定期发送 `keep_alive` frame
- Voice stream WebSocket：连接后立即发一个 KeepAlive（音频硬件初始化可能超过 1s，否则服务端会关连接）

---

### 12.9 数据一致性与并发控制

#### ① Sandbox 权限更新的竞态
```
文件: screens/REPL.tsx L4636-4638
```
```javascript
// Immediately update sandbox in-memory config to prevent race conditions
// where pending requests slip through before settings change is detected
SandboxManager.refreshConfig();
```
- 修改权限后立即刷新内存配置，不等磁盘写入完成

#### ② 消息对象的直接属性变更（避免引用断裂）
```
文件: services/api/claude.ts L2236
```
```javascript
// IMPORTANT: Use direct property mutation, not object replacement.
// The transcript write queue holds a reference to message.message
// and serializes it lazily (100ms flush interval). Object
// replacement would cause the queue to serialize stale data.
```
- 转录写入队列持有消息引用 → 如果用新对象替换会导致序列化过时数据

#### ③ 配置写入的双重失败处理
```
文件: services/api/metricsOptOut.ts
```
- `saveGlobalConfig()` 的 fallback 路径可能抛异常（文件锁 + fallback 写入都失败）→ 额外 catch 防止 unhandled rejection

---

### 12.10 总结：Edge Case 处理的设计哲学

Claude Code 的 edge case 处理展现了几个核心设计原则：

1. **永不卡死**: 无论发生什么错误，都有退化路径。从 fallback 模型到 HEAD-only git mode 到截断式压缩，用户永远能继续工作。

2. **透明降级**: 错误消息极其精确，不是笼统的 "Something went wrong"，而是 "Your org isn't gated into the model, either use ANTHROPIC_MODEL=... or reach out in #channel"。

3. **数据驱动**: 很多阈值和策略来自真实数据（"BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures"），不是拍脑袋的猜测。

4. **防腐蚀设计**: 对话历史是最核心的状态，任何可能腐蚀它的操作都有多重防护（tool_use pairing、duplicate ID检测、orphan过滤）。

5. **注释即文档**: 几乎每个 workaround 都有 issue 编号（CC-1180、CC-1212、#20467、#21931、#34044）和详细的"为什么"注释，而不仅仅是"怎么做"。


---

## 12. 总结

### Claude Code 的核心竞争力

1. **"操作系统"而非"框架"**：它不是让开发者"构建 Agent"的框架，而是为 LLM 构建的完整操作环境。LLM 不需要被"编排"，它需要的是**丰富的环境和工具**。

2. **隐式控制流的力量**：没有预定义的 DAG、chain、pipe。Agent Loop 极其简洁——调用 API、执行工具、循环。所有的"智能"来自 LLM 本身的推理能力 + 精心设计的系统提示 + 丰富的工具反馈。

3. **工程深度远超想象**：
   - `BashTool` 仅安全验证就超过 100KB
   - `AgentTool` 支持递归、fork、后台、worktree 隔离
   - 上下文压缩有 5 个层次
   - 系统提示缓存做到了静态/动态分割

4. **Prompt Cache 意识贯穿始终**：从系统提示的 `DYNAMIC_BOUNDARY` 分割，到代理列表从 tool description 移到 attachment（减少 10.2% 的 cache_creation tokens），到 Fork 子代理共享父缓存——成本优化无处不在。

5. **真正的多代理系统**：不是"概念验证"级别的多代理，而是包含 Coordinator、Worker、Team、消息路由、任务停止、计划审批的**生产级多代理协作系统**。

6. **面对现实世界复杂度的边界条件处理**：数不清用户体验下所触发到的实际问题的edge case处理。

### 一句话总结

> **Claude Code 证明了：最好的 Agent 框架就是没有框架。** 与其用复杂的 DAG/Chain 来"编排" LLM，不如构建一个极其丰富的 harness（工具、上下文、安全、协作），然后**让 LLM 自己决定怎么做**。这种"隐式 Agent"范式比 LangChain 等"显式 Agent"框架更加灵活、强大，也更能发挥 LLM 的真正能力。
