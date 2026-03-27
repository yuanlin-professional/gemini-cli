# ThinkingMessage.tsx

## 概述

`ThinkingMessage` 是一个 React（Ink）组件，用于在终端 CLI 界面中以 **气泡样式** 渲染模型的"思考过程"（Thinking）。它将模型的思考摘要（`ThoughtSummary`）解析为标题行和描述行，以带有左侧边框的缩进块形式呈现，视觉上类似于引用块（blockquote）。

该文件还包含一个内部辅助函数 `normalizeThoughtLines`，负责对思考内容进行规范化处理，过滤噪声文本并分割为行数组。

**文件路径**: `packages/cli/src/ui/components/messages/ThinkingMessage.tsx`

## 架构图（Mermaid）

```mermaid
graph TD
    A[ThinkingMessage 组件] --> B[normalizeThoughtLines 处理思考内容]
    B --> C[normalizeEscapedNewlines 处理转义换行符]
    C --> D[subject 主题文本处理]
    C --> E[description 描述文本处理]
    D --> F{isNoise 噪声检测}
    E --> G[按换行符分割为多行]
    G --> H{逐行 isNoise 噪声过滤}
    F -->|非噪声| I[加入 lines 数组]
    H -->|非噪声| I
    F -->|是噪声| J[丢弃]
    H -->|是噪声| J

    A --> K{fullLines 是否为空?}
    K -->|是| L[返回 null 不渲染]
    K -->|否| M[渲染气泡容器]

    M --> N{isFirstThinking?}
    N -->|是| O[显示 "Thinking..." 标题]
    N -->|否| P[跳过标题]

    M --> Q[左边框引用块容器]
    Q --> R[第一行: 加粗斜体主题]
    Q --> S[后续行: 次要色斜体描述]
```

## 核心组件

### 1. `ThinkingMessage` 组件

**类型**: `React.FC<ThinkingMessageProps>`

**Props 接口**:

```typescript
interface ThinkingMessageProps {
  thought: ThoughtSummary;     // 思考摘要对象，包含 subject 和 description
  terminalWidth: number;       // 终端宽度，用于设置容器宽度
  isFirstThinking?: boolean;   // 是否为当前响应中的第一个思考，决定是否显示 "Thinking..." 标题
}
```

**渲染结构**:

| 区域 | 描述 | 条件 |
|------|------|------|
| **"Thinking..." 标题** | 斜体主色文本，标识进入思考模式 | 仅当 `isFirstThinking` 为 `true` 时 |
| **气泡容器** | 带有左侧单线边框的缩进块，类似 Markdown 引用 | 始终显示（内容非空时） |
| **第一行（主题）** | 加粗斜体，主色文本 | `fullLines[0]` 存在时 |
| **后续行（描述）** | 斜体，次要色文本 | `fullLines.length > 1` 时 |

**气泡样式细节**:

- 左侧留 `THINKING_LEFT_PADDING = 1` 的外边距
- 内容左侧留 1 个字符的内边距
- 使用 `borderStyle="single"` 仅显示左边框（`borderLeft=true`，其余三边为 `false`）
- 边框颜色使用 `theme.text.secondary`（次要文本色）
- 顶部有一个空行作为视觉间距

**空内容处理**: 如果 `normalizeThoughtLines` 处理后返回空数组，组件直接返回 `null`，不渲染任何内容。

### 2. `normalizeThoughtLines` 函数

**签名**: `(thought: ThoughtSummary) => string[]`

**功能**: 将 `ThoughtSummary` 对象的 `subject` 和 `description` 字段处理为干净的文本行数组。

**处理流程**:

1. 对 `subject` 和 `description` 分别调用 `normalizeEscapedNewlines` 处理转义换行符，然后 `trim` 去除首尾空白。
2. 使用 `isNoise` 函数检测噪声文本（空字符串或仅由点号组成的字符串，如 `"..."`, `"."`）。
3. 如果 `subject` 非空且非噪声，将其作为第一行加入结果。
4. 将 `description` 按换行符分割，逐行 trim 并过滤噪声行，将有效行追加到结果数组。

### 3. `isNoise` 内部函数

**签名**: `(text: string) => boolean`

**功能**: 判断文本是否为"噪声"。满足以下条件之一即为噪声：
- trim 后为空字符串
- 仅由点号（`.`）组成，如 `"."`, `"..."`, `"....."` 等

### 4. 常量

| 常量名 | 值 | 用途 |
|--------|-----|------|
| `THINKING_LEFT_PADDING` | `1` | 思考气泡容器的左侧外边距字符数 |

## 依赖关系

### 内部依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `../../semantic-colors.js` | `theme` | 语义化颜色主题对象，用于文本和边框着色 |
| `../../utils/textUtils.js` | `normalizeEscapedNewlines` | 将字符串中的转义换行符（如 `\\n`）转换为实际换行符 |
| `@google/gemini-cli-core` | `ThoughtSummary` (类型) | 思考摘要的 TypeScript 类型定义，包含 `subject` 和 `description` 字段 |

### 外部依赖

| 包名 | 导入内容 | 用途 |
|------|----------|------|
| `react` | `React` (类型), `useMemo` | React 类型支持和记忆化 Hook |
| `ink` | `Box`, `Text` | Ink 终端 UI 基础布局和文本组件 |

## 关键实现细节

1. **useMemo 性能优化**: `normalizeThoughtLines` 的结果通过 `useMemo` 缓存，以 `thought` 对象作为依赖。只有当 `thought` 引用变化时才会重新计算，避免每次渲染都进行字符串解析和过滤操作。

2. **噪声过滤机制**: `isNoise` 函数的正则 `/^\.+$/` 专门处理模型输出中常见的省略号占位符文本（如 `"..."`），这些通常是模型在思考时产生的无意义填充内容。

3. **视觉层次区分**: 第一行（通常是主题/标题）使用加粗斜体和主色（`theme.text.primary`），后续描述行使用普通斜体和次要色（`theme.text.secondary`），形成清晰的视觉层次。

4. **左边框引用块设计**: 通过 Ink 的 `borderStyle="single"` 并仅启用 `borderLeft`，模拟了终端中常见的引用块样式。这种设计让思考内容在视觉上与普通消息明确区分，用户可以快速识别出这是模型的内部推理过程而非最终回答。

5. **首次思考标识**: `isFirstThinking` 属性允许父组件控制是否在思考气泡上方显示 "Thinking..." 提示。这通常只在当前模型响应中第一个思考块出现时设置为 `true`，避免在连续多个思考块中重复显示。

6. **空行间距**: 气泡内顶部的 `<Text> </Text>` （包含一个空格）作为视觉间隔，使得内容不会紧贴边框顶部，提升可读性。

7. **转义换行符处理**: 模型返回的思考内容中可能包含字面量的 `\n` 字符串（而非实际换行符），`normalizeEscapedNewlines` 工具函数将其转换为真正的换行符，确保后续的 `split('\n')` 能正确分行。
