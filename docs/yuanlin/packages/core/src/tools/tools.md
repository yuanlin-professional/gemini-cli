# tools.ts

## 概述

`tools.ts` 是 Gemini CLI 工具系统的**核心类型与基类定义文件**，定义了整个工具体系的类型层次结构、基础抽象类、接口、枚举和辅助函数。它是所有具体工具实现的基础框架，提供了从参数验证、确认流程、执行逻辑到结果返回的完整抽象。

该文件是整个工具子系统中最重要的文件之一，定义了以下关键抽象：
- **`ToolInvocation`** 接口 — 经过验证的、可执行的工具调用
- **`DeclarativeTool`** 抽象类 — 声明式工具基类（将验证与执行分离）
- **`BaseDeclarativeTool`** 抽象类 — 在 `DeclarativeTool` 基础上增加 JSON Schema 验证
- **`BaseToolInvocation`** 抽象类 — 工具调用的便利基类（含策略确认流程）
- **`ToolResult`** 接口 — 工具执行结果的标准结构
- **`Kind`** 枚举 — 工具类型分类

文件路径：`packages/core/src/tools/tools.ts`

## 架构图（Mermaid）

```mermaid
graph TD
    subgraph 接口层
        TI["ToolInvocation&lt;TParams, TResult&gt;<br/>工具调用接口"]
        TB["ToolBuilder&lt;TParams, TResult&gt;<br/>工具构建器接口"]
        TR["ToolResult<br/>工具执行结果接口"]
    end

    subgraph 抽象基类层
        BTI["BaseToolInvocation<br/>工具调用基类"] -->|实现| TI
        DT["DeclarativeTool<br/>声明式工具基类"] -->|实现| TB
        BDT["BaseDeclarativeTool<br/>声明式工具基类<br/>（含 Schema 验证）"] -->|继承| DT
    end

    subgraph 确认流程
        BTI -->|shouldConfirmExecute| 策略决策["策略引擎决策<br/>allow/deny/ask_user"]
        策略决策 -->|ask_user| 确认详情["ToolCallConfirmationDetails"]
        确认详情 --> 编辑确认["ToolEditConfirmationDetails"]
        确认详情 --> 执行确认["ToolExecuteConfirmationDetails"]
        确认详情 --> MCP确认["ToolMcpConfirmationDetails"]
        确认详情 --> 信息确认["ToolInfoConfirmationDetails"]
        确认详情 --> 用户询问确认["ToolAskUserConfirmationDetails"]
        确认详情 --> 沙盒扩展确认["ToolSandboxExpansionConfirmationDetails"]
        确认详情 --> 退出计划确认["ToolExitPlanModeConfirmationDetails"]
    end

    subgraph 工具分类
        K["Kind 枚举"] --> KR["Read 读取"]
        K --> KE["Edit 编辑"]
        K --> KD["Delete 删除"]
        K --> KM["Move 移动"]
        K --> KS["Search 搜索"]
        K --> KX["Execute 执行"]
        K --> KT["Think 思考"]
        K --> KA["Agent 代理"]
        K --> KF["Fetch 抓取"]
        K --> KC["Communicate 通信"]
        K --> KP["Plan 计划"]
        K --> KSM["SwitchMode 切换模式"]
        K --> KO["Other 其他"]
    end

    subgraph 结果类型
        TR --> FD["FileDiff 文件差异"]
        TR --> GR["GrepResult 搜索结果"]
        TR --> LR["ListDirectoryResult 目录列表"]
        TR --> RMR["ReadManyFilesResult 批量读取"]
        TR --> TL["TodoList 待办列表"]
    end

    subgraph 消息总线通信
        BTI -->|getMessageBusDecision| MB["MessageBus<br/>消息总线"]
        MB -->|TOOL_CONFIRMATION_REQUEST| PE["策略引擎"]
        PE -->|TOOL_CONFIRMATION_RESPONSE| MB
    end

    DT -->|build()| TI
    BDT -->|build() 含验证| TI
```

## 核心组件

### 1. 类型与枚举

#### `ForcedToolDecision`
```typescript
type ForcedToolDecision = 'allow' | 'deny' | 'ask_user';
```
强制工具执行行为的决策类型。

#### `Kind` 枚举
工具的类型分类，共 13 种：

| 值 | 含义 | 是否只读 | 是否有副作用 |
|---|------|---------|-------------|
| `Read` | 读取 | 是 | 否 |
| `Edit` | 编辑 | 否 | 是 |
| `Delete` | 删除 | 否 | 是 |
| `Move` | 移动 | 否 | 是 |
| `Search` | 搜索 | 是 | 否 |
| `Execute` | 执行 | 否 | 是 |
| `Think` | 思考 | 否 | 否 |
| `Agent` | 代理 | 否 | 否 |
| `Fetch` | 抓取 | 是 | 否 |
| `Communicate` | 通信 | 否 | 否 |
| `Plan` | 计划 | 否 | 否 |
| `SwitchMode` | 切换模式 | 否 | 否 |
| `Other` | 其他 | 否 | 否 |

#### `MUTATOR_KINDS` 与 `READ_ONLY_KINDS`
```typescript
const MUTATOR_KINDS: Kind[] = [Kind.Edit, Kind.Delete, Kind.Move, Kind.Execute];
const READ_ONLY_KINDS: Kind[] = [Kind.Read, Kind.Search, Kind.Fetch];
```
- `MUTATOR_KINDS`：有副作用的工具类型，执行时可能需要用户确认。
- `READ_ONLY_KINDS`：只读工具类型，可安全并行执行。

#### `ToolConfirmationOutcome` 枚举
用户确认的可能结果：

| 值 | 含义 |
|---|------|
| `ProceedOnce` | 仅本次允许 |
| `ProceedAlways` | 本会话始终允许 |
| `ProceedAlwaysAndSave` | 永久允许并保存策略 |
| `ProceedAlwaysServer` | 允许整个服务器的工具 |
| `ProceedAlwaysTool` | 始终允许该工具 |
| `ModifyWithEditor` | 使用编辑器修改 |
| `Cancel` | 取消 |

### 2. `ToolInvocation<TParams, TResult>` 接口

定义了经过验证的、可执行的工具调用的完整契约：

| 方法/属性 | 说明 |
|----------|------|
| `params: TParams` | 经过验证的参数 |
| `getDescription(): string` | 获取操作描述（Markdown） |
| `getDisplayTitle?(): string` | 获取 UI 显示标题 |
| `getExplanation?(): string` | 获取解释性文本 |
| `toolLocations(): ToolLocation[]` | 获取影响的文件路径 |
| `shouldConfirmExecute(signal, decision?)` | 判断是否需要用户确认 |
| `execute(signal, updateOutput?, options?)` | 执行工具 |
| `getPolicyUpdateOptions?(outcome)` | 获取策略更新选项 |

### 3. `ExecuteOptions` 接口

工具执行时的选项包：
- `shellExecutionConfig?: ShellExecutionConfig` — Shell 执行配置
- `setExecutionIdCallback?: (executionId: number) => void` — 设置执行 ID 的回调

### 4. `BaseToolInvocation<TParams, TResult>` 抽象类

`ToolInvocation` 接口的便利基类，实现了完整的**策略确认流程**。

#### 构造函数参数
- `params: TParams` — 工具参数
- `messageBus: MessageBus` — 消息总线
- `_toolName?: string` — 工具名
- `_toolDisplayName?: string` — 工具显示名
- `_serverName?: string` — 服务器名（MCP 工具）
- `_toolAnnotations?: Record<string, unknown>` — 工具注解
- `respectsAutoEdit: boolean` — 是否遵守自动编辑模式
- `getApprovalMode: () => ApprovalMode` — 获取审批模式的函数

#### `shouldConfirmExecute()` 核心逻辑

```
1. 如果工具遵守 autoEdit 且当前为 AUTO_EDIT 模式且非强制 ask_user → 跳过确认
2. 获取决策（forcedDecision 或通过消息总线获取）
3. 如果 allow → 返回 false（无需确认）
4. 如果 deny → 抛出异常
5. 如果 ask_user → 返回确认详情
```

#### `getMessageBusDecision()` 消息总线决策流程

```
1. 生成 correlationId (UUID)
2. 创建 ToolConfirmationRequest 并发布到消息总线
3. 订阅 TOOL_CONFIRMATION_RESPONSE
4. 等待响应（30秒超时，超时默认 ask_user）
5. 支持 AbortSignal 取消
6. 响应处理: requiresUserConfirmation → ask_user, confirmed → allow, 否则 → deny
```

#### `publishPolicyUpdate()` 策略更新

当用户选择 `ProceedAlways` 或 `ProceedAlwaysAndSave` 时，通过消息总线发布策略更新事件。

### 5. `ToolBuilder<TParams, TResult>` 接口

工具构建器接口，定义了工具的声明信息和构建方法：

| 属性/方法 | 说明 |
|----------|------|
| `name: string` | 内部名称（API 调用用） |
| `displayName: string` | 用户友好的显示名称 |
| `description: string` | 工具描述 |
| `kind: Kind` | 工具类型 |
| `getSchema(modelId?): FunctionDeclaration` | 获取函数声明 schema |
| `schema: FunctionDeclaration` | 默认 schema（已废弃） |
| `isOutputMarkdown: boolean` | 输出是否为 Markdown |
| `canUpdateOutput: boolean` | 是否支持流式输出 |
| `isReadOnly: boolean` | 是否只读 |
| `build(params): ToolInvocation` | 验证参数并创建调用实例 |

### 6. `DeclarativeTool<TParams, TResult>` 抽象类

实现 `ToolBuilder` 接口的声明式工具基类。

#### 核心特性

- **`clone(messageBus?)`**：原型链安全的克隆方法（不使用 `structuredClone`，因为需要保留原型链和不可序列化属性）。
- **`isReadOnly` getter**：根据 `kind` 是否在 `READ_ONLY_KINDS` 中判断。
- **`getSchema(modelId?)`**：生成 `FunctionDeclaration`，自动为 schema 添加 `wait_for_previous` 参数。
- **`addWaitForPreviousParameter(schema)`**：为所有工具的参数 schema 自动注入 `wait_for_previous` 布尔参数，允许模型控制并行/串行执行。
- **`validateToolParams(params)`**：参数验证钩子，子类可覆盖。
- **`buildAndExecute(params, signal, updateOutput?, options?)`**：构建 + 执行的便利方法。
- **`validateBuildAndExecute(params, abortSignal)`**：安全版本，不抛异常，错误时返回包含错误信息的 `ToolResult`。

### 7. `BaseDeclarativeTool<TParams, TResult>` 抽象类

在 `DeclarativeTool` 基础上增加了 **JSON Schema 自动验证**：

#### `build(params)` 实现
1. 调用 `validateToolParams()` 进行参数验证。
2. 验证通过后调用抽象方法 `createInvocation()` 创建调用实例。

#### `validateToolParams(params)` 实现
1. 使用 `SchemaValidator.validate()` 进行 JSON Schema 验证。
2. 然后调用 `validateToolParamValues()` 进行自定义值验证（子类可覆盖）。

#### `createInvocation()` 抽象方法
子类必须实现的工厂方法，创建具体的 `ToolInvocation` 实例。

### 8. `ToolResult` 接口

工具执行结果的标准结构：

| 字段 | 类型 | 说明 |
|------|------|------|
| `llmContent` | `PartListUnion` | 传给 LLM 的内容 |
| `returnDisplay` | `ToolResultDisplay` | 用户展示的内容 |
| `error?` | `{ message, type? }` | 错误信息（存在则视为失败） |
| `data?` | `Record<string, unknown>` | 结构化数据载荷 |
| `tailToolCallRequest?` | `{ name, args }` | 尾调用请求（替换当前结果） |

### 9. 结果类型

#### `ToolResultDisplay` 联合类型
```typescript
type ToolResultDisplay = string | FileDiff | AnsiOutput | TodoList | SubagentProgress;
```

#### `FileDiff` 接口
文件差异信息，包含 diff 文本、文件名、路径、原始内容、新内容、diff 统计等。

#### `DiffStat` 接口
差异统计，分别记录模型和用户的增删行数和字符数。

#### `GrepResult`, `ListDirectoryResult`, `ReadManyFilesResult`
特定工具的结构化结果类型，均继承 `StructuredToolResult`（含 `summary` 字段）。

### 10. 确认详情类型

`ToolCallConfirmationDetails` 是一个联合类型，包含 7 种不同的确认场景：

| 类型 | `type` 值 | 用途 |
|------|-----------|------|
| `ToolSandboxExpansionConfirmationDetails` | `'sandbox_expansion'` | 沙盒权限扩展确认 |
| `ToolEditConfirmationDetails` | `'edit'` | 文件编辑确认（含 diff 预览） |
| `ToolExecuteConfirmationDetails` | `'exec'` | Shell 命令执行确认 |
| `ToolMcpConfirmationDetails` | `'mcp'` | MCP 工具调用确认 |
| `ToolInfoConfirmationDetails` | `'info'` | 一般信息确认 |
| `ToolAskUserConfirmationDetails` | `'ask_user'` | 用户提问确认 |
| `ToolExitPlanModeConfirmationDetails` | `'exit_plan_mode'` | 退出计划模式确认 |

### 11. `BackgroundExecutionData` 接口与类型守卫

后台执行元数据，用于 CLI UI 展示后台任务信息：
- `pid?: number` — 进程 ID
- `command?: string` — 命令
- `initialOutput?: string` — 初始输出

`isBackgroundExecutionData()` 函数用于运行时类型检查。

### 12. `hasCycleInSchema(schema)` 函数

检测 JSON Schema 中是否存在 `$ref` 引用循环。使用 DFS + 路径追踪算法：
- `visitedRefs` — 全局已访问 ref 集合（避免重复遍历）
- `pathRefs` — 当前路径上的 ref 集合（检测回环）
- 特殊处理：`#` 或 `#/` 引用始终视为循环

### 13. `TodoList` 与 `Todo`

```typescript
interface TodoList { todos: Todo[]; }
interface Todo { description: string; status: TodoStatus; }
type TodoStatus = 'pending' | 'in_progress' | 'completed' | 'cancelled' | 'blocked';
```

### 14. 类型守卫函数

| 函数 | 检测类型 |
|------|---------|
| `isTool(obj)` | `AnyDeclarativeTool` |
| `isStructuredToolResult(obj)` | `StructuredToolResult` |
| `hasSummary(res)` | `{ summary: string }` |
| `isGrepResult(res)` | `GrepResult` |
| `isListResult(res)` | `ListDirectoryResult \| ReadManyFilesResult` |
| `isFileDiff(res)` | `FileDiff` |
| `isBackgroundExecutionData(data)` | `BackgroundExecutionData` |

## 依赖关系

### 内部依赖

| 模块 | 导入内容 | 说明 |
|------|---------|------|
| `./tool-error.js` | `ToolErrorType` | 工具错误类型枚举 |
| `./grep-utils.js` | `GrepMatch` (类型) | Grep 匹配结果类型 |
| `../ide/ide-client.js` | `DiffUpdateResult` (类型) | IDE diff 更新结果类型 |
| `../services/shellExecutionService.js` | `ShellExecutionConfig` (类型) | Shell 执行配置类型 |
| `../utils/schemaValidator.js` | `SchemaValidator` | JSON Schema 验证器 |
| `../utils/terminalSerializer.js` | `AnsiOutput` (类型) | ANSI 终端输出类型 |
| `../confirmation-bus/message-bus.js` | `MessageBus` (类型) | 消息总线类型 |
| `../confirmation-bus/types.js` | `MessageBusType`, `ToolConfirmationRequest`, `ToolConfirmationResponse`, `Question` | 消息总线相关类型 |
| `../utils/markdownUtils.js` | `isRecord` | 对象类型守卫 |
| `../policy/types.js` | `ApprovalMode` | 审批模式枚举 |
| `../agents/types.js` | `SubagentProgress` (类型) | 子代理进度类型 |
| `../services/sandboxManager.js` | `SandboxPermissions` (类型，仅在类型声明中引用) | 沙盒权限类型 |

### 外部依赖

| 包名 | 导入内容 | 说明 |
|------|---------|------|
| `@google/genai` | `FunctionDeclaration`, `PartListUnion` | Google GenAI SDK 类型 |
| `node:crypto` | `randomUUID` | 生成 UUID（用于消息总线关联 ID） |

## 关键实现细节

1. **验证-构建-执行三段式**：工具系统采用"先验证参数，再创建调用实例，最后执行"的三段式设计。`DeclarativeTool.build()` 是入口，`BaseDeclarativeTool` 在其中注入了 JSON Schema 验证，子类通过 `createInvocation()` 提供具体实现。

2. **`wait_for_previous` 参数自动注入**：`DeclarativeTool.getSchema()` 会自动为每个工具的参数 schema 添加 `wait_for_previous` 布尔参数。这允许 LLM 显式控制工具调用的并行/串行执行顺序，是实现复杂工具编排的关键机制。

3. **消息总线确认流程**：`BaseToolInvocation.getMessageBusDecision()` 通过发布-订阅模式实现了完全异步的策略确认。使用 `correlationId` 关联请求和响应，30 秒超时默认为 `ask_user`，支持 `AbortSignal` 取消。

4. **策略更新的分离**：确认回调中的 `onConfirm` 目前是空实现（注释："策略更新现由调度器集中处理"），实际的策略更新通过 `publishPolicyUpdate()` 方法经消息总线发布。

5. **`tailToolCallRequest` 尾调用**：`ToolResult` 支持 `tailToolCallRequest` 字段，允许一个工具执行完成后请求立即执行另一个工具，其结果将替换原工具的响应。这实现了工具链式调用。

6. **`clone()` 的原型链保护**：`DeclarativeTool.clone()` 使用 `Object.create(Object.getPrototypeOf(this))` + `Object.assign()` 而非 `structuredClone()`，以保留原型链和函数属性。新的 `messageBus` 通过 `Object.defineProperty()` 设置为只读。

7. **JSON Schema 循环检测**：`hasCycleInSchema()` 函数使用经典的 DFS 回溯算法检测 `$ref` 引用循环。区分"全局已访问"和"当前路径已访问"两个集合，前者避免重复遍历，后者检测真正的循环。

8. **`silentBuild()` 错误静默**：私有方法 `silentBuild()` 将 `build()` 的异常转为 `Error` 对象返回，`validateBuildAndExecute()` 利用它实现了不抛异常的安全执行路径。

9. **`respectsAutoEdit` 快速通道**：当工具声明自己遵守自动编辑模式（`respectsAutoEdit = true`）且当前审批模式为 `AUTO_EDIT` 时，`shouldConfirmExecute()` 直接跳过确认流程，提供快速执行路径。
