# RawMarkdownIndicator.tsx

## 概述

`RawMarkdownIndicator` 是一个 React (Ink) 组件，用于在终端 CLI 界面中显示"原始 Markdown 模式"的状态指示器。当用户切换到原始 Markdown 显示模式（即不渲染 Markdown 格式，直接显示原始标记符号）时，此组件会在界面中展示一个简洁的指示标签，同时告知用户可用的快捷键以切换回正常模式。

该组件的核心能力：
- 显示 "raw markdown mode" 文本标签
- 动态获取并显示切换 Markdown 模式的快捷键
- 使用语义化颜色区分主要文字和辅助提示文字

## 架构图（Mermaid）

```mermaid
graph TD
    A[RawMarkdownIndicator 组件] --> B[formatCommand 函数]
    B --> C[获取 TOGGLE_MARKDOWN<br/>命令的快捷键描述]
    A --> D[渲染 Box 容器]
    D --> E[主文本: "raw markdown mode"]
    D --> F[辅助提示文本<br/>"快捷键 to toggle"<br/>使用次要颜色]

    B --> G[Command.TOGGLE_MARKDOWN<br/>键绑定命令枚举]
    F --> H[theme.text.secondary<br/>语义化次要文字颜色]

    style A fill:#4A90D9,color:#fff
    style C fill:#F5A623,color:#fff
    style E fill:#7ED321,color:#fff
    style F fill:#BD10E0,color:#fff
```

## 核心组件

### RawMarkdownIndicator 组件

**类型**: `React.FC`（无 Props）

**内部逻辑**:

1. **获取快捷键描述**: 调用 `formatCommand(Command.TOGGLE_MARKDOWN)` 获取"切换 Markdown 模式"命令对应的快捷键的人类可读描述字符串（如 `"Ctrl+M"` 或 `"⌘M"`）。

2. **渲染结构**:
   ```
   ┌─────────────────────────────────────────┐
   │ raw markdown mode  (Ctrl+M to toggle)   │
   │ ← 主要文字色 →     ← 次要文字色 →       │
   └─────────────────────────────────────────┘
   ```
   - 外层 `<Box>`: 水平布局容器
   - 外层 `<Text>`: 包裹完整文字，使用默认主要颜色
     - `"raw markdown mode"`: 指示器主文本
     - 内层 `<Text color={theme.text.secondary}>`: 快捷键提示，使用语义化的次要文字颜色，形成视觉层次

## 依赖关系

### 内部依赖

| 模块路径 | 导入项 | 用途 |
|----------|--------|------|
| `../semantic-colors.js` | `theme` | 语义化颜色主题对象，提供 `theme.text.secondary` 次要文字颜色 |
| `../key/keybindingUtils.js` | `formatCommand` | 将命令枚举值转换为用户可读的快捷键描述字符串 |
| `../key/keyBindings.js` | `Command` | 键绑定命令枚举，包含 `TOGGLE_MARKDOWN` 等命令常量 |

### 外部依赖

| 依赖包 | 导入项 | 用途 |
|--------|--------|------|
| `react` | `React` (类型) | 仅导入类型定义，用于 `React.FC` 类型注解 |
| `ink` | `Box` | Ink 框架的布局容器组件 |
| `ink` | `Text` | Ink 框架的文本渲染组件，支持颜色属性和嵌套 |

## 关键实现细节

1. **动态快捷键显示**: 组件不硬编码快捷键文字，而是通过 `formatCommand(Command.TOGGLE_MARKDOWN)` 动态获取。这意味着如果用户自定义了键绑定，指示器会自动反映实际的快捷键配置，保证提示信息与实际行为一致。

2. **Text 嵌套实现颜色分段**: 组件使用 Ink 的 `<Text>` 嵌套特性来实现同一行内的颜色分段——外层 `<Text>` 使用默认颜色渲染 `"raw markdown mode"`，内层 `<Text>` 使用 `theme.text.secondary` 渲染快捷键提示。这是 Ink 框架中实现富文本效果的标准模式。

3. **视觉层次设计**: "raw markdown mode" 使用主要颜色（更醒目），快捷键提示使用次要颜色（较淡），形成明确的视觉层次。主要信息告诉用户当前处于什么模式，次要信息提供如何退出该模式的操作指引。

4. **无状态纯展示组件**: 该组件不接收任何 Props，也不管理任何状态。它的显示/隐藏完全由父组件根据当前 Markdown 模式状态来控制。组件本身只负责"怎么显示"，不负责"何时显示"。

5. **Command 枚举的使用**: 通过引用 `Command.TOGGLE_MARKDOWN` 枚举值而非直接使用字符串，确保了类型安全和重构友好性。如果命令名称变更，TypeScript 编译器会在所有引用处报错。

6. **简洁的组件设计**: 整个组件只有 10 行有效代码（不含导入和 license），体现了单一职责原则——仅负责渲染一个模式指示标签，不涉及任何业务逻辑或状态管理。
