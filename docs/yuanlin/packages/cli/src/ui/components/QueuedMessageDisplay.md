# QueuedMessageDisplay.tsx

## 概述

`QueuedMessageDisplay` 是一个 React (Ink) 组件，用于在终端 CLI 界面中展示用户排队等待发送的消息列表。当用户在 AI 正在处理当前请求时继续输入新消息，这些消息会被排入队列，并通过此组件以缩略预览的形式展示给用户。

该组件具备以下核心能力：
- 当消息队列为空时不渲染任何内容（返回 `null`）
- 最多显示 3 条排队消息的预览（由常量 `MAX_DISPLAYED_QUEUED_MESSAGES` 控制）
- 将多行/多空格消息压缩为单行预览显示
- 超出最大显示数量时，显示剩余消息计数提示（如 `... (+2 more)`）
- 提示用户可按上箭头键（↑）编辑排队消息

## 架构图（Mermaid）

```mermaid
graph TD
    A[QueuedMessageDisplay 组件] --> B{消息队列是否为空?}
    B -- 是 --> C[返回 null，不渲染]
    B -- 否 --> D[渲染消息队列容器]
    D --> E[显示标题提示文字<br/>"Queued press ↑ to edit"]
    D --> F[截取前3条消息]
    F --> G[遍历消息列表]
    G --> H[将空白字符替换为单空格<br/>生成消息预览]
    H --> I[渲染单条消息预览<br/>带 truncate 截断]
    D --> J{消息总数 > 3 ?}
    J -- 是 --> K[显示剩余数量提示<br/>"... +N more"]
    J -- 否 --> L[不显示额外提示]

    style A fill:#4A90D9,color:#fff
    style B fill:#F5A623,color:#fff
    style C fill:#D0021B,color:#fff
    style D fill:#7ED321,color:#fff
    style K fill:#BD10E0,color:#fff
```

## 核心组件

### 常量

| 常量名 | 值 | 说明 |
|--------|---|------|
| `MAX_DISPLAYED_QUEUED_MESSAGES` | `3` | 最多在 UI 中展示的排队消息条数 |

### 接口定义

```typescript
export interface QueuedMessageDisplayProps {
  messageQueue: string[];  // 排队等待发送的消息字符串数组
}
```

### QueuedMessageDisplay 组件

**类型**: React 函数组件 (`FC`)

**入参**:
- `messageQueue: string[]` — 排队消息数组，每个元素为用户输入的原始消息文本

**渲染逻辑**:

1. **空队列守卫**: 如果 `messageQueue.length === 0`，直接返回 `null`，不渲染任何 DOM 节点。

2. **容器布局**: 使用 `<Box flexDirection="column" marginTop={1}>` 作为纵向排列的外层容器，顶部留一行间距。

3. **标题行**: 显示灰色提示文字 `"Queued (press ↑ to edit):"`，左缩进 2 个字符。

4. **消息预览列表**:
   - 使用 `slice(0, MAX_DISPLAYED_QUEUED_MESSAGES)` 截取前 3 条消息
   - 对每条消息执行 `message.replace(/\s+/g, ' ')` 将所有连续空白字符（包括换行、制表符等）替换为单个空格，形成单行预览
   - 每条消息使用 `<Text dimColor wrap="truncate">` 渲染，超出终端宽度时截断显示
   - 左缩进 4 个字符，宽度为 `100%`

5. **溢出提示**: 当消息总数超过 `MAX_DISPLAYED_QUEUED_MESSAGES` 时，显示 `"... (+N more)"` 提示剩余未展示的消息数量。

## 依赖关系

### 内部依赖

无内部依赖。该组件是一个纯展示型叶子组件，不依赖项目中的其他模块。

### 外部依赖

| 依赖包 | 导入项 | 用途 |
|--------|--------|------|
| `ink` | `Box` | Ink 框架的布局容器组件，提供 Flexbox 布局能力 |
| `ink` | `Text` | Ink 框架的文本渲染组件，支持样式控制和文本截断 |

## 关键实现细节

1. **消息预览压缩策略**: 使用正则 `/\s+/g` 将所有空白字符（包括 `\n`、`\t`、连续空格等）统一替换为单个空格。这确保了多行消息在队列预览中以紧凑的单行形式展示，适合终端的有限空间。

2. **文本截断**: 通过 `wrap="truncate"` 属性，当消息预览超出终端宽度时自动截断而非换行，保证每条消息只占一行，维持列表的整洁性。

3. **灰色淡化显示**: 所有文本均使用 `dimColor` 属性渲染为灰色，表明这些是"次要信息"——排队消息尚未被处理，不应抢夺用户对当前交互的注意力。

4. **性能考量**: 使用 `slice(0, 3)` 限制渲染数量，即使消息队列很长也只渲染固定数量的 DOM 节点，避免终端渲染性能问题。

5. **用户交互提示**: 标题中包含 `"press ↑ to edit"` 提示，暗示上层组件实现了通过上箭头键编辑排队消息的交互逻辑，但该交互逻辑不在此组件中实现。

6. **Key 使用**: 列表渲染使用数组索引 `index` 作为 React key。由于排队消息列表是追加式的且不会重新排序，使用索引作为 key 在此场景下是可接受的。
