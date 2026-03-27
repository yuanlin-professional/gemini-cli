# Todo.tsx

## 概述

`Todo.tsx` 导出了 `TodoTray` 组件，用于在终端 CLI 界面底部（或指定位置）展示一个 **待办事项清单托盘**。它从全局 UI 状态的历史记录中自动检索最近一次由工具输出的 `TodoList` 数据，将其转换为 `Checklist` 组件所需的格式并渲染。用户可以通过快捷键切换清单的展开/折叠状态。

**文件路径**: `packages/cli/src/ui/components/messages/Todo.tsx`

## 架构图（Mermaid）

```mermaid
graph TD
    A[TodoTray 组件] --> B[useUIState Hook 获取全局 UI 状态]
    B --> C[uiState.history 历史记录]
    B --> D[uiState.showFullTodos 展开状态]

    A --> E[useMemo: 查找最近的 TodoList]
    E --> F[从后向前遍历 history]
    F --> G{entry.type === 'tool_group'?}
    G -->|否| F
    G -->|是| H[遍历 toolGroup.tools]
    H --> I{tool.resultDisplay 包含 todos?}
    I -->|否| H
    I -->|是| J[返回 TodoList 对象]
    I -->|全部遍历完毕| K[返回 null]

    A --> L[useMemo: 转换为 ChecklistItemData 数组]
    L --> M[映射 todo.status → status]
    L --> N[映射 todo.description → label]

    A --> O{todos 存在?}
    O -->|否| P[返回 null 不渲染]
    O -->|是| Q[渲染 Checklist 组件]
    Q --> R[标题: "Todo"]
    Q --> S[items: checklistItems]
    Q --> T[isExpanded: showFullTodos]
    Q --> U[toggleHint: 快捷键提示]
```

## 核心组件

### `TodoTray` 组件

**类型**: `React.FC`（无 Props）

**功能**: 无需外部传入数据，完全从全局 UI 上下文中自主获取待办事项列表并渲染。

**核心逻辑**:

#### 1. 查找最近的 TodoList

通过 `useMemo` 缓存，从 `uiState.history` 数组**从后向前**遍历，找到最近一个符合条件的工具输出：

```
条件:
  - history entry 的 type 必须是 'tool_group'
  - toolGroup 中某个 tool 的 resultDisplay 是对象
  - 该 resultDisplay 对象包含 'todos' 属性
```

这种从后向前的遍历策略确保获取到的是**最新**的待办事项列表。

#### 2. 数据转换

将 `TodoList.todos` 数组中的每个 todo 项映射为 `ChecklistItemData` 格式：

| TodoList 字段 | ChecklistItemData 字段 | 说明 |
|---------------|----------------------|------|
| `todo.status` | `status` | 待办项状态（如 pending、completed 等） |
| `todo.description` | `label` | 待办项的描述文本 |

#### 3. 渲染

使用 `Checklist` 组件渲染，传入以下参数：

| Prop | 值 | 说明 |
|------|-----|------|
| `title` | `"Todo"` | 清单标题 |
| `items` | `checklistItems` | 转换后的清单项数组 |
| `isExpanded` | `uiState.showFullTodos` | 是否展开显示所有项目 |
| `toggleHint` | `"{快捷键} to toggle"` | 提示用户可用的切换快捷键 |

**空状态处理**: 当没有找到任何 TodoList 或 todos 数组为空时，组件返回 `null`，不渲染任何内容。

## 依赖关系

### 内部依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `@google/gemini-cli-core` | `TodoList` (类型) | 待办事项列表的 TypeScript 类型定义 |
| `../../contexts/UIStateContext.js` | `useUIState` | 获取全局 UI 状态的 React Hook，提供 history 和 showFullTodos |
| `../../types.js` | `HistoryItemToolGroup` (类型) | 工具组历史记录项的类型定义 |
| `../Checklist.js` | `Checklist` | 可展开/折叠的清单渲染组件 |
| `../ChecklistItem.js` | `ChecklistItemData` (类型) | 清单项数据的类型定义，包含 status 和 label |
| `../../key/keybindingUtils.js` | `formatCommand` | 将命令枚举转换为用户可读的快捷键字符串 |
| `../../key/keyBindings.js` | `Command` | 命令枚举，使用 `Command.SHOW_FULL_TODOS` |

### 外部依赖

| 包名 | 导入内容 | 用途 |
|------|----------|------|
| `react` | `React` (类型), `useMemo` | React 类型支持和记忆化 Hook |

## 关键实现细节

1. **自底向上历史检索**: `TodoTray` 不接收 props，而是自行从 `uiState.history` 数组从后向前遍历查找最近的 `TodoList`。这种设计使得组件可以独立放置在 UI 的任何位置（如底部状态栏），无需父组件显式传递数据。

2. **双层遍历查找**: 外层遍历 history entries，内层遍历每个 `tool_group` 中的 tools。一旦找到第一个包含 `todos` 属性的 `resultDisplay`，立即返回，避免不必要的继续搜索。

3. **两级 useMemo 缓存**:
   - 第一级：`todos` 的查找结果缓存，依赖 `uiState.history`。只有当历史记录变化时才重新搜索。
   - 第二级：`checklistItems` 的映射结果缓存，依赖 `todos`。只有当找到的 TodoList 变化时才重新映射。
   这种两级缓存策略确保了即使 history 长度很大，也不会在每次渲染时都进行完整遍历和数据转换。

4. **快捷键提示生成**: 使用 `formatCommand(Command.SHOW_FULL_TODOS)` 动态生成快捷键提示字符串，而非硬编码。这意味着如果快捷键配置发生变化，提示信息会自动更新。

5. **鸭子类型检测**: 判断 `resultDisplay` 是否为 TodoList 时使用的是 `'todos' in tool.resultDisplay` 这种鸭子类型检测方式，而非严格的类型守卫。这提供了灵活性，允许不同的工具（如 `WriteTodosTool` 或 Tracker 工具）以相同的数据结构输出待办事项。

6. **展开/折叠状态外部管理**: `isExpanded` 状态存储在全局 `UIState` 中（`uiState.showFullTodos`），而非组件内部。这使得快捷键处理器可以在任何地方修改此状态，组件会自动响应变化。
