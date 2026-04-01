# OpenCode SDK 速查表

## 安装

```bash
npm install @opencode-ai/sdk
```

## 快速开始

### 创建客户端

```typescript
import { createOpencodeClient } from '@opencode-ai/sdk'

const client = createOpencodeClient({
  baseUrl: 'http://127.0.0.1:4096',
  directory: '/path/to/project'
})
```

### 启动本地服务器

```typescript
import { createOpencode } from '@opencode-ai/sdk'

const { client, server } = await createOpencode({
  port: 4096,
  config: { logLevel: 'INFO' }
})

server.close()
```

## 使用示例

### 创建会话并发送消息

```typescript
const session = await client.session.create({ title: 'My Session' })

const response = await client.session.prompt({
  sessionID: session.id,
  parts: [{ type: 'text', text: 'Hello, OpenCode!' }]
})
```

### 订阅事件

```typescript
const eventStream = await client.event.subscribe()
for await (const event of eventStream) {
  console.log('Received event:', event)
}
```

### 搜索文件

```typescript
const results = await client.find.text({ pattern: 'function.*create' })
```

## 方法速查

### 全局操作 (Global)

| 方法名 | 功能 |
|--------|------|
| `client.global.health()` | 获取服务器健康状态 |
| `client.global.event()` | 订阅全局事件流 (SSE) |
| `client.global.dispose()` | 释放所有实例 |
| `client.global.upgrade({ target })` | 升级 OpenCode |
| `client.global.config.get()` | 获取全局配置 |
| `client.global.config.update({ config })` | 更新全局配置 |

### 认证管理 (Auth)

| 方法名 | 功能 |
|--------|------|
| `client.auth.set({ providerID, auth })` | 设置认证凭据 |
| `client.auth.remove({ providerID })` | 移除认证凭据 |

### 应用管理 (App)

| 方法名 | 功能 |
|--------|------|
| `client.app.log({ service, level, message })` | 写入日志 |
| `client.app.agents()` | 获取 Agent 列表 |
| `client.app.skills()` | 获取 Skill 列表 |

### 项目管理 (Project)

| 方法名 | 功能 |
|--------|------|
| `client.project.list()` | 获取项目列表 |
| `client.project.current()` | 获取当前项目 |
| `client.project.initGit()` | 初始化 Git 仓库 |
| `client.project.update({ projectID, name })` | 更新项目 |

### 终端会话 (PTY)

| 方法名 | 功能 |
|--------|------|
| `client.pty.list()` | 获取 PTY 列表 |
| `client.pty.create({ command, args, cwd })` | 创建 PTY |
| `client.pty.remove({ ptyID })` | 移除 PTY |
| `client.pty.get({ ptyID })` | 获取 PTY 详情 |
| `client.pty.update({ ptyID, title, size })` | 更新 PTY |
| `client.pty.connect({ ptyID })` | 连接 PTY |

### 配置管理 (Config)

| 方法名 | 功能 |
|--------|------|
| `client.config.get()` | 获取配置 |
| `client.config.update({ config })` | 更新配置 |
| `client.config.providers()` | 获取提供商列表 |

### 工具管理 (Tool)

| 方法名 | 功能 |
|--------|------|
| `client.tool.ids()` | 获取工具 ID 列表 |
| `client.tool.list({ provider, model })` | 获取工具列表 |

### 会话管理 (Session)

| 方法名 | 功能 |
|--------|------|
| `client.session.list()` | 获取会话列表 |
| `client.session.create({ title })` | 创建会话 |
| `client.session.status()` | 获取会话状态 |
| `client.session.delete({ sessionID })` | 删除会话 |
| `client.session.get({ sessionID })` | 获取会话详情 |
| `client.session.update({ sessionID, title })` | 更新会话 |
| `client.session.children({ sessionID })` | 获取子会话 |
| `client.session.todo({ sessionID })` | 获取待办事项 |
| `client.session.init({ sessionID })` | 初始化 AGENTS.md |
| `client.session.fork({ sessionID, messageID })` | 派生会话 |
| `client.session.abort({ sessionID })` | 中止会话 |
| `client.session.share({ sessionID })` | 共享会话 |
| `client.session.unshare({ sessionID })` | 取消共享 |
| `client.session.diff({ sessionID })` | 获取文件差异 |
| `client.session.summarize({ sessionID })` | 生成摘要 |
| `client.session.messages({ sessionID })` | 获取消息列表 |
| `client.session.prompt({ sessionID, parts })` | 发送消息 |
| `client.session.deleteMessage({ sessionID, messageID })` | 删除消息 |
| `client.session.message({ sessionID, messageID })` | 获取消息 |
| `client.session.promptAsync({ sessionID, parts })` | 异步发送 |
| `client.session.command({ sessionID, command })` | 发送命令 |
| `client.session.shell({ sessionID, command })` | 执行 Shell |
| `client.session.revert({ sessionID, messageID })` | 回退消息 |
| `client.session.unrevert({ sessionID })` | 恢复消息 |

### 消息部件 (Part)

| 方法名 | 功能 |
|--------|------|
| `client.part.delete({ sessionID, messageID, partID })` | 删除部件 |
| `client.part.update({ sessionID, messageID, partID, part })` | 更新部件 |

### 权限管理 (Permission)

| 方法名 | 功能 |
|--------|------|
| `client.permission.list()` | 获取权限请求列表 |
| `client.permission.reply({ requestID, reply })` | 响应权限请求 |

### 问题管理 (Question)

| 方法名 | 功能 |
|--------|------|
| `client.question.list()` | 获取问题列表 |
| `client.question.reply({ requestID, answers })` | 回复问题 |
| `client.question.reject({ requestID })` | 拒绝问题 |

### 提供商管理 (Provider)

| 方法名 | 功能 |
|--------|------|
| `client.provider.list()` | 获取提供商列表 |
| `client.provider.auth()` | 获取认证方法 |
| `client.provider.oauth.authorize({ providerID })` | OAuth 授权 |
| `client.provider.oauth.callback({ providerID, code })` | OAuth 回调 |

### 文件搜索 (Find)

| 方法名 | 功能 |
|--------|------|
| `client.find.text({ pattern })` | 搜索文本 |
| `client.find.files({ query })` | 搜索文件 |
| `client.find.symbols({ query })` | 搜索符号 |

### 文件操作 (File)

| 方法名 | 功能 |
|--------|------|
| `client.file.list({ path })` | 列出文件 |
| `client.file.read({ path })` | 读取文件 |
| `client.file.status()` | 获取 Git 状态 |

### 事件订阅 (Event)

| 方法名 | 功能 |
|--------|------|
| `client.event.subscribe()` | 订阅事件流 |

### MCP 服务器 (Mcp)

| 方法名 | 功能 |
|--------|------|
| `client.mcp.status()` | 获取状态 |
| `client.mcp.add({ name, config })` | 添加服务器 |
| `client.mcp.connect({ name })` | 连接服务器 |
| `client.mcp.disconnect({ name })` | 断开服务器 |
| `client.mcp.auth.start({ name })` | 启动 OAuth |
| `client.mcp.auth.callback({ name, code })` | OAuth 回调 |
| `client.mcp.auth.authenticate({ name })` | 自动认证 |
| `client.mcp.auth.remove({ name })` | 移除认证 |

### TUI 控制 (Tui)

| 方法名 | 功能 |
|--------|------|
| `client.tui.appendPrompt({ text })` | 追加提示 |
| `client.tui.openHelp()` | 打开帮助 |
| `client.tui.openSessions()` | 打开会话对话框 |
| `client.tui.openThemes()` | 打开主题对话框 |
| `client.tui.openModels()` | 打开模型对话框 |
| `client.tui.submitPrompt()` | 提交提示 |
| `client.tui.clearPrompt()` | 清空提示 |
| `client.tui.executeCommand({ command })` | 执行命令 |
| `client.tui.showToast({ message, variant })` | 显示通知 |
| `client.tui.selectSession({ sessionID })` | 选择会话 |
| `client.tui.control.next()` | 获取请求 |
| `client.tui.control.response({ body })` | 提交响应 |

### 其他

| 方法名 | 功能 |
|--------|------|
| `client.instance.dispose()` | 释放实例 |
| `client.path.get()` | 获取路径信息 |
| `client.vcs.get()` | 获取 VCS 信息 |
| `client.command.list()` | 获取命令列表 |
| `client.lsp.status()` | 获取 LSP 状态 |
| `client.formatter.status()` | 获取格式化状态 |
| `client.worktree.list()` | 获取工作树列表 |
| `client.worktree.create()` | 创建工作树 |
| `client.worktree.remove()` | 移除工作树 |
| `client.worktree.reset()` | 重置工作树 |
| `client.experimental.workspace.list()` | 获取工作区列表 |
| `client.experimental.workspace.create()` | 创建工作区 |
| `client.experimental.workspace.remove()` | 移除工作区 |
| `client.experimental.resource.list()` | 获取资源列表 |

## 导出路径

- `@opencode-ai/sdk` - v1 API
- `@opencode-ai/sdk/v2` - v2 API（推荐）
