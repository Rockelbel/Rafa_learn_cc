# Checkpoint 7 - 2026-04-01

## Module 5: Skills System - 已完成

### 分析文件 (10+)
1. ✅ skills/loadSkillsDir.ts - Skill 加载
2. ✅ skills/bundledSkills.ts - 内置 Skill
3. ✅ skills/parseSkill.ts - Skill 解析
4. ✅ skills/skillSchema.ts - Schema 验证
5. ✅ skills/bundled/*.ts - 内置 Skill 示例
6. ✅ utils/hooks.ts - Hook 执行
7. ✅ utils/hooks/useHooks.ts - Hook 使用
8. ✅ types/hooks.ts - Hook 类型
9. ✅ schemas/hooks.ts - Hook Schema
10. ✅ entrypoints/sdk/coreTypes.ts - SDK Hook 常量

### 生成的笔记文件
- raw_notes/skills/skill_loading.md - Skill 加载系统
- raw_notes/skills/bundled_skills.md - 内置 Skill
- raw_notes/skills/skill_hooks.md - Skill Hook 系统
- raw_notes/skills/hooks_execution.md - Hook 执行系统

### 关键发现摘要

#### 1. Skill 发现算法 (loadSkillsDir.ts)

**发现源优先级 (高到低)**:
```
1. Managed/Policy: ~/.claude-code/managed/.claude/skills/
2. User: ~/.claude/skills/
3. Project: ./.claude/skills/ (walks up to git root)
4. Additional: --add-dir paths
5. Legacy: ~/.claude/commands/ (DEPRECATED)
```

**目录结构**:
```
.skills/
├── my-skill/
│   └── SKILL.md          # Required: skill definition
└── another-skill/
    ├── SKILL.md
    └── helper-script.sh  # Optional: bundled files
```

#### 2. SKILL.md 格式

**YAML Frontmatter + Markdown**:
```yaml
---
name: skill-name
description: Skill description
arguments:
  - name: arg1
    description: First argument
    required: true
allowed-tools: ['Read', 'Edit', 'Bash']
paths:
  - "src/**/*.ts"  # 条件激活模式
hooks:
  PreToolUse:
    - type: command
      command: echo "Tool use detected"
---

# Skill content in markdown
Goal: What this skill does

Steps:
1. Step one
2. Step two
```

**特殊 YAML 处理**:
- 大括号扩展: `src/*.{ts,tsx}`
- Glob 模式: `**/*.md`
- 多行字符串: `|-` 保持格式

#### 3. 变量替换 (argumentSubstitution.ts)

**支持的变量**:
- 命名参数: `${argName}` 来自 frontmatter `arguments:`
- 索引参数: `$0`, `$1`, `$ARGUMENTS[0]`
- 完整参数: `$ARGUMENTS`
- 内置变量:
  - `${CLAUDE_SKILL_DIR}` - Skill 目录
  - `${CLAUDE_SESSION_ID}` - Session ID

**替换逻辑**:
- 如果未找到占位符，追加 `ARGUMENTS: {args}`
- 支持 shell 风格 `$VAR` 和 `${VAR}`
- 数组索引支持: `$ARGUMENTS[n]`

#### 4. 缓存策略

**多层缓存**:
- `lodash-es/memoize` 函数级缓存
- 自定义缓存键: 组合 subdir 和 cwd
- 通过 resolved realpath 去重 (处理 symlinks)
- `clearSkillCaches()` 清除所有缓存

**文件监听 (Chokidar)**:
- 监听 user/project/additional skill 目录
- 300ms debounce 批处理快速变化
- 1 秒文件稳定阈值
- Bun 下使用 polling 避免 PathWatcherManager 死锁
- 忽略 `.git` 目录和特殊文件类型

#### 5. 内置 Skill (bundledSkills.ts)

**注册流程**:
1. `initBundledSkills()` 在启动时调用
2. 每个 skill 调用 `registerBundledSkill()`
3. 提供 `BundledSkillDefinition` 配置

**嵌入式文件处理**:
- `files` 属性: `Record<string, string>` 文件路径和内容
- 首次调用时惰性提取
- 提取到每进程 nonce 目录
- 安全保护: 0o700/0o600 权限, O_NOFOLLOW|O_EXCL 标志

**安全特性**:
- 路径遍历验证
- 符号链接攻击防护
- 原子文件写入 (O_EXCL)
- 每进程随机 nonce

#### 6. Hook 系统 (utils/hooks.ts)

**29 个 Hook 事件**:
```
SessionStart, SessionEnd
PreToolUse, PostToolUse, PostToolUseFailure
PreCompact, PostCompact
UserPromptSubmit, Notification
Stop, PermissionRequest
Elicitation, ConfigChange
CwdChanged, FileChanged
...
```

**Hook 类型**:
1. **command** - Shell 执行 (bash/PowerShell)
2. **prompt** - 单轮 LLM 查询
3. **agent** - 多轮 LLM Agent (最多 50 轮)
4. **http** - POST 到外部 URL
5. **callback** - 内存中 TypeScript 回调
6. **function** - Session-scoped 回调 (不可持久化)

#### 7. Hook 执行流程

```
1. 从所有源加载配置 via getAllHooks()
2. 通过 groupHooksByEventAndMatcher() 按事件和匹配器分组
3. 使用模式匹配 (精确、管道分隔、正则、通配符)
4. 通过 command/prompt/URL 去重
5. 通过 if 条件过滤 (工具事件)
6. 按优先级顺序顺序执行
7. 处理结果和处理阻塞错误 (exit code 2)
```

**注册源优先级**:
1. User Settings → 2. Project Settings → 3. Local Settings → 4. Policy Settings → 5. Session Hooks → 6. Plugin Hooks → 7. Built-in Hooks

#### 8. 状态管理

- **Session hooks**: 存储在 Map 中，O(1) 变更不触发 listeners
- **Async hooks**: 在 `AsyncHookRegistry` 中注册，轮询响应
- **Configuration snapshot**: 启动时捕获，策略强制执行

#### 9. 安全特性

- 交互模式下所有 hooks 需要工作区信任
- 基于策略的限制 (`allowManagedHooksOnly`, `disableAllHooks`)
- HTTP hooks 的 SSRF 防护，域名白名单
- Header 插值的环境变量白名单

### 架构模式提取

#### 模式 1: 目录式 Skill 定义
```
skill-name/
├── SKILL.md        # 主定义
└── ...             # 辅助文件
```

#### 模式 2: YAML Frontmatter + Markdown
```yaml
---
metadata...
---
content...
```

#### 模式 3: 多层发现优先级
```
Managed → User → Project → Additional → Legacy
```

#### 模式 4: Hook 类型多态
```typescript
type Hook =
  | { type: 'command', command: string }
  | { type: 'prompt', prompt: string }
  | { type: 'agent', agent: string }
  | { type: 'http', url: string }
```

### 复用经验

1. **Frontmatter + Markdown** - 人类可读，机器可解析
2. **多层发现** - 用户/项目/系统级配置分离
3. **变量替换** - 灵活的参数传递
4. **文件监听 + Debounce** - 热重载不牺牲性能
5. **惰性提取** - 首次使用时解压资源

### 有趣彩蛋
- 内置 Skill `stuck.ts` 只在 `USER_TYPE === 'ant'` 时注册
- `remember.ts` 使用 `isEnabled` 动态启用
- Chokidar 在 Bun 下使用 polling 避免死锁
- Legacy `~/.claude/commands/` 目录仍支持但已弃用

## 下一步
开始 Module 6: Tasks & Agents

