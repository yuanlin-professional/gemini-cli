# prompt-processors 目录

## 概述

`prompt-processors` 目录实现了 Gemini CLI 的**提示词处理管道（Prompt Pipeline）**。它定义了一套可链式组合的处理器接口 `IPromptProcessor`，每个处理器对提示词执行特定的转换操作。提示词在发送给模型之前，依次经过管道中的各处理器，完成参数注入、Shell 命令执行替换、`@` 文件引用展开等预处理操作。该设计采用**管道模式（Pipeline Pattern）**，各处理器职责单一、可组合、可独立测试。

## 目录结构

```
prompt-processors/
├── types.ts                # 核心接口与常量定义
├── argumentProcessor.ts    # 参数注入处理器
├── shellProcessor.ts       # Shell 命令注入处理器 (!{...})
├── atFileProcessor.ts      # 文件引用注入处理器 (@{...})
├── injectionParser.ts      # 通用注入解析器（花括号匹配）
└── *.test.ts               # 对应测试文件
```

## 架构图

```mermaid
graph TD
    用户输入["用户输入提示词"]

    用户输入 --> 管道["提示词处理管道"]

    管道 --> 参数处理["DefaultArgumentProcessor<br/>参数注入"]
    参数处理 --> Shell处理["ShellProcessor<br/>Shell 命令注入 !{...}"]
    Shell处理 --> 文件处理["AtFileProcessor<br/>文件引用注入 @{...}"]

    Shell处理 -->|调用| 解析器["injectionParser<br/>extractInjections()"]
    文件处理 -->|调用| 解析器

    文件处理 --> 最终提示词["处理后的<br/>PromptPipelineContent"]
    最终提示词 --> 模型["发送给 AI 模型"]
```

## 核心组件

### 1. types.ts - 接口与常量

#### IPromptProcessor 接口

```typescript
interface IPromptProcessor {
  process(
    prompt: PromptPipelineContent,
    context: CommandContext,
  ): Promise<PromptPipelineContent>;
}
```

- **`PromptPipelineContent`**: 类型为 `PartUnion[]`（来自 `@google/genai`），支持文本和多模态内容部分。
- **`CommandContext`**: 提供命令调用详情（原始输入、参数）、应用服务和 UI 处理器。

#### 常量

| 常量 | 值 | 说明 |
|---|---|---|
| `SHORTHAND_ARGS_PLACEHOLDER` | `{{args}}` | 参数注入占位符 |
| `SHELL_INJECTION_TRIGGER` | `!{` | Shell 命令注入触发器 |
| `AT_FILE_INJECTION_TRIGGER` | `@{` | 文件引用注入触发器 |

### 2. DefaultArgumentProcessor - 参数注入处理器

- 当提示词中**不包含** `{{args}}` 时生效。
- 将用户的完整命令调用（`context.invocation.raw`）追加到提示词末尾。
- 允许模型自行解析参数。
- 若提示词已包含 `{{args}}`，则此处理器跳过，由其他处理器进行显式替换。

### 3. ShellProcessor - Shell 命令注入处理器

- 处理 `!{command}` 语法：提取花括号内的 Shell 命令，执行后将标准输出替换回提示词。
- 当 `{{args}}` 出现在 `!{...}` 内部时，参数会经过 Shell 转义处理以防止注入攻击。
- 当 `{{args}}` 出现在 `!{...}` 外部时，参数原样注入。

### 4. AtFileProcessor - 文件引用注入处理器

- 处理 `@{path}` 语法：解析花括号内的文件路径，读取文件内容后替换回提示词。
- 支持相对路径和绝对路径。

### 5. injectionParser - 通用注入解析器

- **`extractInjections(prompt, trigger, contextName)`**: 核心解析函数，支持任意触发器前缀。
- **嵌套花括号支持**: 通过花括号计数正确处理嵌套结构（如 JSON 片段）。
- **严格验证**: 未闭合的花括号会抛出带上下文信息的异常。
- 返回 `Injection[]`，每个元素包含 `content`（提取的内容）、`startIndex`、`endIndex`。

## 依赖关系

```mermaid
graph LR
    argumentProcessor.ts --> types.ts
    shellProcessor.ts --> types.ts
    atFileProcessor.ts --> types.ts

    shellProcessor.ts --> injectionParser.ts
    atFileProcessor.ts --> injectionParser.ts

    argumentProcessor.ts --> core["@google/gemini-cli-core<br/>(appendToLastTextPart)"]
    types.ts --> genai["@google/genai<br/>(PartUnion)"]
    types.ts --> uiTypes["../../ui/commands/types.ts<br/>(CommandContext)"]
```

## 数据流

```mermaid
sequenceDiagram
    participant 用户 as 用户输入
    participant 管道 as 处理管道
    participant 参数 as DefaultArgumentProcessor
    participant Shell as ShellProcessor
    participant 文件 as AtFileProcessor
    participant 解析 as injectionParser
    participant 系统 as 系统 Shell / 文件系统
    participant 模型 as AI 模型

    用户->>管道: "/命令 参数 !{ls -la} @{README.md}"

    管道->>参数: process(prompt, context)
    Note over 参数: 检测到 {{args}}?<br/>否 -> 追加原始输入<br/>是 -> 跳过
    参数-->>管道: 处理后的 prompt

    管道->>Shell: process(prompt, context)
    Shell->>解析: extractInjections(prompt, "!{")
    解析-->>Shell: [{content: "ls -la", ...}]
    Shell->>系统: 执行 "ls -la"
    系统-->>Shell: 命令输出
    Shell->>Shell: 替换 !{ls -la} 为输出
    Shell-->>管道: 处理后的 prompt

    管道->>文件: process(prompt, context)
    文件->>解析: extractInjections(prompt, "@{")
    解析-->>文件: [{content: "README.md", ...}]
    文件->>系统: 读取 README.md
    系统-->>文件: 文件内容
    文件->>文件: 替换 @{README.md} 为内容
    文件-->>管道: 最终 prompt

    管道->>模型: 发送处理后的 PromptPipelineContent
```
