# memoryCommand.ts

## 概述

`memoryCommand.ts` 实现了 `/memory` 斜杠命令及其子命令，用于管理 Gemini CLI 的"记忆"系统。记忆系统基于 `GEMINI.md` 文件，允许用户查看、添加、重新加载记忆内容，以及列出当前使用中的记忆文件路径。

该命令是一个父命令（`autoExecute: false`），包含 4 个子命令：`show`、`add`、`reload`、`list`。与 `mcpCommand` 不同，它没有定义默认 action，因此用户必须显式选择一个子命令。命令的核心业务逻辑全部委托给 `@google/gemini-cli-core` 包中的函数，CLI 层仅负责参数传递和 UI 展示。

## 架构图（Mermaid）

```mermaid
flowchart TD
    A["/memory 父命令"] --> B["/memory show 展示记忆内容"]
    A --> C["/memory add 添加记忆内容"]
    A --> D["/memory reload 重新加载记忆"]
    A --> E["/memory list 列出记忆文件路径"]

    B --> F[调用 showMemory 获取内容]
    F --> G[通过 UI 展示 INFO 消息]

    C --> H[调用 addMemory 处理添加]
    H --> I{返回类型?}
    I -- message --> J[直接返回消息给调用方]
    I -- 其他 --> K[通过 UI 展示提示]
    K --> L[返回结果给调用方]

    D --> M[展示"正在重新加载"提示]
    M --> N[调用 refreshMemory 重新加载]
    N --> O{是否成功?}
    O -- 是 --> P[展示刷新结果]
    O -- 否 --> Q[展示错误消息]

    E --> R[调用 listMemoryFiles 获取路径]
    R --> S[通过 UI 展示文件路径列表]

    subgraph 核心库 "@google/gemini-cli-core"
        F2[showMemory]
        H2[addMemory]
        N2[refreshMemory]
        R2[listMemoryFiles]
    end

    F -.-> F2
    H -.-> H2
    N -.-> N2
    R -.-> R2

    style A fill:#4CAF50,color:#fff
    style B fill:#2196F3,color:#fff
    style C fill:#FF9800,color:#fff
    style D fill:#9C27B0,color:#fff
    style E fill:#00BCD4,color:#fff
    style Q fill:#f44336,color:#fff
```

## 核心组件

### 1. `memoryCommand`（父命令）

导出的主命令对象，类型为 `SlashCommand`。

| 属性 | 值 | 说明 |
|------|------|------|
| `name` | `'memory'` | 命令名称，用户通过 `/memory` 调用 |
| `description` | `'Commands for interacting with memory'` | 命令描述 |
| `kind` | `CommandKind.BUILT_IN` | 内置命令 |
| `autoExecute` | `false` | 不自动执行，需要用户选择子命令 |
| `subCommands` | 4 个子命令数组 | 包含 show、add、reload、list |

注意：父命令没有定义 `action` 属性，这意味着直接执行 `/memory` 不会有默认行为，必须指定子命令。

### 2. `show` 子命令

用于展示当前记忆的完整内容。

| 属性 | 值 |
|------|------|
| `name` | `'show'` |
| `autoExecute` | `true` |

**执行流程**：
1. 从上下文获取 `config` 配置对象，若不存在则静默返回
2. 调用核心库的 `showMemory(config)` 获取记忆内容
3. 通过 `context.ui.addItem` 以 `INFO` 类型展示内容

### 3. `add` 子命令

用于向记忆中添加新内容。这是唯一一个 `autoExecute: false` 的子命令，因为需要用户输入要添加的内容。

| 属性 | 值 |
|------|------|
| `name` | `'add'` |
| `autoExecute` | `false` |

**执行流程**：
1. 调用核心库的 `addMemory(args)` 处理添加逻辑
2. 如果返回类型为 `'message'`，直接返回结果（可能是错误消息，如参数为空）
3. 否则通过 UI 展示"正在尝试保存"的提示，并返回结果

**注意**：`add` 子命令的 `action` 是同步函数（非 async），返回类型为 `SlashCommandActionReturn | void`。这与其他子命令不同，暗示 `addMemory` 是一个同步操作，可能返回 `submit_prompt` 类型的结果让 AI 来实际执行保存。

### 4. `reload` 子命令

用于从源文件重新加载记忆内容。

| 属性 | 值 |
|------|------|
| `name` | `'reload'` |
| `altNames` | `['refresh']` |
| `autoExecute` | `true` |

**执行流程**：
1. 先通过 UI 展示"正在重新加载"的提示信息
2. 获取 `config` 配置对象
3. 调用核心库的异步函数 `refreshMemory(config)` 重新加载
4. 成功时展示刷新结果
5. 失败时捕获异常并展示错误消息

**特殊点**：这是唯一一个包含 `try-catch` 错误处理的子命令，说明 `refreshMemory` 可能因文件系统操作失败而抛出异常。

### 5. `list` 子命令

用于列出当前正在使用的所有 `GEMINI.md` 文件路径。

| 属性 | 值 |
|------|------|
| `name` | `'list'` |
| `autoExecute` | `true` |

**执行流程**：
1. 从上下文获取 `config`，若不存在则静默返回
2. 调用核心库的 `listMemoryFiles(config)` 获取文件路径列表
3. 通过 UI 展示路径信息

## 依赖关系

### 内部依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `./types.js` | `CommandKind`, `SlashCommand`, `SlashCommandActionReturn` | 命令类型定义和枚举 |
| `../types.js` | `MessageType` | UI 消息类型枚举（`INFO`、`ERROR`） |

### 外部依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `@google/gemini-cli-core` | `addMemory` | 添加记忆内容的核心逻辑 |
| `@google/gemini-cli-core` | `listMemoryFiles` | 列出记忆文件路径的核心逻辑 |
| `@google/gemini-cli-core` | `refreshMemory` | 从源文件重新加载记忆的核心逻辑（异步） |
| `@google/gemini-cli-core` | `showMemory` | 展示当前记忆内容的核心逻辑 |

## 关键实现细节

1. **薄 CLI 层设计**：`memoryCommand` 是"薄客户端"模式的典型示例。所有业务逻辑（`showMemory`、`addMemory`、`refreshMemory`、`listMemoryFiles`）都在 `@google/gemini-cli-core` 中实现，CLI 层仅负责：
   - 从上下文中提取 `config` 对象
   - 调用核心函数
   - 将结果通过 UI 展示

2. **`add` 命令的双重返回路径**：`addMemory` 函数可能返回两种类型的结果：
   - `type: 'message'` -- 表示直接向用户展示的消息（如参数校验失败），此时直接返回
   - 其他类型（可能是 `submit_prompt`） -- 表示需要 AI 介入来完成保存操作，CLI 层展示提示后将结果传递给上层调度器

3. **配置缺失的静默处理**：`show` 和 `list` 子命令在配置不可用时直接 `return` 而不返回错误消息。这与 `mcpCommand` 中返回明确错误消息的做法不同，是一种更宽松的处理方式。

4. **`reload` 的错误处理**：`reload` 子命令是唯一使用 `try-catch` 的子命令，它将异常转换为 `MessageType.ERROR` 类型的 UI 消息。类型断言 `(error as Error).message` 使用了 ESLint 抑制注释。

5. **`autoExecute` 的差异化设置**：
   - `show`、`reload`、`list` 设为 `true`，因为它们是只读或自动操作
   - `add` 设为 `false`，因为需要用户提供要添加的内容参数

6. **`Date.now()` 时间戳**：所有 `context.ui.addItem` 调用都传入 `Date.now()` 作为第二个参数，用于在 UI 历史记录中标记消息的时间顺序。

7. **记忆系统与 `GEMINI.md` 的关系**：从 `list` 子命令的描述（"Lists the paths of the GEMINI.md files in use"）可以看出，记忆系统的底层存储就是 `GEMINI.md` 文件，可能包含项目级和全局级的多个文件。这与 `initCommand` 中创建 `GEMINI.md` 的功能形成互补。
