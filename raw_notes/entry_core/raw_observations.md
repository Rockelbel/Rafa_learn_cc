# Entry & Core 模块学习笔记 - 原始观察

## 文件概览

### 1. main.tsx (4683 行) - 主入口文件
这是整个 Claude Code 的入口点，展示了高度工程化的启动流程。

#### 关键观察：

**启动性能优化策略**
- 第 1-20 行：顶级 side effects 优先执行 - profileCheckpoint, startMdmRawRead, startKeychainPrefetch
- 使用并行预取策略减少启动时间
- 注释说明：这些操作在其余 ~135ms 的导入期间并行运行
- 避免 macOS 上串行 spawn 导致的 ~65ms 延迟

**特性门控 (Feature Gating)**
- 使用 `feature('FLAG_NAME')` 模式进行编译时特性控制
- 大量条件导入和死代码消除 (DCE) 优化
- 例如：`feature('ABLATION_BASELINE')`, `feature('BRIDGE_MODE')`, `feature('DAEMON')`

**安全机制**
- 第 232-271 行：调试检测机制，防止在调试器下运行
- 检查 process.execArgv, NODE_OPTIONS 和 inspector
- 如果检测到调试，直接 process.exit(1)

**迁移系统**
- 第 324-352 行：版本迁移机制
- CURRENT_MIGRATION_VERSION = 11
- 包含多个迁移函数：migrateSonnet1mToSonnet45, migrateLegacyOpusToCurrent 等

**延迟预取机制**
- startDeferredPrefetches() 函数 (第 388-431 行)
- 区分"关键路径"和"延迟预取"
- 避免在 --bare 模式或性能测试时执行预取

### 2. context.ts (189 行) - 上下文管理

#### 关键观察：

**Git 状态集成**
- getGitStatus() 函数 - 使用 lodash memoize 缓存
- 获取分支、主分支、状态、最近提交、用户名
- 限制状态字符数为 2000，超出则截断
- 详细的性能日志记录

**双重上下文系统**
- getSystemContext() - 系统级上下文，包含 Git 状态和缓存破坏器
- getUserContext() - 用户级上下文，包含 ClaudeMd 和当前日期
- 都使用 memoize 缓存

**CLAUDE.md 集成**
- 支持禁用 (--bare 模式或环境变量)
- 自动发现和缓存

### 3. entrypoints/cli.tsx - CLI 入口

#### 关键观察：

**多入口设计**
- 不同的启动路径：--version, --dump-system-prompt, --claude-in-chrome-mcp, remote-control, daemon, bg sessions
- 使用动态导入延迟加载模块

**Corepack 修复**
- 第 5 行：`process.env.COREPACK_ENABLE_AUTO_PIN = '0'`
- 防止 corepack 自动修改 package.json

**远程环境优化**
- CCR 环境设置最大堆内存为 8GB

### 4. entrypoints/init.ts - 初始化核心

#### 关键观察：

**分阶段初始化**
- 配置启用 → 安全环境变量 → 优雅关闭 → 事件日志 → OAuth → JetBrains 检测 → 仓库检测 → 远程设置 → mTLS → 代理 → 预连接

**延迟加载策略**
- OpenTelemetry 延迟加载 (~400KB)
- gRPC 导出器进一步延迟 (~700KB)
- LSP 管理器清理注册但延迟初始化

**遥测初始化时机**
- initializeTelemetryAfterTrust() - 信任后才初始化遥测
- 区分远程设置用户和非远程设置用户

### 5. context/modalContext.tsx - 模态上下文

#### 关键观察：

**React Compiler 使用**
- 使用 `import { c as _c } from "react/compiler-runtime"`
- 这是 React 19 的编译器特性

**Modal 尺寸管理**
- 处理模态对话框内的尺寸约束
- 提供 useModalOrTerminalSize hook
- 支持 ScrollBox 引用

## 有趣发现

1. **启动性能极度重视** - 大量的并行预取、延迟加载、性能计时点
2. **安全防御纵深** - 调试检测、环境变量检查、沙箱机制
3. **特性管理精细** - feature() 门控遍布代码，支持条件编译
4. **代码组织模块化** - 清晰的分离：context、entrypoints、init
5. **遥测无处不在** - 详细的性能日志和诊断日志

## 待深入研究

- feature() 系统的实现机制
- 启动性能分析器的详细实现
- 迁移系统的触发机制
- 信任对话框的时机和逻辑
