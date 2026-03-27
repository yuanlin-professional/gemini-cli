# commands.ts

## 概述

`commands.ts` 是 Gemini CLI 的**斜杠命令解析器**模块。它实现了将用户在终端输入的原始斜杠命令字符串（如 `/memory add some data`）解析为结构化的命令对象、参数和规范路径。解析器支持多级子命令树遍历、主名称匹配和别名匹配两级查找策略。

**文件路径**: `packages/cli/src/utils/commands.ts`
**许可证**: Apache-2.0 (Copyright 2025 Google LLC)

## 架构图（Mermaid）

```mermaid
graph TD
    A[用户输入原始字符串] -->|例: "/memory add some data"| B[parseSlashCommand]

    B --> C[去除前导 / 并按空格分词]
    C -->|"memory", "add", "some", "data"| D[命令路径遍历循环]

    D --> E{在当前命令列表中查找}
    E -->|第一轮: 主名称匹配| F{找到?}
    F -->|否| G{第二轮: 别名匹配}
    G -->|找到| H[记录匹配命令]
    G -->|未找到| I[退出循环]
    F -->|是| H

    H --> J{有子命令?}
    J -->|是| K[切换到子命令列表, 继续遍历]
    J -->|否| I
    K --> E

    I --> L[计算剩余部分为参数 args]
    L --> M[返回 ParsedSlashCommand]

    M -->|commandToExecute| N[匹配的 SlashCommand 或 undefined]
    M -->|args| O["剩余参数字符串: 'some data'"]
    M -->|canonicalPath| P["规范路径: ['memory', 'add']"]

    style B fill:#4A90D9,color:#fff
    style E fill:#F5A623,color:#fff
    style M fill:#7ED321,color:#fff
```

## 核心组件

### `ParsedSlashCommand` 类型

```typescript
export type ParsedSlashCommand = {
  commandToExecute: SlashCommand | undefined;
  args: string;
  canonicalPath: string[];
};
```

斜杠命令解析的返回结果类型。

| 字段 | 类型 | 描述 |
|------|------|------|
| `commandToExecute` | `SlashCommand \| undefined` | 解析匹配到的最深层命令对象。如果未找到有效命令则为 `undefined` |
| `args` | `string` | 命令路径之后的剩余部分，作为参数传递给命令的 `action` 函数 |
| `canonicalPath` | `string[]` | 命令的规范路径，由匹配命令的主名称（非别名）组成的数组。例如 `['memory', 'add']` |

---

### `parseSlashCommand` 函数

```typescript
export const parseSlashCommand = (
  query: string,
  commands: readonly SlashCommand[],
): ParsedSlashCommand
```

#### 参数说明

| 参数 | 类型 | 描述 |
|------|------|------|
| `query` | `string` | 用户输入的原始字符串，以 `/` 开头，例如 `"/memory add some data"` |
| `commands` | `readonly SlashCommand[]` | 可用的顶级斜杠命令列表（只读数组） |

#### 解析算法详解

1. **预处理**：对输入进行 `trim()`，去除首字符 `/`，然后按空白字符 `\s+` 分词，过滤空字符串得到 `commandPath` 数组。
   - 输入 `"/memory  add  some data"` -> `["memory", "add", "some", "data"]`

2. **命令树遍历**：从顶级命令列表开始，逐级向下匹配：
   - **第一轮查找（主名称匹配）**：在当前命令列表中用 `find()` 查找 `cmd.name === part`
   - **第二轮查找（别名匹配）**：如果第一轮未找到，查找 `cmd.altNames?.includes(part)`
   - 找到命令后：
     - 更新 `commandToExecute` 为当前找到的命令
     - 将命令的**主名称**（`foundCommand.name`）加入 `canonicalPath`（即使匹配的是别名）
     - 递增 `pathIndex`
     - 如果命令有 `subCommands`，切换搜索范围到子命令列表，继续遍历
     - 如果没有子命令，退出循环
   - 未找到命令：退出循环

3. **参数提取**：从 `parts` 数组的 `pathIndex` 位置开始的所有元素用空格拼接为 `args` 字符串

#### 使用示例

```typescript
// 假设命令树：
// /memory (顶级)
//   └── add (子命令)
//   └── list (子命令)

const result = parseSlashCommand("/memory add some important data", commands);
// result = {
//   commandToExecute: <add 子命令对象>,
//   args: "some important data",
//   canonicalPath: ["memory", "add"]
// }

// 使用别名
const result2 = parseSlashCommand("/mem add data", commands);
// 如果 "mem" 是 "memory" 的 altName
// result2 = {
//   commandToExecute: <add 子命令对象>,
//   args: "data",
//   canonicalPath: ["memory", "add"]  // 仍使用主名称
// }

// 无效命令
const result3 = parseSlashCommand("/nonexistent arg", commands);
// result3 = {
//   commandToExecute: undefined,
//   args: "",
//   canonicalPath: []
// }
```

### `SlashCommand` 接口（外部依赖类型）

```typescript
export interface SlashCommand {
  name: string;              // 主名称
  altNames?: string[];       // 别名列表
  description: string;       // 命令描述
  hidden?: boolean;          // 是否在帮助中隐藏
  kind: CommandKind;         // 命令类型（内置、用户文件、扩展等）
  autoExecute?: boolean;     // 选中时是否自动执行
  isSafeConcurrent?: boolean; // 是否可在 agent 忙碌时执行
  action?: (context, args) => ...; // 命令执行函数
  completion?: (context, partialArg) => ...; // 参数自动补全
  subCommands?: SlashCommand[]; // 子命令列表（树结构）
  // ...其他扩展/MCP 相关元数据
}
```

### `CommandKind` 枚举

```typescript
export enum CommandKind {
  BUILT_IN = 'built-in',
  USER_FILE = 'user-file',
  WORKSPACE_FILE = 'workspace-file',
  EXTENSION_FILE = 'extension-file',
  MCP_PROMPT = 'mcp-prompt',
  AGENT = 'agent',
  SKILL = 'skill',
}
```

## 依赖关系

### 内部依赖

| 依赖模块 | 导入内容 | 用途 |
|----------|----------|------|
| `../ui/commands/types.js` | `SlashCommand`（type-only） | 斜杠命令的接口定义，描述命令的名称、别名、子命令树等结构 |

### 外部依赖

无外部第三方依赖。本模块是纯解析逻辑，不依赖任何运行时库。

## 关键实现细节

1. **两级查找策略**：先匹配主名称再匹配别名，确保主名称始终优先。这意味着如果某个命令的别名与另一个命令的主名称冲突，主名称会胜出。代码中的 TODO 注释指出，未来可以在 `CommandService.ts` 中预计算一个统一的查找映射表以提升性能。

2. **规范路径始终使用主名称**：即使用户输入的是别名，`canonicalPath` 中记录的也是匹配命令的 `name`（主名称）而非用户实际输入。这保证了日志、遥测等下游消费者看到的总是一致的命令标识。

3. **贪婪匹配到最深子命令**：遍历循环会尽可能深入子命令树。每匹配一级，`commandToExecute` 就更新为更深层的命令。如果到达没有子命令的叶节点或无法继续匹配，循环停止。

4. **部分匹配的容错**：如果输入 `/memory unknown_sub arg`，解析器会匹配到 `memory` 命令（假设 `unknown_sub` 不是有效子命令），然后 `args` 将为 `"unknown_sub arg"`。这允许父命令处理未知的子命令参数。

5. **函数式风格**：`parseSlashCommand` 定义为 `const` 箭头函数而非 `function` 声明，是纯函数，不修改任何外部状态或输入参数。

6. **`readonly` 约束**：`commands` 参数声明为 `readonly SlashCommand[]`，明确表达函数不会修改命令列表，增强了类型安全性。
