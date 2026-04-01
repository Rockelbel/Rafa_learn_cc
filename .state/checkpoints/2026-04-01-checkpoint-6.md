# Checkpoint 6 - 2026-04-01

## Module 4: Components System - 已完成

### 分析文件 (10+)
1. ✅ components/App.tsx - 根组件
2. ✅ components/Messages.tsx - 消息列表
3. ✅ components/Message.tsx - 消息组件
4. ✅ components/MessageRow.tsx - 消息行
5. ✅ components/Spinner.tsx - 加载动画
6. ✅ components/FileEditToolDiff.tsx - 差异显示
7. ✅ components/Markdown.tsx - Markdown 渲染
8. ✅ components/StructuredDiff.tsx - 结构化差异
9. ✅ components/design-system/*.tsx - 设计系统
10. ✅ components/ExportDialog.tsx - 导出对话框
11. ✅ components/GlobalSearchDialog.tsx - 全局搜索
12. ✅ components/Onboarding.tsx - 引导流程

### 生成的笔记文件
- raw_notes/components/app_architecture.md - App 架构分析
- raw_notes/components/tool_ui.md - 工具 UI 分析
- raw_notes/components/dialogs.md - 对话框分析
- raw_notes/components/design_system.md - 设计系统分析

### 关键发现摘要

#### 1. 组件架构 (App.tsx)

**组件层次**:
```
App.tsx (Root Provider Wrapper)
├── FpsMetricsProvider
├── StatsProvider
└── AppStateProvider
    └── Messages.tsx
        ├── LogoHeader
        ├── VirtualMessageList
        └── MessageRow → Message.tsx
```

**Context Providers**:
- `FpsMetricsProvider` - 性能监控
- `StatsProvider` - 统计信息
- `AppStateProvider` - 应用状态

#### 2. React Compiler 使用

所有组件使用 React 新编译器进行自动 memoization:
```typescript
import { c as _c } from "react/compiler-runtime"
const $ = _c(5) // 5 cache slots
```

**缓存模式**:
- Module-level caches (Markdown token cache, StructuredDiff render cache)
- WeakMap-based caching for search text
- useMemo 用于昂贵的消息转换

#### 3. 消息渲染流程 (Messages.tsx)

**三阶段渲染**:
1. **Streaming** - 实时流式消息
2. **In-Progress** - 进行中的工具调用
3. **Completed** - 已完成消息

**性能优化**:
- **Message capping with UUID anchoring** - 防止完整终端重置
- **Transform separation** - 昂贵的分组/折叠与廉价的切片分离
- **Virtualization support** - VirtualMessageList 当 scrollRef 存在时
- **OffscreenFreeze** - 缓存离屏内容

#### 4. 工具 UI 渲染模式

**混合渲染**:
- React 组件用于结构/交互
- 预渲染 ANSI 字符串用于性能
- Suspense 边界用于异步加载

**关键组件**:
- `FileEditToolDiff` - 差异显示，Suspense-based 异步加载
- `HighlightedCode` - 代码显示，可选语法高亮
- `Markdown` - Markdown 渲染，token 缓存
- `StructuredDiff` - 结构化差异，WeakMap 缓存

#### 5. 设计系统 (design-system/)

**主题系统**:
- 89 个颜色 token，语义化分类
- 6 个主题变体: dark, light, light-daltonized, dark-daltonized, light-ansi, dark-ansi
- Daltonization 策略用于色盲无障碍

**核心组件**:
- `Pane` - 斜杠命令屏幕主容器
- `Dialog` - 确认对话框，内置快捷键
- `Tabs` - Tab 导航，键盘支持
- `Divider` - 水平分隔线，Unicode-aware
- `StatusIcon` - 语义状态指示器
- `ListItem` - 选择项，指针/对勾/滚动箭头

**设计模式**:
- 终端原生设计使用框线字符和 ANSI 转义序列
- 语义颜色命名 (success, error, warning)
- 键盘优先交互模型
- Modal 上下文感知

#### 6. 对话框架构

**基础对话框 (Dialog.tsx)**:
- 一致的退出处理
- 输入引导
- Pane-based 容器
- `isCancelActive` 用于子输入集成

**模糊选择器 (FuzzyPicker.tsx)**:
- 通用搜索/选择组件
- 窗口化渲染
- 响应式预览定位

**搜索实现**:
- **GlobalSearch**: 防抖 ripgrep，流式结果，AbortController 取消
- **HistorySearch**: Async iterator-based 历史加载，精确 + 模糊匹配
- **QuickOpen**: 文件建议生成，世代计数器用于过时结果检测

**退出流程处理**:
- 双击机制 (第一次显示警告，第二次退出)
- 上下文取消
- `isCancelActive` prop 允许子 TextInput 接收 'n' 键

### 架构模式提取

#### 模式 1: React Compiler Memoization
```typescript
const $ = _c(5) // 5 cache slots
if ($[0] !== messages) {
  $[0] = messages
  $[1] = expensiveTransform(messages)
}
const transformed = $[1]
```

#### 模式 2: Provider 组合
```typescript
<ProviderA>
  <ProviderB>
    <ProviderC>
      <ActualContent />
    </ProviderC>
  </ProviderB>
</ProviderA>
```

#### 模式 3: 消息类型分发
```typescript
function Message({ message }) {
  switch (message.type) {
    case 'user': return <UserMessage message={message} />
    case 'assistant': return <AssistantMessage message={message} />
    // ...
  }
}
```

#### 模式 4: 混合渲染 (React + ANSI)
```typescript
<Box>
  <RawAnsi>{preRenderedAnsi}</RawAnsi>
</Box>
```

### 复用经验

1. **React Compiler 优化** - 自动 memoization 减少手动 useMemo
2. **分层 providers** - 清晰的依赖注入
3. **混合渲染策略** - React 用于交互，ANSI 用于性能
4. **WeakMap 缓存** - 对象关联缓存不阻止垃圾回收
5. **语义化主题** - 功能命名而非美学命名

### 有趣彩蛋
- OffscreenFreeze 组件缓存离屏内容提高性能
- Daltonization 支持色盲用户
- `stringWidth()` 用于 Unicode-aware 文本居中
- VirtualMessageList 用于大量消息虚拟化

## 下一步
开始 Module 5: Skills System (深入分析)

