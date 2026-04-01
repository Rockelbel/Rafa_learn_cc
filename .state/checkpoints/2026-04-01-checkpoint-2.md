# Checkpoint 2 - 2026-04-01

## 已完成文件 (7)
1. ✅ main.tsx (4683 行) - 完整阅读，主入口，包含所有 CLI 命令和启动逻辑
2. ✅ context.ts - 上下文管理系统
3. ✅ entrypoints/init.ts - 系统初始化
4. ✅ entrypoints/agentSdkTypes.ts - SDK API 定义
5. ✅ entrypoints/sdk/coreTypes.ts - SDK 核心类型
6. ✅ context/modalContext.tsx - Modal 上下文
7. ✅ context/notifications.tsx - 通知系统

## 进行中文件 (6)
- entrypoints/cli.tsx - CLI 多入口
- context/overlayContext.tsx - Overlay 管理
- context/stats.tsx - 统计
- context/voice.tsx - 语音
- context/mailbox.tsx - 邮箱
- context/promptOverlayContext.tsx - Prompt Overlay

## main.tsx 关键发现摘要

### 启动流程架构
1. **Side Effects 优先** (第 1-20 行): profileCheckpoint, startMdmRawRead, startKeychainPrefetch
2. **多模式支持**: CLI, SDK, Daemon, Bridge, SSH, Assistant, Remote Control
3. **安全防御**: 调试检测、信任对话框、权限系统
4. **复杂特性门控**: feature() 函数控制 KAIROS, BRIDGE_MODE, DIRECT_CONNECT 等

### 子命令系统
- mcp: MCP 服务器管理 (add, remove, list, get, serve)
- auth: 认证管理 (login, status, logout)
- plugin: 插件管理 (install, uninstall, enable, disable, update)
- server: 启动 Claude Code 服务器
- ssh: 远程 SSH 会话
- assistant: 连接到 assistant 会话
- doctor: 健康检查
- update: 更新检查

### 核心启动模式
1. **Interactive REPL** - 默认交互模式
2. **Print Mode (-p)** - 非交互式，输出结果并退出
3. **Daemon Mode** - 后台守护进程
4. **Bridge/Remote Control** - 远程控制
5. **Server Mode** - HTTP 服务器

### 性能优化技术
- 并行预取 (parallel prefetching)
- 延迟加载 (lazy loading)
- 特性门控死代码消除
- 启动性能分析 (profileCheckpoint)

## 下一步
继续完成 Entry & Core 模块剩余文件，特别是 entrypoints/cli.tsx 和剩余 context 文件。
