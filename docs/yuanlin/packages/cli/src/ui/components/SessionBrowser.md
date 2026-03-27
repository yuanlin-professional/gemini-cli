# SessionBrowser.tsx

## 概述

`SessionBrowser` 是 Gemini CLI 中用于**会话浏览与管理**的完整交互式 UI 组件。它实现了一个类似 `less` 命令的终端界面，允许用户浏览所有历史会话、搜索会话内容、排序和筛选、恢复历史会话、以及删除无用会话。该文件采用了**状态集中管理 + 自定义 Hooks 拆分**的架构模式，将数据加载、状态管理、键盘输入处理、排序/过滤逻辑等分别封装为独立的 Hook，通过统一的 `SessionBrowserState` 接口传递状态，消除了 props drilling 问题。

## 架构图（Mermaid）

```mermaid
graph TD
    A[SessionBrowser 入口组件] --> B[useSessionBrowserState<br/>集中状态管理钩子]
    A --> C[useLoadSessions<br/>会话数据加载钩子]
    A --> D[useMoveSelection<br/>选择移动钩子]
    A --> E[useCycleSortOrder<br/>排序切换钩子]
    A --> F[useSessionBrowserInput<br/>键盘输入处理钩子]
    A --> G[SessionBrowserView<br/>视图渲染组件]

    B --> H[SessionBrowserState<br/>集中状态对象]

    G --> I{加载中?}
    I -- 是 --> J[SessionBrowserLoading<br/>加载状态]
    I -- 否 --> K{有错误?}
    K -- 是 --> L[SessionBrowserError<br/>错误状态]
    K -- 否 --> M{会话为空?}
    M -- 是 --> N[SessionBrowserEmpty<br/>空状态]
    M -- 否 --> O[正常列表视图]

    O --> P[SessionListHeader<br/>列表头部]
    O --> Q{搜索模式?}
    Q -- 是 --> R[SearchModeDisplay<br/>搜索输入框]
    O --> S{有过滤结果?}
    S -- 否 --> T[NoResultsDisplay<br/>无搜索结果]
    S -- 是 --> U[SessionList<br/>会话列表容器]

    U --> V[SessionTableHeader<br/>表头 - 序号/消息数/时间/名称]
    U --> W[SessionItem<br/>单行会话项 x N]
    W --> X[MatchSnippetDisplay<br/>搜索匹配片段高亮]

    F --> Y{搜索模式?}
    Y -- 是 --> Z[搜索输入处理<br/>ESC退出/退格删除/字符输入]
    Y -- 否 --> AA[导航模式处理]
    AA --> AB[g/G: 首尾跳转]
    AA --> AC[s: 排序切换]
    AA --> AD[r: 反转排序]
    AA --> AE[/: 进入搜索]
    AA --> AF[q/ESC: 退出]
    AA --> AG[x: 删除会话]
    AA --> AH[u/d: 半页翻页]

    F --> AI[通用按键处理<br/>Enter选择/上下/翻页]

    C --> AJ[初始加载<br/>getSessionFiles]
    C --> AK[搜索时加载完整内容<br/>includeFullContent]
```

## 核心组件

### 1. `SessionBrowserProps` 接口

| 属性 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `config` | `Config` | 是 | 应用配置对象，用于获取存储路径和当前会话 ID |
| `onResumeSession` | `(session: SessionInfo) => void` | 是 | 用户选择恢复某个会话时的回调 |
| `onDeleteSession` | `(session: SessionInfo) => Promise<void>` | 是 | 用户删除某个会话时的异步回调 |
| `onExit` | `() => void` | 是 | 用户退出会话浏览器时的回调 |

### 2. `SessionBrowserState` 接口

集中化的状态管理接口，包含以下类别的字段：

#### 数据状态
| 字段 | 类型 | 说明 |
|------|------|------|
| `sessions` | `SessionInfo[]` | 所有已加载的会话列表 |
| `filteredAndSortedSessions` | `SessionInfo[]` | 经过过滤和排序后的会话列表 |

#### UI 状态
| 字段 | 类型 | 说明 |
|------|------|------|
| `loading` | `boolean` | 是否正在加载 |
| `error` | `string \| null` | 错误信息 |
| `activeIndex` | `number` | 当前选中会话的索引 |
| `scrollOffset` | `number` | 当前滚动偏移量（分页用） |
| `terminalWidth` | `number` | 终端宽度 |

#### 搜索状态
| 字段 | 类型 | 说明 |
|------|------|------|
| `searchQuery` | `string` | 当前搜索查询字符串 |
| `isSearchMode` | `boolean` | 是否处于搜索输入模式 |
| `hasLoadedFullContent` | `boolean` | 是否已加载完整会话内容（用于深度搜索） |

#### 排序状态
| 字段 | 类型 | 说明 |
|------|------|------|
| `sortOrder` | `'date' \| 'messages' \| 'name'` | 当前排序依据 |
| `sortReverse` | `boolean` | 是否反转排序顺序 |

#### 计算值
| 字段 | 类型 | 说明 |
|------|------|------|
| `totalSessions` | `number` | 过滤后的会话总数 |
| `startIndex` | `number` | 当前页起始索引 |
| `endIndex` | `number` | 当前页结束索引 |
| `visibleSessions` | `SessionInfo[]` | 当前页可见的会话列表 |

此外还包含所有状态对应的 setter 函数（`setSessions`、`setLoading` 等）。

### 3. 常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `SESSIONS_PER_PAGE` | `20` | 每页显示的会话数量 |
| `FIXED_SESSION_COLUMNS_WIDTH` | `30` | 固定列（序号、消息数、时间、分隔符等）的预估字符宽度 |

### 4. 内部子组件

#### `SessionTableHeader`
表格头部组件，渲染列标签（Index / Msgs / Age / Name 或 Match）。在有滚动偏移时显示向上滚动指示器 `▲`。搜索模式下最后一列标签从 "Name" 变为 "Match"。

#### `MatchSnippetDisplay`
搜索结果匹配片段展示组件。显示匹配到的第一条片段，包含角色前缀（You/Gemini）和高亮的匹配文本。用户消息前缀为绿色，Gemini 消息前缀为蓝色，匹配文本为红色加粗。

#### `SessionItem`
单条会话行组件。渲染一行包含以下列信息的会话记录：
- 选中指示器 `❯`（当前选中行）
- 序号 `#N`
- 消息数量
- 相对时间（如 "2h"、"3d"）
- 会话名称/显示名称（带截断处理）或搜索匹配片段
- 附加信息标签（如 "(current)"、"(+N more)"）

当前会话项以灰色 dimColor 显示，并禁止选择恢复。

#### `SessionList`
会话列表容器组件，组合了导航帮助文本（非搜索模式）、表头、会话项列表和底部滚动指示器。

### 5. 自定义 Hooks

#### `useSessionBrowserState(initialSessions?, initialLoading?, initialError?)`
集中状态管理钩子，创建并返回 `SessionBrowserState` 对象。内部：
- 管理所有状态变量（`useState`）。
- 通过 `useMemo` 计算 `filteredAndSortedSessions`：先过滤（`filterSessions`）再排序（`sortSessions`）。
- 搜索查询清空时自动重置 `hasLoadedFullContent` 标记。
- 计算分页相关的派生值（`totalSessions`、`startIndex`、`endIndex`、`visibleSessions`）。

#### `useLoadSessions(config, state)`
会话数据加载钩子，包含两个 `useEffect`：
1. **初始加载**：挂载时从 `config.storage.getProjectTempDir() + '/chats'` 目录读取会话文件。
2. **深度搜索加载**：当进入搜索模式且未加载完整内容时，以 `{ includeFullContent: true }` 参数重新加载所有会话的完整内容，支持全文搜索。

#### `useMoveSelection(state)`
选择移动钩子，返回 `(delta: number) => void` 函数。自动处理边界限制（不超出 0 和 `totalSessions - 1`），并在选中项超出可视范围时自动调整 `scrollOffset`。

#### `useCycleSortOrder(state)`
排序切换钩子，在 `date` -> `messages` -> `name` -> `date` 之间循环切换。

#### `useSessionBrowserInput(state, moveSelection, cycleSortOrder, ...callbacks)`
键盘输入处理钩子，包含两大模式的按键处理：

**搜索模式**：
| 按键 | 操作 |
|------|------|
| ESC | 退出搜索模式，清空查询 |
| Backspace | 删除最后一个字符 |
| 其他可打印字符 | 追加到搜索查询 |

**导航模式**：
| 按键 | 操作 |
|------|------|
| `g` | 跳转到第一个会话 |
| `G` | 跳转到最后一个会话 |
| `s` | 切换排序方式 |
| `r` | 反转排序顺序 |
| `/` | 进入搜索模式 |
| `q` / `Q` / ESC | 退出浏览器 |
| `x` / `X` | 删除选中会话（不可删除当前会话） |
| `u` | 向上翻半页 |
| `d` | 向下翻半页 |

**通用按键**（搜索和导航模式均有效）：
| 按键 | 操作 |
|------|------|
| Enter | 恢复选中会话（当前会话除外） |
| 上/下箭头 | 单步移动 |
| PageUp/PageDown | 整页翻页 |

### 6. 顶层组件

#### `SessionBrowserView({ state })`
纯视图组件，根据 `state` 中的 `loading`、`error`、`sessions.length` 依次判断渲染哪种视图（加载中 / 错误 / 空 / 正常列表）。

#### `SessionBrowser({ config, onResumeSession, onDeleteSession, onExit })`
入口组件，组合所有自定义 Hooks 并将 `state` 传递给 `SessionBrowserView` 渲染。

## 依赖关系

### 内部依赖

| 模块路径 | 导入内容 | 用途 |
|----------|----------|------|
| `../semantic-colors.js` | `theme` | 语义化颜色主题 |
| `../colors.js` | `Colors` | 基础颜色常量（Gray、Foreground、AccentGreen 等） |
| `../hooks/useTerminalSize.js` | `useTerminalSize` | 获取终端尺寸 |
| `../hooks/useKeypress.js` | `useKeypress` | 键盘按键监听钩子 |
| `../../utils/sessionUtils.js` | `formatRelativeTime`, `getSessionFiles`, `SessionInfo`（类型） | 会话文件读取、相对时间格式化、会话信息类型 |
| `./SessionBrowser/SessionBrowserNav.js` | `SearchModeDisplay`, `NavigationHelpDisplay`, `NoResultsDisplay` | 搜索输入框、导航帮助提示、无结果提示子组件 |
| `./SessionBrowser/SessionListHeader.js` | `SessionListHeader` | 列表头部组件 |
| `./SessionBrowser/SessionBrowserLoading.js` | `SessionBrowserLoading` | 加载状态组件 |
| `./SessionBrowser/SessionBrowserError.js` | `SessionBrowserError` | 错误状态组件 |
| `./SessionBrowser/SessionBrowserEmpty.js` | `SessionBrowserEmpty` | 空状态组件 |
| `./SessionBrowser/utils.js` | `sortSessions`, `filterSessions` | 排序和过滤工具函数 |

### 外部依赖

| 包名 | 导入内容 | 用途 |
|------|----------|------|
| `react` | `React`（类型）, `useState`, `useCallback`, `useMemo`, `useEffect`, `useRef` | React 核心库 |
| `ink` | `Box`, `Text` | 终端 UI 框架 |
| `node:path` | `path` | Node.js 路径操作 |
| `@google/gemini-cli-core` | `Config`（类型） | 应用配置类型 |

## 关键实现细节

1. **集中状态管理模式**：所有组件状态（数据、UI、搜索、排序、分页）以及对应的 setter 函数都集中在 `SessionBrowserState` 接口中，通过 `useSessionBrowserState` 钩子统一创建。子组件和其他 Hooks 仅接收 `state` 对象，完全消除了多层 props drilling。

2. **双模式键盘输入**：键盘输入处理分为搜索模式和导航模式两套逻辑。在搜索模式下字母键用于输入搜索文本，在导航模式下字母键用于快捷操作（如 `g`/`G` 跳转、`s` 排序、`x` 删除）。Enter、方向键、翻页键在两种模式下均有效。

3. **延迟加载全文内容**：初始加载仅获取会话元数据（文件名、消息数、时间等），只有当用户进入搜索模式时才触发全文内容加载（`includeFullContent: true`），避免初始加载时读取过多数据。搜索退出时重置 `hasLoadedFullContent` 标志，下次搜索时重新加载以获取最新内容。

4. **less 风格快捷键**：导航快捷键设计借鉴了 Unix `less` 命令的习惯：`g` 到顶部、`G` 到底部、`u` 上翻半页、`d` 下翻半页、`/` 搜索、`q` 退出。这使得熟悉终端操作的用户可以快速上手。

5. **分页与滚动偏移**：使用 `scrollOffset` 和 `SESSIONS_PER_PAGE`（20）实现分页。`useMoveSelection` 钩子在移动选择时自动调整滚动偏移，确保选中项始终在可视范围内。表头和底部分别显示 `▲` 和 `▼` 滚动指示器。

6. **当前会话保护**：当前活跃会话（`isCurrentSession === true`）在列表中以灰色 dim 样式显示并附加 "(current)" 标签，按 Enter 不会触发恢复操作，按 `x` 也不会触发删除操作。

7. **搜索匹配高亮**：搜索结果通过 `MatchSnippetDisplay` 组件渲染，展示匹配片段的上下文文本和高亮的匹配文本。角色前缀以不同颜色区分（用户为绿色，Gemini 为蓝色），匹配文本以红色加粗显示。当有多个匹配时显示 "(+N more)" 提示。

8. **会话名称截断**：`SessionItem` 根据终端宽度动态计算可用显示宽度，将长会话名称截断并添加省略号 `...`。计算时考虑了固定列宽度（`FIXED_SESSION_COLUMNS_WIDTH = 30`）和附加信息标签的长度。

9. **响应式排序/过滤管线**：`filteredAndSortedSessions` 通过 `useMemo` 缓存，仅在 `sessions`、`searchQuery`、`sortOrder`、`sortReverse` 变化时重新计算。排序和过滤逻辑委托给独立的 `sortSessions` 和 `filterSessions` 工具函数。

10. **会话删除与状态同步**：删除操作通过 `onDeleteSession` 异步回调执行，成功后从 `sessions` 数组中移除对应项，并在选中项越界时自动调整 `activeIndex`。删除失败时在 `error` 中显示错误信息。
