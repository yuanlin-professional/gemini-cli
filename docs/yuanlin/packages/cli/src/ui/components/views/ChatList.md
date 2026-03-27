# ChatList.tsx

## 概述

`ChatList` 是一个 React (Ink) 函数组件，用于在 CLI 终端界面中展示**已保存的对话检查点（Conversation Checkpoints）列表**。每个对话项显示其名称和格式化后的修改时间，列表按时间顺序排列（旧的在前，新的在后）。

该组件是一个视图级组件，通常在用户执行查看历史对话的操作时作为主体内容渲染。

**文件路径**: `packages/cli/src/ui/components/views/ChatList.tsx`
**许可证**: Apache-2.0 (Copyright 2025 Google LLC)

## 架构图（Mermaid）

```mermaid
graph TD
    A[父组件] -->|chats 对话列表| B[ChatList 组件]

    B --> C{chats 是否为空?}
    C -->|是| D[显示 "No saved conversation checkpoints found."]
    C -->|否| E[渲染对话列表]

    E --> F[标题文本: "List of saved conversations:"]
    E --> G[遍历 chats 数组]
    E --> H[底部提示: "Note: Newest last, oldest first"]

    G --> I[单个对话项]
    I --> J[chat.mtime ISO 时间字符串]
    J --> K[正则解析]
    K --> L{解析成功?}
    L -->|是| M[格式化为 "YYYY-MM-DD HH:MM:SS"]
    L -->|否| N[显示 "Invalid Date"]

    I --> O[渲染: 名称 + 格式化时间]

    subgraph 时间格式化
        J
        K
        L
        M
        N
    end

    style B fill:#E3F2FD,stroke:#1976D2,color:#000
    style D fill:#FFCDD2,stroke:#D32F2F,color:#000
    style E fill:#C8E6C9,stroke:#388E3C,color:#000
    style M fill:#FFF9C4,stroke:#FBC02D,color:#000
```

## 核心组件

### ChatListProps 接口

| 属性 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `chats` | `readonly ChatDetail[]` | 是 | 只读的对话详情数组，包含所有已保存的对话检查点信息 |

### ChatDetail 类型（推断）

根据组件中的使用方式，`ChatDetail` 至少包含以下字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | 对话的名称标识，用作列表项的 key 和显示内容 |
| `mtime` | `string` | 对话的修改时间，ISO 8601 格式的时间字符串（如 `"2025-06-15T14:30:00Z"`） |

### 时间格式化逻辑

组件内部使用正则表达式解析 ISO 时间字符串：

```typescript
const match = isoString.match(/(\d{4}-\d{2}-\d{2})T(\d{2}:\d{2}:\d{2})/);
const formattedDate = match ? `${match[1]} ${match[2]}` : 'Invalid Date';
```

**正则分析**:
- `(\d{4}-\d{2}-\d{2})` - 捕获组1：日期部分（YYYY-MM-DD）
- `T` - ISO 8601 日期和时间的分隔符
- `(\d{2}:\d{2}:\d{2})` - 捕获组2：时间部分（HH:MM:SS）

**转换示例**:
| 输入 (ISO 格式) | 输出 (格式化) |
|-----------------|--------------|
| `2025-06-15T14:30:00Z` | `2025-06-15 14:30:00` |
| `2025-12-01T09:05:30.123Z` | `2025-12-01 09:05:30` |
| `invalid-string` | `Invalid Date` |

### 渲染结构

#### 空列表状态

```
Text "No saved conversation checkpoints found."
```

#### 有对话时的完整结构

```
Box (垂直布局)
├── Text "List of saved conversations:"
├── Box (height=1, 空行分隔)
├── Box (横向布局, key=chat1.name)
│   └── Text
│       ├── "  - "
│       ├── Text (强调色) "{chat.name}"
│       ├── " "
│       └── Text (次要色) "({formattedDate})"
├── Box (横向布局, key=chat2.name)
│   └── ... 同上结构
├── ... 更多对话项
├── Box (height=1, 空行分隔)
└── Text (dimColor, 次要色) "Note: Newest last, oldest first"
```

**终端渲染效果示意**:

```
List of saved conversations:

  - my-project-chat (2025-06-15 14:30:00)
  - debug-session (2025-06-16 09:15:30)
  - feature-discussion (2025-06-17 11:45:00)

Note: Newest last, oldest first
```

## 依赖关系

### 内部依赖

| 模块 | 导入内容 | 说明 |
|------|---------|------|
| `../../semantic-colors.js` | `theme` | 语义化颜色主题对象，使用 `theme.text.accent`（强调色，用于对话名称）和 `theme.text.secondary`（次要色，用于日期和提示文字） |
| `../../types.js` | `ChatDetail` (类型) | 对话详情的类型接口，描述单个对话检查点的元数据结构 |

### 外部依赖

| 包名 | 导入内容 | 说明 |
|------|---------|------|
| `react` | `React` (类型) | React 类型定义 |
| `ink` | `Box`, `Text` | Ink 框架的终端 UI 布局与文本渲染组件 |

## 关键实现细节

1. **只读数组类型**: Props 中 `chats` 使用 `readonly ChatDetail[]` 类型，这是 TypeScript 的不可变数组类型标注。它明确表示组件不会修改传入的数组，增强了类型安全性，也向调用者传达了组件的纯读取语义。

2. **手动时间格式化**: 组件没有使用 `Date` 对象或第三方日期库（如 dayjs、date-fns），而是直接用正则表达式从 ISO 字符串中提取日期和时间部分。这种方式的优点是：
   - 零依赖，不引入额外的日期处理库。
   - 不涉及时区转换，直接使用 ISO 字符串中的原始时间值。
   - 性能开销极小。
   - 缺点是不处理时区差异，显示的时间可能是 UTC 而非用户本地时间。

3. **Invalid Date 降级**: 当 ISO 字符串格式不符合预期（正则匹配失败）时，显示 "Invalid Date" 而非抛出异常。这种防御性编程确保了即使数据异常也不会导致组件崩溃。

4. **排序提示**: 组件底部显示 "Note: Newest last, oldest first" 提示文本，告知用户列表的排序方式。这意味着组件本身**不负责排序**，它假设传入的 `chats` 数组已经按时间升序排列。排序逻辑由上层父组件或数据提供方负责。

5. **列表项格式**: 每个对话项使用 `{'  '}- ` 格式的缩进前缀，与 `AgentsStatus` 组件的列表项格式保持一致，形成统一的 CLI 列表视觉风格。

6. **颜色层次**: 对话名称使用 `theme.text.accent`（强调色）突出显示，日期使用 `theme.text.secondary`（次要色）作为辅助信息，底部提示同样使用次要色，形成清晰的信息层次：名称 > 日期 > 提示。

7. **无状态纯组件**: 组件不包含任何内部状态、副作用或事件处理，是一个纯展示型组件。所有数据通过 Props 传入，使组件易于测试、预测和复用。

8. **空行分隔**: 使用 `<Box height={1} />` 在标题与列表之间、列表与底部提示之间插入空行，提供良好的视觉间距。这是 Ink 框架中常用的空行实现方式，比使用空 `Text` 组件更语义化。
