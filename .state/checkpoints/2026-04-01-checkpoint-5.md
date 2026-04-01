# Checkpoint 5 - 2026-04-01

## Module 3: Commands System - 已完成

### 分析文件 (4)
1. ✅ commands.ts - 命令注册中心 (100+ 命令)
2. ✅ types/command.ts - 命令类型定义
3. ✅ utils/commandLifecycle.ts - 命令生命周期
4. ✅ skills/loadSkillsDir.ts - Skill 加载系统
5. ✅ skills/bundledSkills.ts - 内置 Skill
6. ✅ commands/commit.ts - Commit 命令
7. ✅ commands/config/index.ts - Config 命令
8. ✅ commands/plan/index.ts - Plan 命令
9. ✅ commands/help/index.ts - Help 命令
10. ✅ commands/skills/index.ts - Skills 命令

### 生成的笔记文件
- raw_notes/commands/command_infrastructure.md - 命令基础设施分析
- raw_notes/commands/sample_commands.md - 示例命令分析
- raw_notes/commands/skills_system.md - Skills 系统分析
- raw_notes/commands/commit_flow.md - Commit 流程分析

### 关键发现摘要

#### 1. 三种命令类型 (types/command.ts)

**PromptCommand** (`type: 'prompt'`)
- 模型可调用的命令，扩展为对话内容
- `getPromptForCommand()` 返回 ContentBlockParam[]
- 支持 `allowedTools` 白名单
- 可配置为 'inline' 或 'fork' 执行上下文
- Skill 的主要实现方式

**LocalCommand** (`type: 'local'`)
- 本地执行，返回文本结果
- `load()` 懒加载实现
- `call()` 返回 `LocalCommandResult`
- 适合不需要 UI 的实用工具

**LocalJSXCommand** (`type: 'local-jsx'`)
- 渲染 React/Ink UI 组件
- `load()` 懒加载
- `call(onDone, context, args)` 返回 React.ReactNode
- `onDone` 回调控制完成行为

#### 2. 命令注册架构 (commands.ts)

**注册源优先级 (由高到低)**:
```
1. Bundled skills (编译进 CLI)
2. Built-in plugin skills (用户可开关)
3. Skill directory commands (~/.claude/skills/, .claude/skills/)
4. Workflow commands (feature flag)
5. Plugin commands (市场插件)
6. Plugin skills (来自插件)
7. Built-in commands (硬编码 COMMANDS 数组)
```

**懒加载模式**:
- Feature flag 命令: 条件 `require()` 在模块级
- Local/JSX 命令: `load()` 函数返回 Promise
- Bundled skills: memoized extraction promises
- Dynamic imports: 延迟重依赖到调用时

#### 3. Auth/Availability 系统

```typescript
type CommandAvailability = 'claude-ai' | 'console'
```

- `meetsAvailabilityRequirement()` - 按 auth provider 过滤
- 在 `isEnabled()` 检查之前运行
- **不缓存** - 每次调用重新评估 (auth 状态可能变化)
- `loadAllCommands()` 缓存 (昂贵的磁盘 I/O)

#### 4. Skill 系统架构

**Skill 定义格式 (SKILL.md)**:
```yaml
---
name: skill-name
description: Skill description
allowed-tools: ['Read', 'Edit', 'Bash']
user-invocable: true
model: sonnet
---
```

**加载架构**:
- 四来源: Bundled、disk-based、plugins、MCP servers
- 加载顺序: Bundled → built-in plugins → disk skills → workflows → plugins
- 缓存: `getSkillDirCommands()`, `loadAllCommands()` memoized per CWD
- 去重: 通过解析后的文件路径使用 `realpath()`

**动态发现**:
- 文件操作触发发现: `discoverSkillDirsForPaths()`
- 条件 Skill: 路径模式匹配，在匹配文件被触碰时激活
- 热重载: Chokidar 文件监听 + debounce (300ms)

#### 5. Commit 流程分析

**验证流程**:
- Tool 白名单通过 `ALLOWED_TOOLS` 数组
- 权限上下文注入限制 tool 访问
- Git 安全协议嵌入 prompts (no --amend, no --no-verify, no secrets)

**Git 操作**:
- 使用内联 shell 命令替换获取上下文
- HEREDOC 语法支持多行 commit 消息
- 仅非交互模式 (无 `-i` flags)

**Attribution 系统**:
- `getAttributionTexts()` 生成 Co-Authored-By 行
- `getEnhancedPRAttribution()` 计算贡献统计
- Undercover 模式为公共仓库剥离所有 attribution

### 架构模式提取

#### 模式 1: 多类型命令系统
```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
// 使用 discriminated union 区分三种命令类型
```

#### 模式 2: 分层命令源
```
用户输入 → 发现 Skills → 加载 Plugins → 合并 Built-in → 过滤 Auth → 过滤 Enabled → 可用命令
```

#### 模式 3: 懒加载工厂
```typescript
const command: LocalJSXCommand = {
  type: 'local-jsx',
  load: async () => {
    const { call } = await import('./implementation.js')
    return { call }
  }
}
```

### 复用经验

1. **命令类型分离** - 清晰区分 AI 调用 vs 本地执行 vs UI 渲染
2. **分层注册** - 多来源合并，优先级明确
3. **条件可用性** - auth 和能力分离，独立过滤
4. **懒加载一切** - 减少启动时间，按需加载
5. **Skill 即代码** - markdown 定义可版本控制的 AI 工作流

### 有趣彩蛋
- INTERNAL_ONLY_COMMANDS 包含 25+ 内部命令 (如 bughunter, mockLimits, antTrace)
- `insights` 命令懒加载 113KB 的分析模块
- Skills 支持 shell 变量替换: `${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}`

## 下一步
开始 Module 4: Components System

