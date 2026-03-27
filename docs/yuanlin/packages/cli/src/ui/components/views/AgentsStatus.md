# AgentsStatus.tsx

## 概述

`AgentsStatus` 是一个 React (Ink) 函数组件，用于在 CLI 终端界面中展示**当前可用的 Agent（智能代理）列表及其状态信息**。组件将 Agent 按来源分为"本地 Agent"和"远程 Agent"两个分组进行渲染，每个 Agent 会显示其名称（支持 `displayName` 和 `name` 双重标识）以及 Markdown 格式的描述信息。

该组件是一个视图级组件（位于 `views/` 目录下），通常作为某个特定页面或面板的主体内容渲染。

**文件路径**: `packages/cli/src/ui/components/views/AgentsStatus.tsx`
**许可证**: Apache-2.0 (Copyright 2025 Google LLC)

## 架构图（Mermaid）

```mermaid
graph TD
    A[父组件] -->|agents 列表, terminalWidth| B[AgentsStatus 组件]

    B --> C{agents 是否为空?}
    C -->|是| D[显示 "No agents available."]
    C -->|否| E[过滤并分组]

    E --> F[localAgents 本地 Agent]
    E --> G[remoteAgents 远程 Agent]

    F --> H[renderAgentList 渲染本地 Agent 列表]
    G --> I[renderAgentList 渲染远程 Agent 列表]

    H --> J[遍历本地 Agent]
    I --> K[遍历远程 Agent]

    J --> L[单个 Agent 项]
    K --> L

    L --> M[Agent 名称显示]
    L --> N{是否有描述?}
    N -->|是| O[MarkdownDisplay 渲染描述]
    N -->|否| P[不渲染描述]

    M --> Q{displayName 与 name 不同?}
    Q -->|是| R[显示 displayName + name 括号标注]
    Q -->|否| S[仅显示 displayName 或 name]

    style B fill:#E3F2FD,stroke:#1976D2,color:#000
    style D fill:#FFCDD2,stroke:#D32F2F,color:#000
    style H fill:#C8E6C9,stroke:#388E3C,color:#000
    style I fill:#BBDEFB,stroke:#1976D2,color:#000
    style O fill:#FFF9C4,stroke:#FBC02D,color:#000
```

## 核心组件

### AgentsStatusProps 接口

| 属性 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `agents` | `AgentDefinitionJson[]` | 是 | Agent 定义数组，包含所有可用 Agent 的元信息 |
| `terminalWidth` | `number` | 是 | 终端宽度（字符数），传递给 `MarkdownDisplay` 控制 Markdown 渲染宽度 |

### AgentDefinitionJson 类型（推断）

根据组件中的使用方式，`AgentDefinitionJson` 至少包含以下字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | Agent 的唯一标识名称 |
| `displayName` | `string \| undefined` | Agent 的显示名称，用于用户界面展示 |
| `description` | `string \| undefined` | Agent 的描述文本，支持 Markdown 格式 |
| `kind` | `'local' \| 'remote'` | Agent 的类型：本地 Agent 或远程 Agent |

### 组件逻辑

#### 数据分组

```typescript
const localAgents = agents.filter((a) => a.kind === 'local');
const remoteAgents = agents.filter((a) => a.kind === 'remote');
```

组件在渲染前先将 `agents` 数组按 `kind` 字段分为两组：
- **localAgents**: 本地运行的 Agent
- **remoteAgents**: 远程服务的 Agent

#### renderAgentList 内部渲染函数

这是一个在组件内部定义的渲染辅助函数，接收分组标题和 Agent 列表，返回对应的 JSX 元素。

**参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `title` | `string` | 分组标题文本（如 "Local Agents" 或 "Remote Agents"） |
| `agentList` | `AgentDefinitionJson[]` | 该分组下的 Agent 列表 |

**渲染逻辑**:
- 如果列表为空，返回 `null` 不渲染。
- 否则渲染分组标题和 Agent 列表项。

### 渲染结构

#### 空列表状态

```
Box (垂直布局, marginBottom=1)
└── Text "No agents available."
```

#### 有 Agent 时的完整结构

```
Box (垂直布局, marginBottom=1)
├── [本地 Agent 分组]
│   Box (垂直布局)
│   ├── Text (加粗, 主要色) "Local Agents"
│   ├── Box (height=1, 空行分隔)
│   ├── Box (横向布局, key=agent1.name)
│   │   ├── Text "  - "
│   │   └── Box (垂直布局)
│   │       ├── Text (加粗, 强调色) "显示名称 (标识名称)"
│   │       └── [条件] MarkdownDisplay (Agent 描述)
│   └── ... 更多本地 Agent 项
│
├── [条件: 同时有本地和远程 Agent]
│   Box (height=1, 空行分隔两组)
│
└── [远程 Agent 分组]
    Box (垂直布局)
    ├── Text (加粗, 主要色) "Remote Agents"
    ├── Box (height=1, 空行分隔)
    └── ... Agent 列表项（结构同上）
```

#### Agent 名称显示逻辑

```
如果 displayName 存在:
  显示 displayName
  如果 displayName !== name:
    额外显示 " (name)"  ← 括号中显示标识名称
否则:
  显示 name
```

**示例**:
- `displayName="Code Assistant"`, `name="code-assistant"` --> 显示: **Code Assistant** (code-assistant)
- `displayName="code-assistant"`, `name="code-assistant"` --> 显示: **code-assistant**
- `displayName` 不存在, `name="my-agent"` --> 显示: **my-agent**

## 依赖关系

### 内部依赖

| 模块 | 导入内容 | 说明 |
|------|---------|------|
| `../../semantic-colors.js` | `theme` | 语义化颜色主题对象，使用 `theme.text.primary`（主要文字色）和 `theme.text.accent`（强调色） |
| `../../types.js` | `AgentDefinitionJson` (类型) | Agent 定义的 JSON 类型接口，描述 Agent 的元数据结构 |
| `../../utils/MarkdownDisplay.js` | `MarkdownDisplay` | Markdown 渲染组件，用于将 Agent 描述中的 Markdown 文本渲染为终端格式化输出 |

### 外部依赖

| 包名 | 导入内容 | 说明 |
|------|---------|------|
| `react` | `React` (类型) | React 类型定义 |
| `ink` | `Box`, `Text` | Ink 框架的终端 UI 布局与文本渲染组件 |

## 关键实现细节

1. **Agent 分组策略**: 组件通过 `kind` 字段将 Agent 分为 `local` 和 `remote` 两组。这种分组方式使用户能快速识别 Agent 的部署位置。两组之间通过 `<Box height={1} />` 添加一行空白作为视觉分隔，但仅当两组都有内容时才添加。

2. **双重名称显示**: Agent 名称的显示采用 `displayName` 优先策略。当 `displayName` 存在且与 `name` 不同时，会同时显示两者（`displayName (name)` 格式），这样用户既能看到友好的显示名称，也能知道用于命令调用的标识名称。

3. **Markdown 描述渲染**: Agent 的 `description` 字段通过 `MarkdownDisplay` 组件渲染，支持 Markdown 格式（如加粗、链接、列表等）。`terminalWidth` 参数确保 Markdown 内容不会超出终端宽度。`isPending={false}` 表示内容已完全加载，无需显示加载状态。

4. **空状态处理**: 当没有任何 Agent 时，组件不会渲染空白区域，而是显示明确的 "No agents available." 提示文本。`renderAgentList` 内部也对空列表做了处理（返回 `null`），确保不会渲染空的分组标题。

5. **列表项缩进**: 每个 Agent 项前使用 `{'  '}- ` 格式进行缩进，模拟了 Markdown 无序列表的视觉效果。缩进部分使用 `theme.text.primary` 颜色，而 Agent 名称使用 `theme.text.accent` 强调色突出显示。

6. **无状态设计**: 组件不包含任何内部状态或副作用，是一个纯渲染组件。所有数据（Agent 列表和终端宽度）均通过 Props 传入，使组件易于测试和复用。

7. **分组间距控制**: 通过条件判断 `localAgents.length > 0 && remoteAgents.length > 0` 来决定是否在两组之间插入空行，避免了只有一组时出现多余的空白。
