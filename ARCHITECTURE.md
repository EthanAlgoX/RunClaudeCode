# Claude Code 源码架构分析

> 本文档对 `claude-code/` 目录下的 Claude Code 源码进行全面的架构分析，涵盖系统设计、核心模块、数据流与关键设计模式。

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [目录结构](#3-目录结构)
4. [启动流程](#4-启动流程)
5. [核心引擎：QueryEngine](#5-核心引擎queryengine)
6. [工具系统（Tool System）](#6-工具系统tool-system)
7. [命令系统（Command System）](#7-命令系统command-system)
8. [任务系统（Task System）](#8-任务系统task-system)
9. [状态管理](#9-状态管理)
10. [UI 层（React + Ink）](#10-ui-层react--ink)
11. [权限系统](#11-权限系统)
12. [Bridge / IDE 集成](#12-bridge--ide-集成)
13. [MCP 协议集成](#13-mcp-协议集成)
14. [服务层（Services）](#14-服务层services)
15. [插件与技能系统](#15-插件与技能系统)
16. [构建系统与特性标志](#16-构建系统与特性标志)
17. [数据流全景](#17-数据流全景)
18. [关键设计模式总结](#18-关键设计模式总结)

---

## 1. 项目概述

Claude Code 是 Anthropic 官方发布的终端 AI 编程助手 CLI 工具，于 **2026 年 3 月 31 日** 通过 npm 包内泄露的 source map 文件意外公开源代码。

| 属性 | 值 |
|------|-----|
| 语言 | TypeScript（严格模式） |
| 运行时 | [Bun](https://bun.sh)（非 Node.js） |
| 终端 UI | React + [Ink](https://github.com/vadimdemedes/ink) |
| 代码规模 | ~1,900 文件 · 512,000+ 行 |
| Schema 验证 | [Zod v4](https://zod.dev) |
| CLI 框架 | [Commander.js](https://github.com/tj/commander.js) |
| AI API | [Anthropic SDK](https://docs.anthropic.com) |

**核心能力：**
- 与 Claude 进行自然语言对话，执行文件编辑、代码审查、Git 工作流等复杂任务
- 多智能体协调（Multi-Agent Orchestration）
- IDE 双向集成（VS Code / JetBrains）
- MCP（Model Context Protocol）服务器集成
- 插件与技能扩展系统

---

## 2. 整体架构

### 2.1 宏观架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Claude Code CLI                               │
│                                                                     │
│  ┌────────────┐   ┌─────────────────────────────────────────────┐   │
│  │  入口层    │   │              核心处理层                      │   │
│  │ main.tsx   │──►│  QueryEngine.ts  (LLM API 调用中枢)          │   │
│  │ cli.tsx    │   │  ├── Streaming API 调用                     │   │
│  │ init.ts    │   │  ├── Tool 调用循环                           │   │
│  └────────────┘   │  ├── 思维模式 (Thinking Mode)               │   │
│                   │  └── Token 计数 & 成本追踪                   │   │
│  ┌────────────┐   └─────────┬───────────────────────────────────┘   │
│  │  UI 层     │             │                                       │
│  │ React+Ink  │◄────────────┤                                       │
│  │ components │             │                                       │
│  │ screens    │   ┌─────────▼───────────────────────────────────┐   │
│  └────────────┘   │              执行层                          │   │
│                   │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │   │
│  ┌────────────┐   │  │ Tools    │  │ Commands │  │  Tasks   │   │   │
│  │  状态层    │   │  │ (~40个)  │  │ (~50个)  │  │ (~7类型) │   │   │
│  │ AppState   │   │  └──────────┘  └──────────┘  └──────────┘   │   │
│  │ 全局状态   │   └─────────────────────────────────────────────┘   │
│  └────────────┘                                                     │
│                   ┌─────────────────────────────────────────────┐   │
│  ┌────────────┐   │              服务层 (Services)               │   │
│  │  集成层    │   │  Anthropic API · MCP · OAuth · LSP           │   │
│  │ Bridge     │   │  Analytics · Plugins · Compact · Memory      │   │
│  │ MCP Server │   └─────────────────────────────────────────────┘   │
│  └────────────┘                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 设计哲学

Claude Code 的架构遵循以下核心设计原则：

1. **特性标志驱动的死代码消除** — 通过 Bun 的 `bun:bundle` 实现编译期 tree-shaking
2. **懒加载重模块** — OpenTelemetry（~400KB）、gRPC（~700KB）等仅在首次使用时按需加载
3. **并行初始化** — MDM 读取、Keychain 预取、GrowthBook 初始化在启动时并发执行
4. **模块化的工具/命令/任务系统** — 所有扩展点均基于 Schema 验证（Zod）
5. **React + Ink 终端 UI** — 将 React 的声明式 UI 带入终端，支持流式渲染

---

## 3. 目录结构

```
claude-code/
├── src/                        # 主源码目录
│   ├── main.tsx                # CLI 入口：Commander.js + React/Ink 渲染器
│   ├── QueryEngine.ts          # 核心 LLM API 引擎（~46K 行）
│   ├── Tool.ts                 # 工具类型定义（~29K 行）
│   ├── Task.ts                 # 任务类型定义
│   ├── commands.ts             # 命令注册表（~25K 行）
│   ├── tools.ts                # 工具注册表
│   ├── tasks.ts                # 任务注册表
│   ├── context.ts              # 系统/用户上下文收集
│   ├── cost-tracker.ts         # Token 成本追踪
│   │
│   ├── tools/                  # 工具实现（~40 个）
│   │   ├── BashTool/           # Shell 命令执行
│   │   ├── FileEditTool/       # 文件局部编辑
│   │   ├── FileReadTool/       # 文件读取（支持图片/PDF）
│   │   ├── FileWriteTool/      # 文件写入
│   │   ├── GlobTool/           # 文件模式匹配
│   │   ├── GrepTool/           # ripgrep 内容搜索
│   │   ├── AgentTool/          # 子智能体生成
│   │   ├── MCPTool/            # MCP 工具调用
│   │   ├── TaskCreateTool/     # 任务创建
│   │   └── ...                 # 其他工具
│   │
│   ├── commands/               # 斜杠命令实现（~50 个）
│   │   ├── commit.ts           # /commit
│   │   ├── review.ts           # /review
│   │   ├── config/             # /config
│   │   └── ...
│   │
│   ├── components/             # Ink UI 组件（~140 个）
│   ├── screens/                # 全屏 UI（REPL、Doctor、Resume）
│   ├── hooks/                  # React hooks（~80 个）
│   ├── state/                  # 全局状态管理
│   ├── services/               # 外部服务集成
│   ├── bridge/                 # IDE 双向通信层
│   ├── coordinator/            # 多智能体协调
│   ├── plugins/                # 插件系统
│   ├── skills/                 # 技能系统
│   ├── tasks/                  # 任务实现
│   ├── memdir/                 # 持久化记忆（CLAUDE.md）
│   ├── entrypoints/            # 初始化逻辑
│   ├── types/                  # TypeScript 类型定义
│   ├── utils/                  # 工具函数
│   ├── schemas/                # Zod 配置 Schema
│   ├── voice/                  # 语音输入（特性标志：VOICE_MODE）
│   ├── vim/                    # Vim 模式键绑定
│   └── ...
│
├── web/                        # Next.js Web 前端
│   ├── app/                    # Next.js App Router
│   ├── components/             # React 组件
│   └── ...
│
├── mcp-server/                 # MCP Explorer 服务器（独立 npm 包）
│   └── src/                    # MCP 服务器实现
│
├── docs/                       # 英文文档
├── package.json                # 项目配置
├── tsconfig.json               # TypeScript 配置
└── biome.json                  # 代码格式/Lint 配置
```

---

## 4. 启动流程

### 4.1 启动时序图

```
main.tsx (Commander.js)
  │
  ├── [并发执行，启动优化]
  │   ├── startMdmRawRead()          // MDM 企业策略读取
  │   ├── ensureKeychainPrefetch()   // API Key / OAuth 预取
  │   └── GrowthBook 初始化          // 特性标志加载
  │
  ├── 解析 CLI 参数（--version, --print, --mcp 等）
  │
  └── 根据模式分发：
      ├── REPL 模式 → replLauncher.tsx → React/Ink 渲染
      ├── --print 模式 → 一次性查询，输出结果
      ├── --mcp 模式 → entrypoints/mcp.ts（MCP 服务器模式）
      └── SDK 模式 → entrypoints/sdk/（程序化 API）
```

### 4.2 入口点说明

| 文件 | 作用 |
|------|------|
| `src/main.tsx` | Commander.js CLI 解析器 + React/Ink 渲染器，并行化启动副作用 |
| `src/entrypoints/cli.tsx` | CLI 会话编排，从启动到 REPL 的主路径 |
| `src/entrypoints/init.ts` | 配置加载、遥测、OAuth、MDM 策略初始化 |
| `src/entrypoints/mcp.ts` | MCP 服务器模式入口（Claude Code 作为 MCP 服务器） |
| `src/entrypoints/sdk/` | Agent SDK（程序化嵌入 API） |
| `src/replLauncher.tsx` | 动态导入 App 组件并渲染 REPL |

### 4.3 并行启动优化

```typescript
// main.tsx - 在重模块导入前触发 I/O 副作用
profileCheckpoint('main_tsx_entry')
startMdmRawRead()              // MDM 子进程并发读取
ensureKeychainPrefetchCompleted()  // OAuth + API Key 预取
// 此后约 135ms 的模块评估与上述 I/O 并发执行
```

---

## 5. 核心引擎：QueryEngine

`src/QueryEngine.ts`（~46K 行）是整个系统的心脏，负责协调 LLM API 调用与工具执行循环。

### 5.1 核心职责

| 职责 | 描述 |
|------|------|
| **流式 API 调用** | 调用 Anthropic API，流式接收响应 |
| **工具调用循环** | LLM 请求工具时执行工具并将结果反馈给 LLM |
| **思维模式** | 管理 Extended Thinking 的 token 预算 |
| **重试逻辑** | 对瞬时错误自动退避重试 |
| **Token 计数** | 追踪每轮对话的输入/输出 token 和成本 |
| **上下文管理** | 管理对话历史和上下文窗口 |
| **权限执行** | 在工具执行前检查权限 |

### 5.2 QueryEngine 配置

```typescript
type QueryEngineConfig = {
  cwd: string                          // 当前工作目录
  tools: Tools                         // 可用工具列表
  commands: Command[]                  // 可用命令列表
  mcpClients: MCPServerConnection[]    // MCP 客户端连接
  agents: AgentDefinition[]            // 智能体定义
  canUseTool: CanUseToolFn             // 权限检查函数
  getAppState: () => AppState          // 状态读取
  setAppState: SetAppState             // 状态更新
  initialMessages?: Message[]          // 初始消息
  readFileCache: FileStateCache        // 文件状态缓存
  customSystemPrompt?: string          // 自定义系统提示
  appendSystemPrompt?: string          // 追加系统提示
  userSpecifiedModel?: string          // 用户指定模型
  fallbackModel?: string               // 备用模型
  thinkingConfig?: ThinkingConfig      // 思维模式配置
  maxTurns?: number                    // 最大对话轮数
  maxBudgetUsd?: number                // 最大预算（美元）
  jsonSchema?: Record<string, unknown> // JSON 输出 Schema
  verbose?: boolean                    // 详细日志
}
```

### 5.3 核心工具调用循环

```
用户输入
  │
  ▼
processUserInput()
  ├── 解析斜杠命令
  ├── 处理 @-mentions（文件/智能体）
  └── REPL 特殊语法检测
  │
  ▼
QueryEngine.query()
  ├── 构建系统提示（context.ts）
  │   ├── 工具描述
  │   ├── 上下文（git 状态、记忆）
  │   └── 命令注入的技能
  │
  ├── 调用 Claude API（流式）
  │
  └── 工具调用循环 ──────────────────────────┐
        │                                    │
        ├── 接收 tool_use 块                  │
        ├── 权限检查（canUseTool）            │
        ├── Zod Schema 验证输入               │
        ├── 执行工具                          │
        │   ├── BashTool → 子进程             │
        │   ├── FileEditTool → 文件系统       │
        │   ├── MCPTool → MCP 服务器          │
        │   └── AgentTool → 递归 QueryEngine  │
        ├── 收集结果                          │
        └── 将 tool_result 反馈给 LLM ───────┘
              （循环直至无工具调用）
```

---

## 6. 工具系统（Tool System）

### 6.1 工具定义模式

每个工具是一个自包含模块，通过 `buildTool()` 工厂函数定义：

```typescript
export const MyTool = buildTool({
  name: 'MyTool',
  aliases: ['my_tool'],
  description: '工具功能描述',

  // Zod Schema 验证输入
  inputSchema: z.object({
    param: z.string().describe('参数说明'),
    optional: z.number().optional(),
  }),

  // 核心执行逻辑
  async call(args, context, canUseTool, parentMessage, onProgress) {
    // context 包含：workingDir, appState, mcpClients 等
    return {
      data: { type: 'text', text: `结果: ${args.param}` },
      newMessages: [],  // 可选：注入额外消息
    }
  },

  // 权限检查
  async checkPermissions(input, context) {
    return { granted: true }
    // 或: return { granted: false, reason: '拒绝原因', prompt: '请求文本' }
  },

  // 并发安全性声明
  isConcurrencySafe(input) { return true },

  // 只读声明（不修改状态）
  isReadOnly(input) { return true },

  // 系统提示注入（此工具向 Claude 提供的上下文）
  prompt(options) { /* 返回系统提示内容 */ },

  // 工具调用 UI 渲染
  renderToolUseMessage(input, options) { /* 返回 JSX */ },

  // 工具结果 UI 渲染
  renderToolResultMessage(content, progressMessages, options) { /* 返回 JSX */ },
})
```

### 6.2 工具目录结构

```
src/tools/FileEditTool/
├── FileEditTool.tsx    # 主工具定义
├── UI.tsx              # 终端渲染组件
├── prompt.ts           # 系统提示注入
├── utils.ts            # 辅助函数
└── types.ts            # 类型定义
```

### 6.3 所有工具一览（~40 个）

#### 文件 I/O 类

| 工具 | 描述 |
|------|------|
| `FileReadTool` | 读取文件（支持图片、PDF、Jupyter Notebook） |
| `FileWriteTool` | 创建/覆盖文件 |
| `FileEditTool` | 局部修改（字符串替换） |
| `NotebookEditTool` | Jupyter Notebook 编辑 |

#### 搜索类

| 工具 | 描述 |
|------|------|
| `GlobTool` | 文件模式匹配 |
| `GrepTool` | 基于 ripgrep 的内容搜索 |
| `WebSearchTool` | Web 搜索 |
| `WebFetchTool` | 抓取 URL 内容 |

#### 执行类

| 工具 | 描述 |
|------|------|
| `BashTool` | Shell 命令执行 |
| `PowerShellTool` | PowerShell 命令执行 |
| `REPLTool` | 交互式 REPL 会话 |
| `SkillTool` | 技能执行 |
| `MCPTool` | MCP 服务器工具调用 |
| `LSPTool` | Language Server Protocol 集成 |

#### 智能体 & 团队类

| 工具 | 描述 |
|------|------|
| `AgentTool` | 生成子智能体（递归 QueryEngine） |
| `SendMessageTool` | 智能体间消息传递 |
| `TeamCreateTool` / `TeamDeleteTool` | 团队管理 |
| `AskUserQuestionTool` | 向用户提问 |

#### 任务管理类

| 工具 | 描述 |
|------|------|
| `TaskCreateTool` | 创建任务 |
| `TaskGetTool` | 获取任务信息 |
| `TaskUpdateTool` | 更新任务状态 |
| `TaskListTool` | 列出任务 |
| `TaskStopTool` | 停止任务 |
| `TaskOutputTool` | 读取任务输出 |

#### 模式 & 状态类

| 工具 | 描述 |
|------|------|
| `EnterPlanModeTool` / `ExitPlanModeTool` | 计划模式切换 |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git Worktree 隔离 |
| `ToolSearchTool` | 延迟工具发现 |
| `SleepTool` | 主动模式等待（特性：PROACTIVE） |
| `ScheduleCronTool` | 定时触发 |
| `RemoteTriggerTool` | 远程触发 |
| `SyntheticOutputTool` | 结构化输出生成 |
| `TodoWriteTool` | 待办事项管理 |
| `BriefTool` | 简报生成 |
| `ConfigTool` | 配置管理 |

### 6.4 工具注册表（`src/tools.ts`）

```typescript
// 始终启用的工具（直接导入）
import { BashTool } from './tools/BashTool/BashTool.js'
import { FileEditTool } from './tools/FileEditTool/FileEditTool.js'

// 条件启用的工具（特性标志）
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

// 懒加载工具（解决循环依赖）
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool

export function getTools(): Tools {
  return [
    BashTool, FileEditTool, /* ... */,
    ...(SleepTool ? [SleepTool] : []),
  ]
}
```

---

## 7. 命令系统（Command System）

### 7.1 三种命令类型

| 类型 | 描述 | 示例 |
|------|------|------|
| **PromptCommand** | 构建格式化提示发送给 LLM，可注入工具 | `/review`, `/commit` |
| **LocalCommand** | 进程内执行，返回纯文本 | `/cost`, `/version` |
| **LocalJSXCommand** | 进程内执行，返回 JSX 组件 | `/doctor`, `/install` |

### 7.2 PromptCommand 示例

```typescript
{
  type: 'prompt',
  name: 'review',
  description: '代码审查',
  allowedTools: ['Bash(git *)', 'FileRead(*)'],
  getPromptForCommand(args, context) {
    return [{ type: 'text', text: '请审查以下代码变更...' }]
  },
}
```

### 7.3 主要命令列表（~50 个）

| 命令 | 描述 | | 命令 | 描述 |
|------|------|---|------|------|
| `/commit` | Git 提交 | | `/memory` | 持久化记忆管理 |
| `/review` | 代码审查 | | `/skills` | 技能管理 |
| `/compact` | 上下文压缩 | | `/tasks` | 任务管理 |
| `/mcp` | MCP 服务器管理 | | `/vim` | Vim 模式切换 |
| `/config` | 设置配置 | | `/diff` | 查看变更 |
| `/doctor` | 环境诊断 | | `/cost` | 使用成本查看 |
| `/login` / `/logout` | 认证 | | `/theme` | 主题切换 |
| `/context` | 上下文可视化 | | `/share` | 分享会话 |
| `/plan` | 计划模式 | | `/resume` | 恢复会话 |
| `/branch` | 分支操作 | | `/agents` | 智能体管理 |

---

## 8. 任务系统（Task System）

### 8.1 任务类型

```typescript
type TaskType =
  | 'local_bash'          // 本地 Shell 命令
  | 'local_agent'         // 本地智能体生成
  | 'remote_agent'        // 远程智能体生成
  | 'in_process_teammate' // 同进程队友智能体
  | 'local_workflow'      // 工作流脚本（特性：WORKFLOW_SCRIPTS）
  | 'monitor_mcp'         // MCP 监控（特性：MONITOR_TOOL）
  | 'dream'               // 实验性后台思考
```

### 8.2 任务状态机

```
pending ──► running ──► completed
                   └──► failed
                   └──► killed
```

`isTerminalTaskStatus()` 函数防止向已终止的任务注入消息。

### 8.3 任务上下文

```typescript
type TaskContext = {
  abortController: AbortController  // 取消控制
  getAppState: () => AppState        // 读取可变状态
  setAppState: SetAppState           // 更新状态
}
```

### 8.4 任务注册表（`src/tasks.ts`）

```typescript
export function getAllTasks(): Task[] {
  const tasks: Task[] = [
    LocalShellTask,    // src/tasks/LocalShellTask/
    LocalAgentTask,    // src/tasks/LocalAgentTask/
    RemoteAgentTask,   // src/tasks/RemoteAgentTask/
    DreamTask,         // src/tasks/DreamTask/（实验性）
  ]
  // 特性标志条件加载
  if (LocalWorkflowTask) tasks.push(LocalWorkflowTask)
  if (MonitorMcpTask) tasks.push(MonitorMcpTask)
  return tasks
}
```

### 8.5 多智能体协调（Coordinator）

**特性标志：** `feature('COORDINATOR_MODE')`  
**位置：** `src/coordinator/coordinatorMode.ts`

- 主管智能体生成工作智能体（通过 `AgentTool`）
- 工作智能体通过 `SendMessageTool` 通信
- Coordinator 模式为工作智能体注入特殊上下文
- `getCoordinatorUserContext()` 自定义工作智能体的系统提示

---

## 9. 状态管理

### 9.1 AppState 架构

```typescript
type AppState = {
  // 任务
  tasks: Record<string, TaskState>
  localAgentCount: number

  // 消息与历史
  messages: Message[]
  selectedMessages: Set<string>

  // 用户输入
  pendingUserInput: string

  // 权限
  toolPermissionContext: ToolPermissionContext

  // 状态快照
  fileStateCache: FileStateCache        // 文件内容缓存
  attributionState: AttributionState    // 提交归属
  fileHistoryState: FileHistoryState    // 文件历史

  // UI 状态
  selectedTaskId?: string
  theme: ThemeName

  // 队友 & 团队
  teammates: Teammate[]
  teams: Team[]
  isTeammateViewExpanded: boolean
  selectedTeammateId?: AgentId
}
```

### 9.2 状态流向

```
┌─────────────────────────────────────┐
│     AppStateStore（类 Zustand）      │
│  getState() → AppState              │
│  setState(fn) → void                │
└─────────────┬───────────────────────┘
              │
    ┌─────────┼─────────────────────┐
    ▼         ▼                     ▼
┌─────────┐ ┌────────────┐ ┌─────────────┐
│ React   │ │ 选择器     │ │ 观察者      │
│ 上下文  │ │ (Selectors)│ │ (Observers) │
│ 提供者  │ └────────────┘ └─────────────┘
└─────────┘
```

**关键文件：**
- `src/state/AppState.tsx` — React Context + Compiler hooks
- `src/state/AppStateStore.ts` — 不可变存储 + 选择器模式
- `src/state/store.ts` — 自定义存储实现
- `src/state/onChangeAppState.ts` — 状态变化观察者

---

## 10. UI 层（React + Ink）

### 10.1 技术选型

Claude Code 使用 [Ink](https://github.com/vadimdemedes/ink) 将 React 的组件模型带入终端：

- 函数式 React 组件使用 Ink 原语（`Box`, `Text`, `useInput()`）
- 用 [Chalk](https://github.com/chalk/chalk) 为终端着色
- 启用 React Compiler 优化重渲染
- `src/components/design-system/` 提供基础设计系统组件

### 10.2 UI 组件层级

```
src/screens/REPL.tsx            // 主 REPL 界面（默认屏幕）
  └── src/components/App.tsx    // 顶层应用组件
      ├── MessageList.tsx        // 消息历史列表
      │   ├── Message.tsx        // 单条消息渲染
      │   │   ├── ToolUse.tsx    // 工具调用渲染
      │   │   └── ToolResult.tsx // 工具结果渲染
      ├── InputBar.tsx           // 用户输入栏
      ├── TaskList.tsx           // 后台任务列表
      ├── TeammateList.tsx       // 团队成员视图
      └── StatusBar.tsx          // 状态栏
```

### 10.3 全屏 UI

| 屏幕 | 触发 | 用途 |
|------|------|------|
| `REPL.tsx` | 默认 | 主交互 REPL |
| `Doctor.tsx` | `/doctor` | 环境诊断 |
| `ResumeConversation.tsx` | `/resume` | 会话恢复 |

### 10.4 核心 Hooks（~80 个）

| 类别 | Hook | 用途 |
|------|------|------|
| 权限 | `useCanUseTool` | 工具调用权限检查 |
| IDE 集成 | `useIDEIntegration`, `useDiffInIDE` | IDE 交互 |
| 输入处理 | `useTextInput`, `useVimInput`, `usePasteHandler` | 终端输入 |
| 会话 | `useSessionBackgrounding`, `useRemoteSession` | 会话管理 |
| 通知 | `useRateLimitNotification`, `useDeprecationWarning` | 系统通知 |

---

## 11. 权限系统

### 11.1 权限模式

| 模式 | 描述 |
|------|------|
| `default` | 每次工具调用均提示用户批准 |
| `plan` | 展示计划，一次性批准 |
| `bypassPermissions` | 自动批准所有操作 |
| `auto` | ML 分类器（如可用）自动判断 |

### 11.2 权限上下文

```typescript
type ToolPermissionContext = {
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource  // 白名单通配符模式
  alwaysDenyRules: ToolPermissionRulesBySource   // 黑名单
  alwaysAskRules: ToolPermissionRulesBySource    // 强制询问
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  shouldAvoidPermissionPrompts?: boolean         // 后台智能体使用
  awaitAutomatedChecksBeforeDialog?: boolean     // Coordinator worker 使用
}
```

### 11.3 通配符规则示例

```
Bash(git *)        // 允许所有 git 命令
FileEdit(/src/*)   // 允许编辑 /src/ 下所有文件
FileRead(*.md)     // 允许读取所有 Markdown 文件
```

### 11.4 权限检查流程

```
工具调用请求
  │
  ▼
tool.checkPermissions(input, context)
  │
  ├── 通配符规则匹配（always_allow / always_deny / always_ask）
  ├── 权限模式检查
  └── [如需用户确认] 显示对话框 → 等待用户输入
        ├── 批准 → 执行工具
        └── 拒绝 → 返回错误信息给 LLM
```

---

## 12. Bridge / IDE 集成

### 12.1 架构

```
┌──────────────────┐    JWT 认证     ┌──────────────────────┐
│   IDE 插件       │◄───────────────►│   Bridge 层          │
│  (VS Code/JB)    │  双向通信       │  (src/bridge/)       │
│                  │                 │                      │
│  - UI 渲染       │                 │  - 会话管理          │
│  - 文件监视      │                 │  - 消息路由          │
│  - Diff 展示     │                 │  - 权限代理          │
└──────────────────┘                 └──────────┬───────────┘
                                               │
                                               ▼
                                     ┌──────────────────────┐
                                     │   Claude Code 核心   │
                                     │  (QueryEngine, Tools)│
                                     └──────────────────────┘
```

### 12.2 关键文件

| 文件 | 职责 |
|------|------|
| `src/bridge/bridgeMain.ts` | Bridge 主循环 |
| `src/bridge/bridgeMessaging.ts` | RPC 协议层 |
| `src/bridge/bridgePermissionCallbacks.ts` | 权限回调流 |
| `src/bridge/replBridge.ts` | REPL 会话绑定 |
| `src/bridge/jwtUtils.ts` | JWT 认证 |
| `src/bridge/sessionRunner.ts` | 会话执行 |

---

## 13. MCP 协议集成

### 13.1 MCP 客户端（`src/services/mcp/`）

Claude Code 可作为 MCP **客户端**连接到外部 MCP 服务器，通过 `MCPTool` 调用服务器提供的工具。

**工作流：**
```
MCPTool.call()
  ├── 查找对应 MCPServerConnection
  ├── 发送工具调用请求
  └── 返回结果给 LLM
```

### 13.2 MCP 服务器模式（`src/entrypoints/mcp.ts`）

Claude Code 本身也可作为 MCP **服务器**运行，将自身能力暴露给其他 MCP 客户端。

### 13.3 独立 MCP Explorer 服务器（`mcp-server/`）

项目内置一个独立的 MCP 服务器（发布为 `claude-code-explorer-mcp` npm 包），允许任何 MCP 客户端探索 Claude Code 源码：

| 工具 | 描述 |
|------|------|
| `list_tools` | 列出所有 ~40 个工具 |
| `list_commands` | 列出所有 ~50 个命令 |
| `get_tool_source` | 读取工具源码 |
| `get_command_source` | 读取命令源码 |
| `read_source_file` | 读取任意 src/ 文件 |
| `search_source` | 在源码树中搜索 |
| `get_architecture` | 架构概览 |

**支持的传输协议：**
- **STDIO** — 标准输入输出（Claude Desktop、VS Code）
- **Streamable HTTP** — `POST/GET /mcp`
- **Legacy SSE** — `GET /sse` + `POST /messages`

---

## 14. 服务层（Services）

`src/services/` 目录包含所有外部集成和核心基础设施：

| 服务 | 位置 | 描述 |
|------|------|------|
| **API 客户端** | `services/api/` | Anthropic SDK 包装器、文件 API、引导 |
| **MCP 连接** | `services/mcp/` | Model Context Protocol 连接管理 |
| **OAuth 认证** | `services/oauth/` | OAuth 2.0 认证流程 |
| **LSP 管理** | `services/lsp/` | Language Server Protocol 管理器 |
| **特性标志** | `services/analytics/` | GrowthBook 特性标志 & A/B 测试 |
| **插件加载** | `services/plugins/` | 插件加载器 |
| **上下文压缩** | `services/compact/` | 对话上下文压缩 |
| **记忆提取** | `services/extractMemories/` | 自动记忆提取 |
| **团队记忆** | `services/teamMemorySync/` | 团队记忆同步 |
| **Token 估算** | `services/tokenEstimation.ts` | Token 计数估算 |
| **策略限制** | `services/policyLimits/` | 企业策略限制 |
| **远程配置** | `services/remoteManagedSettings/` | 远程托管设置 |

---

## 15. 插件与技能系统

### 15.1 技能系统（`src/skills/`）

技能是可复用的多步骤工作流，通过 `SkillTool` 被 LLM 调用：

```typescript
export const MySkill = buildSkill({
  name: 'my-skill',
  description: '复杂多步骤工作流',
  async run(context) {
    // 可以使用任何可用工具
    // Claude 控制执行流程
  },
})
```

**系统提示注入模式：**
```
"可用技能列表：[skill1, skill2, ...]"
// LLM 调用: SkillTool({ skillName: 'my-skill', args: {...} })
```

**技能来源：**
- `src/skills/bundledSkills.ts` — 内置技能
- `src/skills/loadSkillsDir.ts` — 从目录加载用户自定义技能

### 15.2 插件系统（`src/plugins/`）

| 文件 | 描述 |
|------|------|
| `src/plugins/builtinPlugins.ts` | 内置插件定义 |
| `src/plugins/bundled/` | 捆绑插件 |
| `src/utils/plugins/pluginLoader.ts` | 插件动态加载器 |

### 15.3 持久化记忆（`src/memdir/`）

- 通过 `CLAUDE.md` 文件实现跨会话持久化记忆
- `src/memdir/memdir.ts` — 记忆目录管理
- `src/memdir/claudemd.ts` — CLAUDE.md 解析与写入
- 支持多层记忆（用户级、项目级、企业级）

---

## 16. 构建系统与特性标志

### 16.1 Bun 运行时特性

Claude Code 运行在 [Bun](https://bun.sh) 而非 Node.js 上，关键优势：

- 原生 JSX/TSX 支持，无需转译步骤
- `bun:bundle` 特性标志实现编译期死代码消除
- ES modules 使用 `.js` 扩展名（Bun 约定）

### 16.2 特性标志系统

```typescript
import { feature } from 'bun:bundle'

// 非激活特性标志内的代码在构建时被完全剔除
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

### 16.3 主要特性标志

| 标志 | 功能 |
|------|------|
| `PROACTIVE` | 主动智能体模式（自主触发动作） |
| `KAIROS` | Kairos 子系统 |
| `BRIDGE_MODE` | IDE Bridge 集成 |
| `DAEMON` | 后台守护进程模式 |
| `VOICE_MODE` | 语音输入/输出 |
| `AGENT_TRIGGERS` | 触发型智能体动作 |
| `MONITOR_TOOL` | 监控工具 |
| `COORDINATOR_MODE` | 多智能体协调器 |
| `WORKFLOW_SCRIPTS` | 工作流自动化脚本 |
| `HISTORY_SNIP` | 历史消息片段压缩 |

### 16.4 懒加载策略

```typescript
// 重型模块延迟加载，仅在首次使用时按需导入
// OpenTelemetry (~400KB) 和 gRPC (~700KB)
const otel = await import('@opentelemetry/sdk-node')
```

### 16.5 代码质量工具

| 工具 | 配置文件 | 用途 |
|------|----------|------|
| [Biome](https://biomejs.dev) | `biome.json` | 代码格式化 + Lint（替代 ESLint + Prettier） |
| TypeScript | `tsconfig.json` | 严格模式类型检查 |
| Bun | `bunfig.toml` | 运行时与打包配置 |

---

## 17. 数据流全景

### 17.1 完整交互数据流

```
[用户终端输入]
      │
      ▼
REPL.tsx (Ink 组件)
      │ 特殊字符转义
      ▼
processUserInput()
  ├── 解析 /斜杠命令 → 分发给 Command 处理器
  ├── 处理 @mentions（文件/智能体引用）
  └── 普通文本 → 继续
      │
      ▼
QueryEngine.query()
      │
      ├── 构建 System Prompt
      │   ├── 工具描述（来自各工具的 prompt()）
      │   ├── 上下文（git 状态、记忆、工作目录）
      │   └── 技能注入
      │
      ├── 调用 Anthropic API（流式）
      │         ↓ 流式响应
      │   ┌─────────────────────────────────┐
      │   │  处理流式事件                   │
      │   │  ├── text_block → 累积文本      │
      │   │  ├── tool_use_block → 工具调用  │
      │   │  └── thinking_block → 思维内容  │
      │   └─────────────────────────────────┘
      │
      ├── [如有 tool_use]工具执行循环
      │   ├── 权限检查 (useCanUseTool)
      │   ├── Zod Schema 验证
      │   ├── 执行工具
      │   │   ├── 同步工具 → 立即返回
      │   │   └── 异步工具 → 进度更新 + 等待完成
      │   ├── 收集 tool_result
      │   └── 将结果反馈给 LLM → 继续循环
      │
      ▼
消息组装
  ├── UserMessage（用户文本）
  ├── AssistantMessage（LLM 响应）
  ├── ToolUseBlockParam（工具调用记录）
  └── ToolResultBlockParam（工具结果记录）
      │
      ▼
AppState.setAppState(s => ({ ...s, messages: [...] }))
      │
      ▼
React 重渲染 → 终端 UI 更新
  ├── MessageList 更新
  ├── ToolResult 渲染
  └── 成本信息更新
```

### 17.2 消息类型体系

```typescript
type Message =
  | UserMessage             // 用户输入
  | AssistantMessage        // LLM 回复
  | SystemMessage           // 系统消息（内部）
  | ProgressMessage         // 工具进度消息
  | AttachmentMessage       // 附件消息
  | SystemLocalCommandMessage  // 本地命令输出
```

---

## 18. 关键设计模式总结

### 18.1 并行预取（Parallel Prefetch）

**问题：** CLI 启动延迟高，因 MDM 读取和 Keychain I/O 阻塞主流程

**解决方案：** 在主模块评估之前触发异步副作用

```typescript
// main.tsx - 最早执行
startMdmRawRead()              // 立即触发，不等待
startKeychainPrefetch()        // 立即触发，不等待
// 后续约 135ms 模块加载与这些 I/O 并发执行
```

### 18.2 工厂函数模式（Factory Function Pattern）

所有工具、任务、命令均通过工厂函数定义，确保类型安全和接口一致：

```typescript
buildTool({ name, inputSchema, call, checkPermissions, ... })
buildSkill({ name, description, run })
```

### 18.3 注册表模式（Registry Pattern）

工具、命令、任务均通过集中注册表管理，支持条件注册：

```typescript
// tools.ts
export function getTools(): Tools { return [/* 按条件组装 */] }

// tasks.ts
export function getAllTasks(): Task[] { return [/* 按条件组装 */] }
```

### 18.4 特性标志死代码消除（Feature Flag DCE）

利用 Bun 的 `bun:bundle` 实现编译期 tree-shaking，未激活的特性零运行时开销：

```typescript
import { feature } from 'bun:bundle'
// 特性未激活时，整个分支在编译时被剔除
const voiceModule = feature('VOICE_MODE') ? require('./voice') : null
```

### 18.5 惰性循环依赖解决（Lazy Circular Dependency Breaking）

存在循环依赖的模块使用懒加载 `require()` 延迟解析：

```typescript
// 避免 ESM 静态导入的循环依赖问题
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```

### 18.6 React + Ink 终端 UI 模式

将 Web 开发的 React 组件模型完整移植到终端：

- 声明式 UI，状态驱动渲染
- 组件化架构，关注点分离
- Hooks 封装副作用逻辑
- React Compiler 优化性能

### 18.7 Schema 驱动的工具接口（Schema-Driven Tool Interface）

工具的输入 Schema 同时用于：
1. 运行时输入验证（Zod）
2. 生成发送给 Claude 的 JSON Schema（工具描述）
3. TypeScript 类型推断

```typescript
const inputSchema = z.object({ path: z.string(), content: z.string() })
// 同时满足：类型推断 + 运行时验证 + API Schema 生成
```

### 18.8 递归智能体模式（Recursive Agent Pattern）

`AgentTool` 通过创建新的 `QueryEngine` 实例实现子智能体，支持无限嵌套深度的智能体树：

```
用户 → 主 QueryEngine
         └── AgentTool → 子 QueryEngine 1
               └── AgentTool → 孙 QueryEngine 2
                                 └── ...（递归）
```

---

## 附录：Web 前端（`web/`）

Claude Code 包含一个基于 Next.js 的 Web 前端，作为 CLI 的图形界面伴侣：

| 技术 | 版本 |
|------|------|
| Next.js | 14.2.0（App Router） |
| React | 18.3.0 |
| Zustand | 4.5.0（状态管理） |
| Radix UI | 各组件版本 |
| Tailwind CSS | — |
| Shiki | 1.10.0（代码高亮） |
| SWR | 2.2.0（数据获取） |
| Framer Motion | 11.0.0（动画） |

---

*本分析基于 2026-03-31 通过 npm source map 泄露的 Claude Code 源码，版本对应 Anthropic 官方 npm 包中的原始代码。所有源码版权归 [Anthropic](https://www.anthropic.com) 所有。*
