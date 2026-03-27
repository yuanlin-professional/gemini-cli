# agentUtils.ts

## 概述

`agentUtils.ts` 是 Gemini CLI 项目中用于渲染 **Agent 操作反馈消息**的工具模块。它提供了一个核心函数 `renderAgentActionFeedback`，负责将 Agent 的启用/禁用操作结果转化为面向用户的可读文本消息。该模块与具体的 UI 层解耦，通过回调函数让调用方自行决定如何格式化作用域标签（如加粗、变暗等），体现了良好的关注点分离设计。

**文件路径**: `packages/cli/src/utils/agentUtils.ts`
**许可证**: Apache-2.0 (Copyright 2026 Google LLC)

## 架构图（Mermaid）

```mermaid
graph TD
    A[调用方 / UI层] -->|传入 AgentActionResult + formatScope 回调| B[renderAgentActionFeedback]
    B -->|读取操作状态| C{status 判断}
    C -->|error| D[返回错误信息]
    C -->|no-op| E[返回"已处于目标状态"信息]
    C -->|success| F{计算受影响的作用域数量}
    F -->|2个作用域| G[返回双作用域消息]
    F -->|1个作用域| H[返回单作用域消息]

    I[AgentActionResult] -->|提供数据| B
    J[SettingScope 枚举] -->|用于作用域标签映射| B
    K[formatScope 回调] -->|格式化作用域显示| B

    style B fill:#4A90D9,color:#fff
    style C fill:#F5A623,color:#fff
    style I fill:#7ED321,color:#fff
```

## 核心组件

### `renderAgentActionFeedback` 函数

#### 函数签名

```typescript
export function renderAgentActionFeedback(
  result: AgentActionResult,
  formatScope: (label: string, path: string) => string,
): string
```

#### 参数说明

| 参数 | 类型 | 描述 |
|------|------|------|
| `result` | `AgentActionResult` | Agent 操作的结果对象，包含代理名称、操作类型、状态、错误信息以及受影响的作用域列表 |
| `formatScope` | `(label: string, path: string) => string` | 格式化回调函数，接收作用域标签（如 `"project"`、`"user"`）和对应路径，返回格式化后的字符串。调用方可在此实现加粗、着色等 UI 效果 |

#### 返回值

返回一个 `string` 类型的用户可读消息，描述 Agent 操作的结果。

#### 内部逻辑流程

1. **错误处理**（`status === 'error'`）：
   - 如果 `result.error` 存在，直接返回错误信息
   - 否则返回默认的错误描述，格式为：`An error occurred while attempting to ${action} agent "${agentName}".`

2. **空操作处理**（`status === 'no-op'`）：
   - 当 Agent 已经处于目标状态时，返回提示信息
   - 例如：`Agent "myAgent" is already enabled.`

3. **成功处理**（隐含 `status === 'success'`）：
   - 合并 `modifiedScopes`（实际修改的作用域）和 `alreadyInStateScopes`（已在目标状态的作用域）为 `totalAffectedScopes`
   - 根据受影响作用域数量生成消息：
     - **2个作用域**：生成包含两个作用域的消息，启用和禁用的措辞略有不同
     - **1个作用域**：生成单作用域消息

4. **作用域标签映射**（内部 `formatScopeItem` 函数）：
   - `SettingScope.Workspace` 映射为 `"project"`
   - 其他作用域转为小写（如 `"User"` -> `"user"`）

#### 使用示例

```typescript
// 在 TUI（终端 UI）中使用
const message = renderAgentActionFeedback(result, (label, path) => {
  return `**${label}** (${path})`;
});

// 在纯文本环境中使用
const message = renderAgentActionFeedback(result, (label, path) => {
  return `${label} [${path}]`;
});
```

### `AgentActionResult` 接口（外部依赖类型）

```typescript
// 来自 agentSettings.ts
export interface AgentActionResult extends Omit<FeatureActionResult, 'featureName'> {
  agentName: string;
}
```

该接口继承自 `FeatureActionResult`（移除了 `featureName` 字段），并添加了 `agentName` 属性。包含以下关键字段：

| 字段 | 类型 | 描述 |
|------|------|------|
| `agentName` | `string` | Agent 的名称标识 |
| `action` | `'enable' \| 'disable'` | 执行的操作类型 |
| `status` | `'success' \| 'no-op' \| 'error'` | 操作结果状态 |
| `error` | `string \| undefined` | 错误信息（仅在 status 为 error 时有值） |
| `modifiedScopes` | `Array<{ scope: SettingScope; path: string }>` | 实际被修改的作用域列表 |
| `alreadyInStateScopes` | `Array<{ scope: SettingScope; path: string }>` | 已经处于目标状态的作用域列表 |

### `SettingScope` 枚举（外部依赖类型）

```typescript
// 来自 config/settings.ts
export enum SettingScope {
  User = 'User',
  Workspace = 'Workspace',
  System = 'System',
  SystemDefaults = 'SystemDefaults',
  Session = 'Session',
}
```

## 依赖关系

### 内部依赖

| 依赖模块 | 导入内容 | 用途 |
|----------|----------|------|
| `../config/settings.js` | `SettingScope` | 枚举类型，用于判断作用域类型并映射显示标签 |
| `./agentSettings.js` | `AgentActionResult`（type-only） | 类型定义，描述 Agent 操作结果的数据结构 |

### 外部依赖

无外部第三方依赖。本模块是纯工具函数，仅依赖项目内部模块。

## 关键实现细节

1. **UI 解耦设计**：通过 `formatScope` 回调函数，将消息内容的生成与消息的展示格式完全分离。不同的 UI 层（TUI 终端、Web 界面、纯文本日志等）可以各自决定如何渲染作用域信息，而核心消息逻辑只需维护一份。

2. **作用域标签语义化**：`Workspace` 作用域在面向用户的消息中被映射为更易理解的 `"project"`，而非直接显示技术术语 `"workspace"`，提升了用户体验。

3. **防御性编程**：在 `status === 'error'` 分支中，当 `result.error` 为空时提供了兜底的默认错误消息，避免向用户返回空字符串。

4. **双作用域差异化措辞**：启用操作使用 `"enabled by setting it to enabled in X and Y settings"` 的主动语态，而禁用操作使用 `"is now disabled in both X and Y settings"` 的描述语态，使反馈更加自然。

5. **合并作用域列表**：将 `modifiedScopes` 和 `alreadyInStateScopes` 合并为 `totalAffectedScopes` 来决定消息格式，确保无论某个作用域是新修改的还是已在目标状态，都能完整地呈现给用户。

6. **纯函数特性**：该函数没有副作用，不修改任何外部状态，输入相同的参数始终返回相同的结果，便于单元测试和推理。
