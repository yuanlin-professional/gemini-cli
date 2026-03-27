# jsonoutput.ts

## 概述

`jsonoutput.ts` 是一个轻量级的 JSON 输入验证与解析工具模块。它提供两个核心函数：`checkInput` 用于预检输入字符串是否可能为有效的 JSON 对象/数组，`tryParseJSON` 用于安全地尝试解析 JSON 字符串。该模块在解析前会过滤掉空值、空字符串、非 JSON 格式的文本以及包含 ANSI 转义序列的字符串，确保只有干净、有效的 JSON 数据才能通过验证。

文件路径：`packages/cli/src/utils/jsonoutput.ts`

## 架构图（Mermaid）

```mermaid
flowchart TD
    A[外部调用 tryParseJSON] --> B[调用 checkInput 预校验]
    B --> C{输入是否为 null/undefined?}
    C -- 是 --> D[返回 false]
    C -- 否 --> E{去除空白后是否为空?}
    E -- 是 --> D
    E -- 否 --> F{是否以 '{' 或 '[' 开头?}
    F -- 否 --> D
    F -- 是 --> G{是否包含 ANSI 转义序列?}
    G -- 是 --> D
    G -- 否 --> H[返回 true - 通过预校验]
    H --> I[JSON.parse 解析]
    I --> J{解析是否成功?}
    J -- 否 --> K[返回 null]
    J -- 是 --> L{结果是否为对象类型?}
    L -- 否 --> K
    L -- 是 --> M{是否为空数组?}
    M -- 是 --> K
    M -- 否 --> N{是否为空对象?}
    N -- 是 --> K
    N -- 否 --> O[返回解析后的对象]
```

## 核心组件

### 1. `checkInput()` 函数

```typescript
export function checkInput(input: string | null | undefined): boolean
```

**功能**：对输入字符串进行多层预检，判断其是否可能是有效的 JSON 对象或数组。

**参数**：
- `input: string | null | undefined` — 待检测的输入字符串

**返回值**：`boolean` — 通过预检返回 `true`，否则返回 `false`

**验证步骤**（按顺序）：

| 步骤 | 检查内容 | 不通过则返回 |
|------|----------|-------------|
| 1 | `input` 是否为 `null` 或 `undefined` | `false` |
| 2 | `input.trim()` 后是否为空字符串 | `false` |
| 3 | 去空白后是否以 `{` 或 `[` 开头（正则 `^(?:\[|\{)`） | `false` |
| 4 | 是否包含 ANSI 转义序列（通过 `stripAnsi` 比较） | `false` |

**ANSI 检测原理**：使用 `strip-ansi` 库移除所有 ANSI 转义码后与原字符串比较，如果不相等说明原字符串包含 ANSI 序列（如终端颜色代码），这类字符串不是纯 JSON。

### 2. `tryParseJSON()` 函数

```typescript
export function tryParseJSON(input: string): object | null
```

**功能**：安全地尝试将输入字符串解析为 JSON 对象。不会抛出异常。

**参数**：
- `input: string` — 待解析的 JSON 字符串

**返回值**：`object | null` — 解析成功返回 JSON 对象/数组，失败返回 `null`

**处理逻辑**：

1. 先调用 `checkInput` 进行预校验，不通过直接返回 `null`
2. 对输入执行 `trim()` 去除首尾空白
3. 调用 `JSON.parse()` 尝试解析
4. 解析成功后还需通过以下额外验证：
   - 解析结果不能是 `null`
   - 解析结果必须是 `object` 类型（排除原始类型如数字、字符串等）
   - 如果是数组，不能为空数组 `[]`
   - 如果是对象，不能为空对象 `{}`
5. 全部通过后返回解析结果；任何步骤失败或 `JSON.parse` 抛出异常都返回 `null`

## 依赖关系

### 内部依赖

无。本模块不依赖任何项目内部模块。

### 外部依赖

| 模块 | 用途 |
|------|------|
| `strip-ansi` | 第三方库，用于移除字符串中的 ANSI 转义序列。在 `checkInput` 中用于检测输入是否包含终端颜色/样式代码 |

## 关键实现细节

### 防御性编程策略

该模块采用了严格的防御性编程模式：

1. **空值保护**：显式检查 `null` 和 `undefined`，避免后续操作报错
2. **格式预检**：通过正则 `^(?:\[|\{)` 快速排除非 JSON 格式的字符串，避免不必要的 `JSON.parse` 调用
3. **ANSI 过滤**：终端输出可能混入 ANSI 转义序列，这些不可见字符会导致 JSON 解析失败或产生错误结果，因此提前过滤
4. **空值结果排除**：即使 `JSON.parse` 成功，空数组 `[]` 和空对象 `{}` 也会被视为无效返回 `null`，这表明模块的设计意图是只接受有实际内容的 JSON 数据
5. **异常安全**：`JSON.parse` 的异常被静默捕获并返回 `null`，保证函数永远不会抛出异常

### 为什么要过滤 ANSI 序列

在 CLI 环境中，程序的标准输出可能包含终端样式控制字符（如颜色代码 `\x1b[31m`）。当 CLI 尝试解析来自子进程或管道的输出时，这些不可见字符会破坏 JSON 格式。`checkInput` 通过提前检测并拒绝含有 ANSI 序列的输入，避免了这类问题。

### 设计意图

从函数签名和过滤逻辑来看，该模块主要用于处理来自命令行输入或外部进程输出的 JSON 数据。它被设计为"宁可拒绝，不可误判"——任何存疑的输入都会返回 `null`，将错误处理权交给调用方。
