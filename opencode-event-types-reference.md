# OpenCode 事件类型消息体速查参考

> 快速查阅每个事件的完整消息结构  
> 基于 OpenCode SDK v2 类型定义  
> 生成日期: 2026-04-01

---

## 目录

- [会话事件 (Session Events)](#一会话事件-session-events)
- [消息事件 (Message Events)](#二消息事件-message-events)
- [文件事件 (File Events)](#三文件事件-file-events)
- [权限事件 (Permission Events)](#四权限事件-permission-events)
- [工具/命令事件](#五工具命令事件)
- [LSP 事件](#六lsp-事件)
- [MCP 事件](#七mcp-事件)
- [PTY 终端事件](#八pty-终端事件)
- [工作空间事件](#九工作空间事件)
- [TUI 事件](#十tui-事件)
- [问答事件](#十一问答事件)
- [项目/安装事件](#十二项目安装事件)
- [错误类型](#十三错误类型)
- [Hook 输入输出类型](#十四hook-输入输出类型)

---

## 一、会话事件 (Session Events)

### 1. `session.created`
**触发时机**: 新会话创建成功

```typescript
{
  type: "session.created"
  properties: {
    sessionID: string
    info: Session
  }
}

// Session 对象结构
interface Session {
  id: string
  slug: string
  projectID: string
  workspaceID?: string
  directory: string
  parentID?: string
  summary?: {
    additions: number
    deletions: number
    files: number
    diffs?: FileDiff[]
  }
  share?: {
    url: string
  }
  title: string
  version: string
  time: {
    created: number
    updated: number
    compacting?: number
    archived?: number
  }
  permission?: PermissionRuleset
  revert?: {
    messageID: string
    partID?: string
    snapshot?: string
    diff?: string
  }
}
```

**示例**:
```json
{
  "type": "session.created",
  "properties": {
    "sessionID": "sess_abc123",
    "info": {
      "id": "sess_abc123",
      "slug": "feature-auth-implementation",
      "projectID": "proj_xyz789",
      "workspaceID": "ws_dev",
      "directory": "/home/user/projects/myapp",
      "title": "Feature: Auth Implementation",
      "version": "1.0.0",
      "time": {
        "created": 1704067200000,
        "updated": 1704067200000
      }
    }
  }
}
```

---

### 2. `session.updated`
**触发时机**: 会话信息更新（标题、权限等）

```typescript
{
  type: "session.updated"
  properties: {
    sessionID: string
    info: Session  // 同 session.created
  }
}
```

---

### 3. `session.deleted`
**触发时机**: 会话被删除

```typescript
{
  type: "session.deleted"
  properties: {
    sessionID: string
    info: Session  // 会话删除前的快照
  }
}
```

---

### 4. `session.status`
**触发时机**: 会话状态变化

```typescript
{
  type: "session.status"
  properties: {
    sessionID: string
    status: SessionStatus
  }
}

// SessionStatus 可以是以下三种之一:
type SessionStatus =
  | { type: "idle" }                           // 空闲状态
  | { type: "busy" }                           // 忙碌状态
  | { 
      type: "retry"
      attempt: number      // 重试次数
      message: string      // 错误信息
      next: number         // 下次重试时间戳
    }
```

**示例**:
```json
// 空闲状态
{
  "type": "session.status",
  "properties": {
    "sessionID": "sess_abc123",
    "status": { "type": "idle" }
  }
}

// 重试状态
{
  "type": "session.status",
  "properties": {
    "sessionID": "sess_abc123",
    "status": {
      "type": "retry",
      "attempt": 2,
      "message": "Rate limit exceeded",
      "next": 1704067260000
    }
  }
}
```

---

### 5. `session.idle`
**触发时机**: 会话完成所有工作，进入空闲状态

```typescript
{
  type: "session.idle"
  properties: {
    sessionID: string
  }
}
```

**示例**:
```json
{
  "type": "session.idle",
  "properties": {
    "sessionID": "sess_abc123"
  }
}
```

---

### 6. `session.compacted`
**触发时机**: 会话压缩完成

```typescript
{
  type: "session.compacted"
  properties: {
    sessionID: string
  }
}
```

---

### 7. `session.diff`
**触发时机**: 会话文件差异更新

```typescript
{
  type: "session.diff"
  properties: {
    sessionID: string
    diff: FileDiff[]
  }
}

// FileDiff 结构
interface FileDiff {
  file: string
  before: string
  after: string
  additions: number
  deletions: number
  status?: "added" | "deleted" | "modified"
}
```

**示例**:
```json
{
  "type": "session.diff",
  "properties": {
    "sessionID": "sess_abc123",
    "diff": [
      {
        "file": "src/auth.ts",
        "before": "export function login() {}",
        "after": "export function login() { return true; }",
        "additions": 1,
        "deletions": 0,
        "status": "modified"
      }
    ]
  }
}
```

---

### 8. `session.error`
**触发时机**: 会话发生错误

```typescript
{
  type: "session.error"
  properties: {
    sessionID?: string
    error?: 
      | ProviderAuthError
      | UnknownError
      | MessageOutputLengthError
      | MessageAbortedError
      | StructuredOutputError
      | ContextOverflowError
      | ApiError
  }
}
```

**示例**:
```json
{
  "type": "session.error",
  "properties": {
    "sessionID": "sess_abc123",
    "error": {
      "name": "ApiError",
      "data": {
        "message": "Rate limit exceeded",
        "statusCode": 429,
        "isRetryable": true
      }
    }
  }
}
```

---

## 二、消息事件 (Message Events)

### 1. `message.updated`
**触发时机**: 消息更新

```typescript
{
  type: "message.updated"
  properties: {
    sessionID: string
    info: Message
  }
}

// Message 可以是 UserMessage 或 AssistantMessage
type Message = UserMessage | AssistantMessage

interface UserMessage {
  id: string
  sessionID: string
  role: "user"
  time: { created: number }
  format?: OutputFormat
  summary?: {
    title?: string
    body?: string
    diffs: FileDiff[]
  }
  agent: string
  model: {
    providerID: string
    modelID: string
  }
  system?: string
  tools?: { [key: string]: boolean }
  variant?: string
}

interface AssistantMessage {
  id: string
  sessionID: string
  role: "assistant"
  time: { created: number; completed?: number }
  error?: ErrorTypes
  parentID: string
  modelID: string
  providerID: string
  mode: string
  agent: string
  path: { cwd: string; root: string }
  summary?: boolean
  cost: number
  tokens: {
    total?: number
    input: number
    output: number
    reasoning: number
    cache: { read: number; write: number }
  }
  structured?: unknown
  variant?: string
  finish?: string
}
```

---

### 2. `message.removed`
**触发时机**: 消息删除

```typescript
{
  type: "message.removed"
  properties: {
    sessionID: string
    messageID: string
  }
}
```

---

### 3. `message.part.updated`
**触发时机**: 消息部分内容更新

```typescript
{
  type: "message.part.updated"
  properties: {
    sessionID: string
    part: Part
    time: number
  }
}

// Part 可以是以下类型之一
type Part =
  | TextPart
  | SubtaskPart
  | ReasoningPart
  | FilePart
  | ToolPart
  | StepStartPart
  | StepFinishPart
  | SnapshotPart
  | PatchPart
  | AgentPart
  | RetryPart
  | CompactionPart

// 文本部分
interface TextPart {
  id: string
  sessionID: string
  messageID: string
  type: "text"
  text: string
  synthetic?: boolean
  ignored?: boolean
  time?: { start: number; end?: number }
  metadata?: { [key: string]: unknown }
}

// 推理部分
interface ReasoningPart {
  id: string
  sessionID: string
  messageID: string
  type: "reasoning"
  text: string
  metadata?: { [key: string]: unknown }
  time: { start: number; end?: number }
}

// 工具调用部分
interface ToolPart {
  id: string
  sessionID: string
  messageID: string
  type: "tool"
  callID: string
  tool: string
  state: ToolState
  metadata?: { [key: string]: unknown }
}

// 工具状态
type ToolState =
  | { status: "pending"; input: object; raw: string }
  | { 
      status: "running"
      input: object
      title?: string
      metadata?: object
      time: { start: number }
    }
  | {
      status: "completed"
      input: object
      output: string
      title: string
      metadata: object
      time: { start: number; end: number; compacted?: number }
      attachments?: FilePart[]
    }
  | {
      status: "error"
      input: object
      error: string
      metadata?: object
      time: { start: number; end: number }
    }
```

**示例**:
```json
{
  "type": "message.part.updated",
  "properties": {
    "sessionID": "sess_abc123",
    "part": {
      "id": "part_xyz789",
      "sessionID": "sess_abc123",
      "messageID": "msg_def456",
      "type": "tool",
      "callID": "call_ghi789",
      "tool": "read",
      "state": {
        "status": "completed",
        "input": { "filePath": "/home/user/projects/myapp/src/auth.ts" },
        "output": "export function login() { ... }",
        "title": "Read file: src/auth.ts",
        "metadata": {},
        "time": { "start": 1704067200000, "end": 1704067200100 }
      }
    },
    "time": 1704067200100
  }
}
```

---

### 4. `message.part.removed`
**触发时机**: 消息部分内容删除

```typescript
{
  type: "message.part.removed"
  properties: {
    sessionID: string
    messageID: string
    partID: string
  }
}
```

---

### 5. `message.part.delta`
**触发时机**: 消息内容增量更新（流式响应）

```typescript
{
  type: "message.part.delta"
  properties: {
    sessionID: string
    messageID: string
    partID: string
    field: string      // 更新的字段名，如 "text"
    delta: string      // 增量内容
  }
}
```

**示例**:
```json
{
  "type": "message.part.delta",
  "properties": {
    "sessionID": "sess_abc123",
    "messageID": "msg_def456",
    "partID": "part_xyz789",
    "field": "text",
    "delta": "Hello, "
  }
}
```

---

## 三、文件事件 (File Events)

### 1. `file.edited`
**触发时机**: 文件被 AI 编辑

```typescript
{
  type: "file.edited"
  properties: {
    file: string  // 文件路径
  }
}
```

**示例**:
```json
{
  "type": "file.edited",
  "properties": {
    "file": "/home/user/projects/myapp/src/auth.ts"
  }
}
```

---

### 2. `file.watcher.updated`
**触发时机**: 文件监控检测到变化

```typescript
{
  type: "file.watcher.updated"
  properties: {
    file: string
    event: "add" | "change" | "unlink"
  }
}
```

**示例**:
```json
{
  "type": "file.watcher.updated",
  "properties": {
    "file": "/home/user/projects/myapp/src/auth.ts",
    "event": "change"
  }
}
```

---

## 四、权限事件 (Permission Events)

### 1. `permission.asked`
**触发时机**: 请求用户权限

```typescript
{
  type: "permission.asked"
  properties: PermissionRequest
}

interface PermissionRequest {
  id: string
  sessionID: string
  permission: string      // 权限类型: read, edit, bash, etc.
  patterns: string[]      // 请求的文件/路径模式
  metadata: { [key: string]: unknown }
  always: string[]        // 用户已允许的模式
  tool?: {
    messageID: string
    callID: string
  }
}
```

**示例**:
```json
{
  "type": "permission.asked",
  "properties": {
    "id": "perm_abc123",
    "sessionID": "sess_xyz789",
    "permission": "edit",
    "patterns": ["src/auth.ts"],
    "metadata": {
      "tool": "edit_file",
      "description": "Modify authentication logic"
    },
    "always": ["*.md"],
    "tool": {
      "messageID": "msg_def456",
      "callID": "call_ghi789"
    }
  }
}
```

---

### 2. `permission.replied`
**触发时机**: 用户回复权限请求

```typescript
{
  type: "permission.replied"
  properties: {
    sessionID: string
    requestID: string
    reply: "once" | "always" | "reject"
  }
}
```

**示例**:
```json
{
  "type": "permission.replied",
  "properties": {
    "sessionID": "sess_xyz789",
    "requestID": "perm_abc123",
    "reply": "once"
  }
}
```

---

## 五、工具/命令事件

### 1. `command.executed`
**触发时机**: 斜杠命令被执行

```typescript
{
  type: "command.executed"
  properties: {
    name: string        // 命令名称
    sessionID: string
    arguments: string   // 命令参数
    messageID: string
  }
}
```

**示例**:
```json
{
  "type": "command.executed",
  "properties": {
    "name": "commit",
    "sessionID": "sess_abc123",
    "arguments": "-m \"Update auth logic\"",
    "messageID": "msg_def456"
  }
}
```

---

### 2. `todo.updated`
**触发时机**: 待办事项更新

```typescript
{
  type: "todo.updated"
  properties: {
    sessionID: string
    todos: Todo[]
  }
}

interface Todo {
  content: string
  status: "pending" | "in_progress" | "completed" | "cancelled"
  priority: "high" | "medium" | "low"
}
```

**示例**:
```json
{
  "type": "todo.updated",
  "properties": {
    "sessionID": "sess_abc123",
    "todos": [
      {
        "content": "Implement JWT authentication",
        "status": "in_progress",
        "priority": "high"
      },
      {
        "content": "Add password validation",
        "status": "pending",
        "priority": "medium"
      }
    ]
  }
}
```

---

## 六、LSP 事件

### 1. `lsp.updated`
**触发时机**: LSP 服务器状态更新

```typescript
{
  type: "lsp.updated"
  properties: {
    [key: string]: unknown
  }
}
```

---

### 2. `lsp.client.diagnostics`
**触发时机**: LSP 诊断信息更新

```typescript
{
  type: "lsp.client.diagnostics"
  properties: {
    serverID: string
    path: string
  }
}
```

**示例**:
```json
{
  "type": "lsp.client.diagnostics",
  "properties": {
    "serverID": "typescript-language-server",
    "path": "/home/user/projects/myapp/src/auth.ts"
  }
}
```

---

## 七、MCP 事件

### 1. `mcp.tools.changed`
**触发时机**: MCP 工具列表变化

```typescript
{
  type: "mcp.tools.changed"
  properties: {
    server: string  // MCP 服务器名称
  }
}
```

**示例**:
```json
{
  "type": "mcp.tools.changed",
  "properties": {
    "server": "filesystem-mcp"
  }
}
```

---

### 2. `mcp.browser.open.failed`
**触发时机**: MCP 浏览器打开失败

```typescript
{
  type: "mcp.browser.open.failed"
  properties: {
    mcpName: string
    url: string
  }
}
```

---

## 八、PTY 终端事件

### 1. `pty.created`
**触发时机**: PTY 终端创建

```typescript
{
  type: "pty.created"
  properties: {
    info: Pty
  }
}

interface Pty {
  id: string
  title: string
  command: string
  args: string[]
  cwd: string
  status: "running" | "exited"
  pid: number
}
```

**示例**:
```json
{
  "type": "pty.created",
  "properties": {
    "info": {
      "id": "pty_abc123",
      "title": "bash",
      "command": "/bin/bash",
      "args": [],
      "cwd": "/home/user/projects/myapp",
      "status": "running",
      "pid": 12345
    }
  }
}
```

---

### 2. `pty.updated`
**触发时机**: PTY 终端更新

```typescript
{
  type: "pty.updated"
  properties: {
    info: Pty
  }
}
```

---

### 3. `pty.exited`
**触发时机**: PTY 终端退出

```typescript
{
  type: "pty.exited"
  properties: {
    id: string
    exitCode: number
  }
}
```

**示例**:
```json
{
  "type": "pty.exited",
  "properties": {
    "id": "pty_abc123",
    "exitCode": 0
  }
}
```

---

### 4. `pty.deleted`
**触发时机**: PTY 终端删除

```typescript
{
  type: "pty.deleted"
  properties: {
    id: string
  }
}
```

---

## 九、工作空间事件

### 1. `workspace.ready`
**触发时机**: 工作空间就绪

```typescript
{
  type: "workspace.ready"
  properties: {
    name: string
  }
}
```

---

### 2. `workspace.failed`
**触发时机**: 工作空间初始化失败

```typescript
{
  type: "workspace.failed"
  properties: {
    message: string
  }
}
```

---

### 3. `worktree.ready`
**触发时机**: Git worktree 就绪

```typescript
{
  type: "worktree.ready"
  properties: {
    name: string
    branch: string
  }
}
```

**示例**:
```json
{
  "type": "worktree.ready",
  "properties": {
    "name": "feature/auth",
    "branch": "feature/auth"
  }
}
```

---

### 4. `worktree.failed`
**触发时机**: Git worktree 初始化失败

```typescript
{
  type: "worktree.failed"
  properties: {
    message: string
  }
}
```

---

### 5. `vcs.branch.updated`
**触发时机**: Git 分支变化

```typescript
{
  type: "vcs.branch.updated"
  properties: {
    branch?: string
  }
}
```

**示例**:
```json
{
  "type": "vcs.branch.updated",
  "properties": {
    "branch": "main"
  }
}
```

---

## 十、TUI 事件

### 1. `tui.prompt.append`
**触发时机**: TUI 提示框追加文本

```typescript
{
  type: "tui.prompt.append"
  properties: {
    text: string
  }
}
```

---

### 2. `tui.command.execute`
**触发时机**: TUI 执行命令

```typescript
{
  type: "tui.command.execute"
  properties: {
    command: 
      | "session.list"
      | "session.new"
      | "session.share"
      | "session.interrupt"
      | "session.compact"
      | "session.page.up"
      | "session.page.down"
      | "session.line.up"
      | "session.line.down"
      | "session.half.page.up"
      | "session.half.page.down"
      | "session.first"
      | "session.last"
      | "prompt.clear"
      | "prompt.submit"
      | "agent.cycle"
      | string
  }
}
```

---

### 3. `tui.toast.show`
**触发时机**: TUI 显示通知

```typescript
{
  type: "tui.toast.show"
  properties: {
    title?: string
    message: string
    variant: "info" | "success" | "warning" | "error"
    duration?: number  // 毫秒
  }
}
```

**示例**:
```json
{
  "type": "tui.toast.show",
  "properties": {
    "title": "Success",
    "message": "File saved successfully",
    "variant": "success",
    "duration": 3000
  }
}
```

---

### 4. `tui.session.select`
**触发时机**: TUI 选择会话

```typescript
{
  type: "tui.session.select"
  properties: {
    sessionID: string
  }
}
```

---

## 十一、问答事件

### 1. `question.asked`
**触发时机**: 向用户提问

```typescript
{
  type: "question.asked"
  properties: QuestionRequest
}

interface QuestionRequest {
  id: string
  sessionID: string
  questions: QuestionInfo[]
  tool?: {
    messageID: string
    callID: string
  }
}

interface QuestionInfo {
  question: string
  header: string
  options: QuestionOption[]
  multiple?: boolean
  custom?: boolean
}

interface QuestionOption {
  label: string
  description: string
}
```

**示例**:
```json
{
  "type": "question.asked",
  "properties": {
    "id": "qst_abc123",
    "sessionID": "sess_xyz789",
    "questions": [
      {
        "question": "Which authentication method would you like to use?",
        "header": "Auth Method",
        "options": [
          { "label": "JWT", "description": "JSON Web Tokens" },
          { "label": "Session", "description": "Server-side sessions" },
          { "label": "OAuth", "description": "OAuth 2.0" }
        ],
        "multiple": false,
        "custom": true
      }
    ]
  }
}
```

---

### 2. `question.replied`
**触发时机**: 用户回答问题

```typescript
{
  type: "question.replied"
  properties: {
    sessionID: string
    requestID: string
    answers: string[][]  // 每个问题的答案数组
  }
}
```

**示例**:
```json
{
  "type": "question.replied",
  "properties": {
    "sessionID": "sess_xyz789",
    "requestID": "qst_abc123",
    "answers": [["JWT"]]
  }
}
```

---

### 3. `question.rejected`
**触发时机**: 用户拒绝回答

```typescript
{
  type: "question.rejected"
  properties: {
    sessionID: string
    requestID: string
  }
}
```

---

## 十二、项目/安装事件

### 1. `project.updated`
**触发时机**: 项目配置更新

```typescript
{
  type: "project.updated"
  properties: Project
}

interface Project {
  id: string
  worktree: string
  vcs?: "git"
  name?: string
  icon?: {
    url?: string
    override?: string
    color?: string
  }
  commands?: {
    start?: string
  }
  time: {
    created: number
    updated: number
    initialized?: number
  }
  sandboxes: string[]
}
```

---

### 2. `installation.updated`
**触发时机**: OpenCode 安装更新

```typescript
{
  type: "installation.updated"
  properties: {
    version: string
  }
}
```

---

### 3. `installation.update-available`
**触发时机**: 有新版本可用

```typescript
{
  type: "installation.update-available"
  properties: {
    version: string
  }
}
```

---

### 4. `server.instance.disposed`
**触发时机**: 服务器实例释放

```typescript
{
  type: "server.instance.disposed"
  properties: {
    directory: string
  }
}
```

---

### 5. `server.connected`
**触发时机**: 服务器连接成功

```typescript
{
  type: "server.connected"
  properties: {
    [key: string]: unknown
  }
}
```

---

### 6. `global.disposed`
**触发时机**: 全局资源释放

```typescript
{
  type: "global.disposed"
  properties: {
    [key: string]: unknown
  }
}
```

---

## 十三、错误类型

### 错误类型定义

```typescript
type ErrorTypes =
  | ProviderAuthError
  | UnknownError
  | MessageOutputLengthError
  | MessageAbortedError
  | StructuredOutputError
  | ContextOverflowError
  | ApiError

// 提供商认证错误
interface ProviderAuthError {
  name: "ProviderAuthError"
  data: {
    providerID: string
    message: string
  }
}

// 未知错误
interface UnknownError {
  name: "UnknownError"
  data: {
    message: string
  }
}

// 消息输出长度错误
interface MessageOutputLengthError {
  name: "MessageOutputLengthError"
  data: { [key: string]: unknown }
}

// 消息中止错误
interface MessageAbortedError {
  name: "MessageAbortedError"
  data: {
    message: string
  }
}

// 结构化输出错误
interface StructuredOutputError {
  name: "StructuredOutputError"
  data: {
    message: string
    retries: number
  }
}

// 上下文溢出错误
interface ContextOverflowError {
  name: "ContextOverflowError"
  data: {
    message: string
    responseBody?: string
  }
}

// API 错误
interface ApiError {
  name: "APIError"
  data: {
    message: string
    statusCode?: number
    isRetryable: boolean
    responseHeaders?: { [key: string]: string }
    responseBody?: string
    metadata?: { [key: string]: string }
  }
}
```

---

## 十四、Hook 输入输出类型

### 1. `chat.message` Hook

**输入**:
```typescript
{
  sessionID: string
  agent?: string
  model?: { providerID: string; modelID: string }
  messageID?: string
  variant?: string
}
```

**输出**:
```typescript
{
  message: UserMessage
  parts: Part[]
}
```

---

### 2. `chat.params` Hook

**输入**:
```typescript
{
  sessionID: string
  agent: string
  model: Model
  provider: ProviderContext
  message: UserMessage
}
```

**输出**:
```typescript
{
  temperature: number
  topP: number
  topK: number
  options: Record<string, any>
}
```

---

### 3. `chat.headers` Hook

**输入**: 同 `chat.params`

**输出**:
```typescript
{
  headers: Record<string, string>
}
```

---

### 4. `tool.execute.before` Hook

**输入**:
```typescript
{
  tool: string
  sessionID: string
  callID: string
}
```

**输出**:
```typescript
{
  args: any
}
```

---

### 5. `tool.execute.after` Hook

**输入**:
```typescript
{
  tool: string
  sessionID: string
  callID: string
  args: any
}
```

**输出**:
```typescript
{
  title: string
  output: string
  metadata: any
}
```

---

### 6. `shell.env` Hook

**输入**:
```typescript
{
  cwd: string
  sessionID?: string
  callID?: string
}
```

**输出**:
```typescript
{
  env: Record<string, string>
}
```

---

### 7. `permission.ask` Hook

**输入**: `Permission` 对象

**输出**:
```typescript
{
  status: "ask" | "deny" | "allow"
}
```

---

### 8. `command.execute.before` Hook

**输入**:
```typescript
{
  command: string
  sessionID: string
  arguments: string
}
```

**输出**:
```typescript
{
  parts: Part[]
}
```

---

### 9. `experimental.session.compacting` Hook

**输入**:
```typescript
{
  sessionID: string
}
```

**输出**:
```typescript
{
  context: string[]
  prompt?: string  // 如果设置，替换默认提示词
}
```

---

### 10. `tool.definition` Hook

**输入**:
```typescript
{
  toolID: string
}
```

**输出**:
```typescript
{
  description: string
  parameters: any
}
```

---

## 十五、快速索引

### 按类别索引

| 类别 | 事件 |
|-----|------|
| **会话** | session.created, session.updated, session.deleted, session.status, session.idle, session.compacted, session.diff, session.error |
| **消息** | message.updated, message.removed, message.part.updated, message.part.removed, message.part.delta |
| **文件** | file.edited, file.watcher.updated |
| **权限** | permission.asked, permission.replied |
| **工具** | command.executed, todo.updated |
| **LSP** | lsp.updated, lsp.client.diagnostics |
| **MCP** | mcp.tools.changed, mcp.browser.open.failed |
| **PTY** | pty.created, pty.updated, pty.exited, pty.deleted |
| **工作空间** | workspace.ready, workspace.failed, worktree.ready, worktree.failed, vcs.branch.updated |
| **TUI** | tui.prompt.append, tui.command.execute, tui.toast.show, tui.session.select |
| **问答** | question.asked, question.replied, question.rejected |
| **项目** | project.updated, installation.updated, installation.update-available, server.instance.disposed, server.connected, global.disposed |

### 常用字段说明

| 字段 | 类型 | 说明 |
|-----|------|------|
| `sessionID` | string | 会话唯一标识 |
| `messageID` | string | 消息唯一标识 |
| `partID` | string | 消息部分唯一标识 |
| `callID` | string | 工具调用唯一标识 |
| `timestamp` | number | 时间戳（毫秒） |
| `status` | string | 状态：idle, busy, retry, pending, running, completed, error |

---

**文档结束**

这份速查手册涵盖了 OpenCode 所有 46 种事件类型的完整消息体结构，方便您在开发插件时快速查阅。
