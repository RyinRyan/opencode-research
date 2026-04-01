# OpenCode 插件开发完整指南

> 版本: 基于 OpenCode 源码分析  
> 生成日期: 2026-04-01  
> 适用版本: OpenCode 1.3.13+

---

## 目录

1. [OpenCode 消息发送机制](#一opencode-消息发送机制)
2. [事件系统详解](#二事件系统详解)
3. [事件触发时机](#三事件触发时机)
4. [插件框架基础](#四插件框架基础)
5. [完整示例代码](#五完整示例代码)
6. [最佳实践](#六最佳实践)

---

## 一、OpenCode 消息发送机制

### 1.1 架构概述

OpenCode 采用**事件驱动架构 (Event-Driven Architecture)**，通过内部事件总线 (Event Bus) 实现各组件间的松耦合通信。

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode Core                            │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   Session   │    │   Message    │    │    Tool      │   │
│  │   Manager   │    │   Handler    │    │   Executor   │   │
│  └──────┬──────┘    └──────┬───────┘    └──────┬───────┘   │
│         │                  │                   │           │
│         └──────────────────┼───────────────────┘           │
│                            │                               │
│                    ┌───────▼───────┐                       │
│                    │   Event Bus   │                       │
│                    │  (Internal)   │                       │
│                    └───────┬───────┘                       │
│                            │                               │
│         ┌──────────────────┼───────────────────┐           │
│         │                  │                   │           │
│  ┌──────▼──────┐    ┌──────▼───────┐    ┌──────▼───────┐   │
│  │   Plugin    │    │     TUI      │    │     LSP      │   │
│  │   Hooks     │    │   Interface  │    │   Servers    │   │
│  └─────────────┘    └──────────────┘    └──────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 消息流转流程

```
用户输入
    │
    ▼
┌─────────────────┐
│  1. 消息接收     │  ← chat.message Hook
│  (User Message) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  2. 参数处理     │  ← chat.params Hook
│  (Modify Params)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  3. 发送 LLM    │  ← chat.headers Hook
│  (HTTP Request) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  4. 工具调用     │  ← tool.execute.before
│  (Tool Call)    │  ← tool.execute.after
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  5. 响应生成     │  ← message.part.updated
│  (Response)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  6. 会话完成     │  ← session.idle
│  (Session Idle) │
└─────────────────┘
```

### 1.3 事件传播机制

OpenCode 使用两种事件传播方式：

| 传播方式 | 说明 | 使用场景 |
|---------|------|---------|
| **同步事件** | 阻塞式执行，可修改数据 | Hooks (chat.params, tool.execute.before 等) |
| **异步事件** | 非阻塞通知，只读数据 | 系统事件 (session.created, file.edited 等) |

---

## 二、事件系统详解

### 2.1 事件分类

OpenCode 的事件分为 **3 大类**：

#### 2.1.1 系统通知事件 (System Events)
通过 `event` Hook 接收，用于状态通知

#### 2.1.2 拦截器 Hooks (Interceptor Hooks)
在操作执行前后触发，可修改输入输出

#### 2.1.3 TUI 事件 (TUI Events)
终端界面相关的事件

### 2.2 完整事件列表

#### A. 系统通知事件 (46 种)

##### 会话生命周期事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `session.created` | 通知 | 新会话创建 |
| `session.updated` | 通知 | 会话信息更新 |
| `session.deleted` | 通知 | 会话被删除 |
| `session.status` | 通知 | 会话状态变化 |
| `session.idle` | 通知 | 会话进入空闲状态 |
| `session.compacted` | 通知 | 会话压缩完成 |
| `session.diff` | 通知 | 会话文件差异更新 |
| `session.error` | 通知 | 会话发生错误 |

##### 消息事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `message.updated` | 通知 | 消息更新 |
| `message.removed` | 通知 | 消息删除 |
| `message.part.updated` | 通知 | 消息部分内容更新 |
| `message.part.removed` | 通知 | 消息部分内容删除 |
| `message.part.delta` | 增量 | 消息内容增量更新（流式） |

##### 文件事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `file.edited` | 通知 | 文件被编辑 |
| `file.watcher.updated` | 通知 | 文件监控检测到变化 |

##### 权限事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `permission.asked` | 通知 | 请求用户权限 |
| `permission.replied` | 通知 | 用户回复权限请求 |

##### 工具/命令事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `command.executed` | 通知 | 斜杠命令被执行 |
| `todo.updated` | 通知 | 待办事项更新 |

##### LSP 事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `lsp.updated` | 通知 | LSP 服务器状态更新 |
| `lsp.client.diagnostics` | 通知 | LSP 诊断信息更新 |

##### MCP 事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `mcp.tools.changed` | 通知 | MCP 工具列表变化 |
| `mcp.browser.open.failed` | 通知 | MCP 浏览器打开失败 |

##### 终端事件 (PTY)
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `pty.created` | 通知 | PTY 终端创建 |
| `pty.updated` | 通知 | PTY 终端更新 |
| `pty.exited` | 通知 | PTY 终端退出 |
| `pty.deleted` | 通知 | PTY 终端删除 |

##### 工作空间事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `workspace.ready` | 通知 | 工作空间就绪 |
| `workspace.failed` | 通知 | 工作空间初始化失败 |
| `worktree.ready` | 通知 | Git worktree 就绪 |
| `worktree.failed` | 通知 | Git worktree 初始化失败 |
| `vcs.branch.updated` | 通知 | Git 分支变化 |

##### TUI 事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `tui.prompt.append` | 通知 | TUI 提示框追加文本 |
| `tui.command.execute` | 通知 | TUI 执行命令 |
| `tui.toast.show` | 通知 | TUI 显示通知 |
| `tui.session.select` | 通知 | TUI 选择会话 |

##### 问答事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `question.asked` | 通知 | 向用户提问 |
| `question.replied` | 通知 | 用户回答问题 |
| `question.rejected` | 通知 | 用户拒绝回答 |

##### 项目/安装事件
| 事件名称 | 类型 | 描述 |
|---------|------|------|
| `project.updated` | 通知 | 项目配置更新 |
| `installation.updated` | 通知 | OpenCode 安装更新 |
| `installation.update-available` | 通知 | 有新版本可用 |
| `server.instance.disposed` | 通知 | 服务器实例释放 |
| `server.connected` | 通知 | 服务器连接成功 |
| `global.disposed` | 通知 | 全局资源释放 |

#### B. 拦截器 Hooks (14 种)

| Hook 名称 | 触发时机 | 可修改 | 用途 |
|----------|---------|--------|------|
| `config` | 配置加载时 | ✅ | 修改配置 |
| `chat.message` | 收到用户消息时 | ✅ | 修改消息内容 |
| `chat.params` | 发送 LLM 请求前 | ✅ | 修改模型参数 |
| `chat.headers` | 发送 HTTP 请求前 | ✅ | 修改请求头 |
| `permission.ask` | 请求权限时 | ✅ | 自定义权限处理 |
| `command.execute.before` | 执行命令前 | ✅ | 修改命令行为 |
| `tool.execute.before` | 工具执行前 | ✅ | 验证/修改参数 |
| `tool.execute.after` | 工具执行后 | ✅ | 修改输出结果 |
| `tool.definition` | 工具定义发送前 | ✅ | 修改工具描述 |
| `shell.env` | 创建 Shell 环境时 | ✅ | 注入环境变量 |
| `experimental.chat.messages.transform` | 消息转换时 | ✅ | 转换消息列表 |
| `experimental.chat.system.transform` | 系统提示转换时 | ✅ | 修改系统提示 |
| `experimental.session.compacting` | 会话压缩前 | ✅ | 自定义压缩提示 |
| `experimental.text.complete` | 文本补全时 | ✅ | 修改补全结果 |

---

## 三、事件触发时机

### 3.1 会话生命周期事件流

```
[用户创建会话]
    │
    ▼
session.created ────────────────► 会话创建完成
    │
    ▼
[用户发送消息]
    │
    ▼
chat.message ───────────────────► 收到用户消息
    │
    ▼
chat.params ────────────────────► 准备 LLM 参数
    │
    ▼
chat.headers ───────────────────► 准备 HTTP 头
    │
    ▼
[LLM 处理中...]
    │
    ▼
message.part.delta ─────────────► 流式接收内容
    │
    ▼
tool.execute.before ────────────► LLM 请求调用工具
    │
    ▼
[工具执行中...]
    │
    ▼
tool.execute.after ─────────────► 工具执行完成
    │
    ▼
message.part.updated ───────────► 消息部分更新
    │
    ▼
[LLM 响应完成]
    │
    ▼
session.idle ───────────────────► 会话进入空闲
    │
    ▼
[上下文过长]
    │
    ▼
experimental.session.compacting ─► 开始压缩会话
    │
    ▼
session.compacted ──────────────► 会话压缩完成
```

### 3.2 文件操作事件流

```
[AI 编辑文件]
    │
    ▼
file.edited ────────────────────► 文件被编辑
    │
    ▼
file.watcher.updated ───────────► 文件监控检测到变化
    │                           (add/change/unlink)
    ▼
session.diff ───────────────────► 会话差异更新
```

### 3.3 权限请求事件流

```
[工具需要权限]
    │
    ▼
permission.ask ─────────────────► 拦截器：可修改权限策略
    │
    ▼
permission.asked ───────────────► 系统事件：通知 UI 显示权限请求
    │
    ▼
[用户做出选择]
    │
    ▼
permission.replied ─────────────► 用户回复权限请求
```

### 3.4 详细触发时机表

| 事件/Hook | 触发时机 | 数据可用性 |
|----------|---------|-----------|
| `session.created` | 调用 `client.session.create()` 成功后 | 完整 Session 对象 |
| `session.updated` | 会话元数据变化（标题、权限等） | 更新的字段 |
| `session.deleted` | 调用 `client.session.delete()` 后 | Session ID 和快照 |
| `session.status` | 状态从 busy/idle/retry 变化时 | 新状态 |
| `session.idle` | LLM 响应完成且所有工具执行完毕 | Session ID |
| `chat.message` | 用户消息被接收后、处理前 | 原始消息和 parts |
| `chat.params` | 构建 LLM 请求参数时 | temperature, topP, topK, options |
| `chat.headers` | 构建 HTTP 请求头时 | headers 对象 |
| `message.part.delta` | 流式接收 LLM 响应时 | field, delta (增量文本) |
| `message.part.updated` | 消息部分状态变化时 | 完整 Part 对象 |
| `tool.execute.before` | LLM 调用工具、实际执行前 | tool 名称, callID, args |
| `tool.execute.after` | 工具执行完成后 | tool, args, 原始输出 |
| `shell.env` | 每次创建 Shell 实例时 | cwd, sessionID?, callID? |
| `file.edited` | AI 调用编辑工具修改文件后 | 文件路径 |
| `file.watcher.updated` | 文件系统监控检测到变化时 | 文件路径, 事件类型 |
| `experimental.session.compacting` | 上下文超过限制、需要压缩前 | sessionID |

---

## 四、插件框架基础

### 4.1 最小插件结构

```typescript
// .opencode/plugins/minimal.ts
export const MyPlugin = async (ctx) => {
  // ctx 包含:
  // - client: OpenCode SDK 客户端
  // - project: 当前项目信息
  // - directory: 当前工作目录
  // - worktree: Git worktree 路径
  // - serverUrl: 服务器 URL
  // - $: Bun Shell API

  console.log("Plugin loaded!")

  return {
    // Hooks 返回这里
  }
}
```

### 4.2 带类型的插件结构

```typescript
// .opencode/plugins/typed.ts
import type { Plugin } from "@opencode-ai/plugin"

export const TypedPlugin: Plugin = async ({
  client,
  project,
  directory,
  worktree,
  serverUrl,
  $
}) => {
  // 使用结构化日志
  await client.app.log({
    body: {
      service: "my-plugin",
      level: "info",
      message: "Plugin initialized",
      extra: { project: project.name }
    }
  })

  return {
    // Hooks
  }
}
```

### 4.3 插件输入参数详解

```typescript
interface PluginInput {
  // OpenCode SDK 客户端
  // 用于调用 OpenCode API
  client: ReturnType<typeof createOpencodeClient>
  
  // 当前项目信息
  project: {
    id: string
    worktree: string
    vcs?: "git"
    name?: string
    icon?: { url?: string; override?: string; color?: string }
    commands?: { start?: string }
    time: { created: number; updated: number; initialized?: number }
    sandboxes: string[]
  }
  
  // 当前工作目录（绝对路径）
  directory: string
  
  // Git worktree 根目录
  worktree: string
  
  // OpenCode 服务器 URL
  serverUrl: URL
  
  // Bun Shell API
  // 文档: https://bun.sh/docs/runtime/shell
  $: BunShell
}
```

### 4.4 Hooks 返回对象结构

```typescript
interface Hooks {
  // ========== 系统事件监听 ==========
  event?: (input: { event: Event }) => Promise<void>
  
  // ========== 配置处理 ==========
  config?: (input: Config) => Promise<void>
  
  // ========== 聊天处理 ==========
  "chat.message"?: (input, output) => Promise<void>
  "chat.params"?: (input, output) => Promise<void>
  "chat.headers"?: (input, output) => Promise<void>
  
  // ========== 权限处理 ==========
  "permission.ask"?: (input, output) => Promise<void>
  
  // ========== 命令处理 ==========
  "command.execute.before"?: (input, output) => Promise<void>
  
  // ========== 工具处理 ==========
  "tool.execute.before"?: (input, output) => Promise<void>
  "tool.execute.after"?: (input, output) => Promise<void>
  "tool.definition"?: (input, output) => Promise<void>
  
  // ========== Shell 环境 ==========
  "shell.env"?: (input, output) => Promise<void>
  
  // ========== 实验性功能 ==========
  "experimental.chat.messages.transform"?: (input, output) => Promise<void>
  "experimental.chat.system.transform"?: (input, output) => Promise<void>
  "experimental.session.compacting"?: (input, output) => Promise<void>
  "experimental.text.complete"?: (input, output) => Promise<void>
  
  // ========== 自定义工具 ==========
  tool?: { [key: string]: ToolDefinition }
  
  // ========== 认证 ==========
  auth?: AuthHook
}
```

---

## 五、完整示例代码

### 5.1 基础事件监听插件

```typescript
// .opencode/plugins/event-monitor.ts
import type { Plugin } from "@opencode-ai/plugin"

/**
 * 事件监控插件
 * 监听并记录所有系统事件
 */
export const EventMonitorPlugin: Plugin = async ({ client }) => {
  // 插件初始化日志
  await client.app.log({
    body: {
      service: "event-monitor",
      level: "info",
      message: "Event monitor plugin started"
    }
  })

  return {
    event: async ({ event }) => {
      const timestamp = new Date().toISOString()
      
      // 根据事件类型处理
      switch (event.type) {
        case "session.created":
          console.log(`[${timestamp}] 🆕 Session created: ${event.properties.sessionID}`)
          break
          
        case "session.idle":
          console.log(`[${timestamp}] ✅ Session idle: ${event.properties.sessionID}`)
          break
          
        case "message.part.delta":
          // 流式更新，避免频繁日志
          process.stdout.write(event.properties.delta)
          break
          
        case "file.edited":
          console.log(`[${timestamp}] 📝 File edited: ${event.properties.file}`)
          break
          
        case "permission.asked":
          console.log(`[${timestamp}] 🔒 Permission asked: ${event.properties.permission}`)
          break
          
        case "tool.execute.before":
          console.log(`[${timestamp}] 🔧 Tool executing: ${event.properties.tool}`)
          break
          
        default:
          console.log(`[${timestamp}] 📡 Event: ${event.type}`)
      }
    }
  }
}
```

### 5.2 消息处理插件

```typescript
// .opencode/plugins/message-handler.ts
import type { Plugin } from "@opencode-ai/plugin"

/**
 * 消息处理插件
 * 修改聊天参数和消息内容
 */
export const MessageHandlerPlugin: Plugin = async () => {
  return {
    // 修改 LLM 请求参数
    "chat.params": async (input, output) => {
      // 根据 Agent 调整参数
      if (input.agent === "code-review") {
        // 代码审查时降低随机性
        output.temperature = 0.1
        output.topP = 0.9
      } else if (input.agent === "creative") {
        // 创意任务时增加随机性
        output.temperature = 0.9
        output.topP = 0.95
      }
      
      // 添加自定义选项
      output.options.customOption = "value"
    },

    // 修改 HTTP 请求头
    "chat.headers": async (input, output) => {
      // 添加追踪头
      output.headers["X-Request-ID"] = crypto.randomUUID()
      output.headers["X-Session-ID"] = input.sessionID
      
      // 添加自定义认证头（如果需要）
      // output.headers["Authorization"] = "Bearer token"
    },

    // 处理用户消息
    "chat.message": async (input, output) => {
      // 可以修改消息内容
      // output.message.content = modifiedContent
      
      // 可以添加额外的 parts
      // output.parts.push(additionalPart)
      
      console.log(`Processing message in session: ${input.sessionID}`)
      console.log(`Agent: ${input.agent}, Model: ${input.model?.modelID}`)
    }
  }
}
```

### 5.3 工具拦截插件

```typescript
// .opencode/plugins/tool-interceptor.ts
import type { Plugin } from "@opencode-ai/plugin"

/**
 * 工具拦截插件
 * 验证和修改工具调用
 */
export const ToolInterceptorPlugin: Plugin = async ({ client }) => {
  // 危险命令列表
  const dangerousPatterns = [
    /rm\s+-rf\s+\//,
    />\s*\/dev\/null/,
    /mkfs\./,
    /dd\s+if=/
  ]
  
  // 敏感文件列表
  const sensitiveFiles = [
    ".env",
    ".env.local",
    ".env.production",
    "id_rsa",
    "id_ed25519",
    ".aws/credentials",
    ".ssh/config"
  ]

  return {
    // 工具执行前拦截
    "tool.execute.before": async (input, output) => {
      console.log(`[Interceptor] Tool: ${input.tool}, CallID: ${input.callID}`)
      
      // 拦截 bash 命令
      if (input.tool === "bash") {
        const command = output.args.command as string
        
        // 检查危险命令
        for (const pattern of dangerousPatterns) {
          if (pattern.test(command)) {
            throw new Error(`Dangerous command blocked: ${command}`)
          }
        }
        
        // 记录命令执行
        await client.app.log({
          body: {
            service: "tool-interceptor",
            level: "info",
            message: "Bash command executed",
            extra: { command, sessionID: input.sessionID }
          }
        })
      }
      
      // 拦截文件读取
      if (input.tool === "read") {
        const filePath = output.args.filePath as string
        
        // 检查敏感文件
        for (const sensitive of sensitiveFiles) {
          if (filePath.includes(sensitive)) {
            throw new Error(`Access to sensitive file blocked: ${filePath}`)
          }
        }
      }
      
      // 修改工具参数示例
      if (input.tool === "grep") {
        // 自动添加忽略大小写
        if (!output.args.flags) {
          output.args.flags = []
        }
        if (!output.args.flags.includes("-i")) {
          output.args.flags.push("-i")
        }
      }
    },

    // 工具执行后处理
    "tool.execute.after": async (input, output) => {
      // 修改输出结果
      if (input.tool === "bash") {
        // 为长输出添加摘要
        if (output.output.length > 1000) {
          output.title = "Command executed (output truncated)"
          output.metadata = {
            ...output.metadata,
            truncated: true,
            originalLength: output.output.length
          }
        }
      }
      
      // 添加自定义元数据
      output.metadata = {
        ...output.metadata,
        interceptedAt: new Date().toISOString(),
        interceptedBy: "tool-interceptor"
      }
    },

    // 修改工具定义
    "tool.definition": async (input, output) => {
      if (input.toolID === "bash") {
        // 修改 bash 工具的描述
        output.description = `${output.description}\n\n[Enhanced by Plugin] This tool is monitored for security.`
      }
    }
  }
}
```

### 5.4 环境变量注入插件

```typescript
// .opencode/plugins/env-injector.ts
import type { Plugin } from "@opencode-ai/plugin"

/**
 * 环境变量注入插件
 * 为所有 Shell 执行注入环境变量
 */
export const EnvInjectorPlugin: Plugin = async ({ project, worktree }) => {
  // 从安全位置加载敏感信息
  // 注意：不要硬编码敏感信息！
  const loadEnvFromSecureStorage = () => {
    // 示例：从系统 keychain 或加密文件读取
    return {
      API_KEY: process.env.OPENCODE_PLUGIN_API_KEY,
      PROJECT_ROOT: worktree
    }
  }

  const secureEnv = loadEnvFromSecureStorage()

  return {
    "shell.env": async (input, output) => {
      // 注入项目相关变量
      output.env.PROJECT_NAME = project.name || "unknown"
      output.env.PROJECT_ROOT = worktree
      output.env.WORKING_DIR = input.cwd
      
      // 注入自定义变量
      output.env.CUSTOM_VAR = "custom_value"
      
      // 注入安全变量（如果存在）
      if (secureEnv.API_KEY) {
        output.env.API_KEY = secureEnv.API_KEY
      }
      
      // 根据目录注入不同变量
      if (input.cwd.includes("/frontend")) {
        output.env.NODE_ENV = "development"
        output.env.FRONTEND_MODE = "true"
      }
      
      // 会话特定变量
      if (input.sessionID) {
        output.env.SESSION_ID = input.sessionID
      }
      
      console.log(`[EnvInjector] Injected env for: ${input.cwd}`)
    }
  }
}
```

### 5.5 会话压缩定制插件

```typescript
// .opencode/plugins/compaction-customizer.ts
import type { Plugin } from "@opencode-ai/plugin"

/**
 * 会话压缩定制插件
 * 自定义会话压缩时的提示词
 */
export const CompactionCustomizerPlugin: Plugin = async ({ worktree }) => {
  return {
    "experimental.session.compacting": async (input, output) => {
      // 方式1：追加上下文（推荐）
      output.context.push(`
## Project Context

Project Root: ${worktree}
Session ID: ${input.sessionID}
Compaction Time: ${new Date().toISOString()}

## Important State to Preserve

1. Current task status and progress
2. Files being actively modified
3. Recent architectural decisions
4. Pending TODO items
5. Test results and coverage info
`)

      // 方式2：完全替换压缩提示词（高级用法）
      // 注意：这会覆盖默认的压缩逻辑
      /*
      output.prompt = `
You are generating a continuation prompt for an AI coding assistant session.

The session has reached its context limit and needs to be compacted.

Summarize the following:
1. Current task objective and completion status
2. Key files modified and their purposes
3. Important decisions made during the session
4. Any blockers or issues encountered
5. Next steps to continue the work

Format as a structured prompt that a new agent can use to resume work seamlessly.

Original session ID: ${input.sessionID}
      `.trim()
      */
    }
  }
}
```

### 5.6 自定义工具插件

```typescript
// .opencode/plugins/custom-tools.ts
import type { Plugin } from "@opencode-ai/plugin"
import { tool } from "@opencode-ai/plugin"

/**
 * 自定义工具插件
 * 添加新的工具供 AI 使用
 */
export const CustomToolsPlugin: Plugin = async ({ project, worktree, $ }) => {
  return {
    tool: {
      // 项目信息工具
      getProjectInfo: tool({
        description: "Get detailed information about the current project",
        args: {},
        execute: async (_args, ctx) => {
          return JSON.stringify({
            name: project.name,
            worktree: ctx.worktree,
            directory: ctx.directory,
            sessionID: ctx.sessionID,
            agent: ctx.agent
          }, null, 2)
        }
      }),

      // 执行 Git 命令工具
      git: tool({
        description: "Execute git commands safely",
        args: {
          command: tool.schema.string().describe("Git command to execute (e.g., 'status', 'log --oneline')"),
          args: tool.schema.array(tool.schema.string()).optional().describe("Additional arguments")
        },
        execute: async (args, ctx) => {
          const { command, args: gitArgs = [] } = args
          
          // 只允许安全的 git 命令
          const allowedCommands = ["status", "log", "branch", "diff", "show"]
          const cmd = command.split(" ")[0]
          
          if (!allowedCommands.includes(cmd)) {
            return `Error: Git command '${cmd}' is not allowed for safety reasons`
          }
          
          try {
            const result = await $`git ${command} ${gitArgs}`.text()
            return result
          } catch (err: any) {
            return `Git error: ${err.message}`
          }
        }
      }),

      // 计算工具
      calculate: tool({
        description: "Perform mathematical calculations",
        args: {
          expression: tool.schema.string().describe("Mathematical expression to evaluate (e.g., '2 + 2', 'Math.sqrt(16)')")
        },
        execute: async (args) => {
          try {
            // 使用 Function 构造器安全地计算表达式
            const result = new Function(`return ${args.expression}`)()
            return `Result: ${result}`
          } catch (err: any) {
            return `Calculation error: ${err.message}`
          }
        }
      }),

      // 文件统计工具
      fileStats: tool({
        description: "Get statistics about a file or directory",
        args: {
          path: tool.schema.string().describe("Path to file or directory")
        },
        execute: async (args, ctx) => {
          const { path } = args
          const fullPath = path.startsWith("/") ? path : `${ctx.directory}/${path}`
          
          try {
            const file = Bun.file(fullPath)
            const exists = await file.exists()
            
            if (!exists) {
              return `Error: Path does not exist: ${path}`
            }
            
            const info = {
              path,
              size: file.size,
              type: file.type,
              lastModified: new Date(file.lastModified).toISOString()
            }
            
            return JSON.stringify(info, null, 2)
          } catch (err: any) {
            return `Error: ${err.message}`
          }
        }
      })
    }
  }
}
```

### 5.7 综合示例：智能通知插件

```typescript
// .opencode/plugins/smart-notifications.ts
import type { Plugin } from "@opencode-ai/plugin"

/**
 * 智能通知插件
 * 根据事件类型发送不同的系统通知
 */
export const SmartNotificationsPlugin: Plugin = async ({ client, $ }) => {
  // 通知配置
  const config = {
    enabled: true,
    notifyOnComplete: true,
    notifyOnError: true,
    notifyOnFileEdit: false,
    soundEnabled: process.platform === "darwin"
  }

  // 发送 macOS 通知
  const sendMacNotification = async (title: string, message: string) => {
    if (!config.enabled) return
    
    try {
      await $`osascript -e ${`display notification "${message}" with title "${title}"`}`
      
      if (config.soundEnabled) {
        await $`afplay /System/Library/Sounds/Glass.aiff`
      }
    } catch (err) {
      console.error("Failed to send notification:", err)
    }
  }

  // 发送日志通知
  const logNotification = async (level: string, title: string, message: string) => {
    await client.app.log({
      body: {
        service: "smart-notifications",
        level: level as any,
        message: title,
        extra: { notificationMessage: message }
      }
    })
  }

  return {
    event: async ({ event }) => {
      switch (event.type) {
        case "session.idle": {
          // 会话完成通知
          if (config.notifyOnComplete) {
            await sendMacNotification(
              "OpenCode",
              "Session completed successfully!"
            )
            await logNotification("info", "Session Complete", "Session finished")
          }
          break
        }

        case "session.error": {
          // 错误通知
          if (config.notifyOnError) {
            const errorMsg = event.properties.error?.data?.message || "Unknown error"
            await sendMacNotification(
              "OpenCode Error",
              `Session error: ${errorMsg.substring(0, 100)}`
            )
            await logNotification("error", "Session Error", errorMsg)
          }
          break
        }

        case "file.edited": {
          // 文件编辑通知（可选）
          if (config.notifyOnFileEdit) {
            await logNotification(
              "debug",
              "File Edited",
              `File: ${event.properties.file}`
            )
          }
          break
        }

        case "permission.asked": {
          // 权限请求通知
          await sendMacNotification(
            "OpenCode",
            `Permission required: ${event.properties.permission}`
          )
          break
        }

        case "installation.update-available": {
          // 更新可用通知
          await sendMacNotification(
            "OpenCode",
            `Update available: ${event.properties.version}`
          )
          break
        }
      }
    }
  }
}
```

### 5.8 权限管理插件

```typescript
// .opencode/plugins/permission-manager.ts
import type { Plugin } from "@opencode-ai/plugin"

/**
 * 权限管理插件
 * 实现自定义权限策略
 */
export const PermissionManagerPlugin: Plugin = async () => {
  // 权限规则配置
  const rules = {
    // 自动允许的权限
    autoAllow: [
      { permission: "read", pattern: "*.md" },
      { permission: "read", pattern: "*.ts" },
      { permission: "read", pattern: "*.js" },
      { permission: "list", pattern: "*" }
    ],
    
    // 自动拒绝的权限
    autoDeny: [
      { permission: "edit", pattern: "*.env*" },
      { permission: "edit", pattern: "*secret*" },
      { permission: "bash", pattern: "*rm -rf*" },
      { permission: "bash", pattern: "*sudo*" }
    ],
    
    // 需要询问的权限（白名单外）
    askPatterns: [
      { permission: "edit", pattern: "*.json" },
      { permission: "edit", pattern: "*.yaml" },
      { permission: "edit", pattern: "*.yml" }
    ]
  }

  // 匹配规则
  const matchesRule = (
    permission: string,
    patterns: string[],
    ruleList: Array<{ permission: string; pattern: string }>
  ): boolean => {
    return ruleList.some(rule => {
      if (rule.permission !== permission && rule.permission !== "*") {
        return false
      }
      
      // 简单的 glob 匹配
      const regex = new RegExp(
        "^" + rule.pattern.replace(/\*/g, ".*").replace(/\?/g, ".") + "$"
      )
      
      return patterns.some(pattern => regex.test(pattern))
    })
  }

  return {
    "permission.ask": async (input, output) => {
      const { permission, patterns } = input
      
      console.log(`[PermissionManager] Checking: ${permission} for ${patterns.join(", ")}`)
      
      // 检查自动拒绝
      if (matchesRule(permission, patterns, rules.autoDeny)) {
        console.log(`[PermissionManager] Auto-denied: ${permission}`)
        output.status = "deny"
        return
      }
      
      // 检查自动允许
      if (matchesRule(permission, patterns, rules.autoAllow)) {
        console.log(`[PermissionManager] Auto-allowed: ${permission}`)
        output.status = "allow"
        return
      }
      
      // 其他情况询问用户
      console.log(`[PermissionManager] Asking user: ${permission}`)
      output.status = "ask"
    }
  }
}
```

---

## 六、最佳实践

### 6.1 插件开发规范

#### 1. 错误处理
```typescript
export const SafePlugin: Plugin = async () => {
  return {
    "tool.execute.before": async (input, output) => {
      try {
        // 你的逻辑
        if (someCondition) {
          throw new Error("Invalid parameter")
        }
      } catch (err) {
        // 记录错误但不要让插件崩溃
        console.error("Plugin error:", err)
        // 可以选择抛出阻止操作
        // throw err
      }
    }
  }
}
```

#### 2. 异步操作
```typescript
export const AsyncPlugin: Plugin = async ({ client }) => {
  // 并行初始化
  const [config, data] = await Promise.all([
    loadConfig(),
    fetchData()
  ])
  
  return {
    event: async ({ event }) => {
      // 异步处理事件
      if (event.type === "session.idle") {
        // 非阻塞的后处理
        processAsync(event.properties.sessionID).catch(console.error)
      }
    }
  }
}
```

#### 3. 配置管理
```typescript
// 使用 package.json 管理依赖
// .opencode/package.json
{
  "dependencies": {
    "lodash": "^4.17.21",
    "date-fns": "^2.30.0"
  }
}
```

### 6.2 性能优化

1. **避免在 `message.part.delta` 中执行耗时操作** - 这是高频事件
2. **使用缓存** - 对于重复计算的结果
3. **延迟加载** - 只在需要时加载资源
4. **批量处理** - 聚合多个事件后再处理

### 6.3 安全考虑

1. **不要硬编码敏感信息** - 使用环境变量或安全存储
2. **验证所有输入** - 在 `tool.execute.before` 中检查参数
3. **限制文件访问** - 阻止对敏感文件的访问
4. **审计日志** - 记录重要的安全事件

### 6.4 调试技巧

```typescript
// 启用详细日志
export const DebugPlugin: Plugin = async ({ client }) => {
  const DEBUG = process.env.DEBUG_PLUGIN === "true"
  
  const log = (message: string, data?: any) => {
    if (DEBUG) {
      console.log(`[DebugPlugin] ${message}`, data || "")
    }
  }
  
  return {
    event: async ({ event }) => {
      log(`Event received: ${event.type}`, event.properties)
    },
    
    "tool.execute.before": async (input) => {
      log(`Tool called: ${input.tool}`, { sessionID: input.sessionID })
    }
  }
}
```

### 6.5 测试插件

```typescript
// 简单的测试脚本
// test-plugin.ts
import { MyPlugin } from "./.opencode/plugins/my-plugin"

const mockContext = {
  client: {
    app: {
      log: async (data: any) => console.log("Log:", data)
    }
  },
  project: { name: "test-project", worktree: "/tmp/test" },
  directory: "/tmp/test",
  worktree: "/tmp/test",
  serverUrl: new URL("http://localhost:4096"),
  $: (strings: any, ...values: any[]) => {
    console.log("Shell:", strings, values)
    return { text: async () => "output" }
  }
}

async function test() {
  const hooks = await MyPlugin(mockContext as any)
  
  // 测试 event hook
  if (hooks.event) {
    await hooks.event({
      event: { type: "session.idle", properties: { sessionID: "test-123" } } as any
    })
  }
  
  console.log("Test completed!")
}

test().catch(console.error)
```

---

## 七、参考资源

### 7.1 官方文档
- [OpenCode 插件文档](https://opencode.ai/docs/plugins)
- [OpenCode SDK 文档](https://opencode.ai/docs/sdk)
- [Bun Shell 文档](https://bun.sh/docs/runtime/shell)

### 7.2 源码位置
```
packages/plugin/src/
├── index.ts      # 核心类型定义 (Hooks, Plugin)
├── tool.ts       # 工具定义 (tool, ToolContext)
├── tui.ts        # TUI 插件 API
└── shell.ts      # Bun Shell 类型

packages/sdk/js/src/v2/gen/types.gen.ts  # 事件类型定义
```

### 7.3 社区资源
- [插件示例仓库](https://opencode.ai/docs/ecosystem#plugins)
- [Discord 社区](https://discord.gg/opencode)

---

## 八、快速参考卡片

### 8.1 常用 Hook 速查

```typescript
// 监听所有事件
{ event: async ({ event }) => { } }

// 修改 LLM 参数
{ "chat.params": async (input, output) => { output.temperature = 0.5 } }

// 拦截工具调用
{ "tool.execute.before": async (input, output) => { } }

// 注入环境变量
{ "shell.env": async (input, output) => { output.env.KEY = "value" } }

// 自定义压缩提示
{ "experimental.session.compacting": async (input, output) => { output.context.push("...") } }

// 注册自定义工具
{ tool: { myTool: tool({ ... }) } }
```

### 8.2 常用事件类型速查

```typescript
// 会话生命周期
"session.created" | "session.idle" | "session.error"

// 消息相关
"message.updated" | "message.part.delta" | "message.part.updated"

// 文件相关
"file.edited" | "file.watcher.updated"

// 权限相关
"permission.asked" | "permission.replied"

// 工具相关
"command.executed" | "todo.updated"
```

---

**文档结束**

如有问题，请参考 OpenCode 官方文档或在社区寻求帮助。
