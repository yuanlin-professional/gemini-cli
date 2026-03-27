# tool-registry.ts

## 概述

`tool-registry.ts` 是 Gemini CLI 工具系统的**核心注册表**，负责管理所有工具的注册、发现、排序、过滤和查询。它实现了两个主要类：

1. **`DiscoveredTool` / `DiscoveredToolInvocation`** — 通过外部命令动态发现的工具的定义和执行逻辑。
2. **`ToolRegistry`** — 工具注册表，是整个工具体系的中央管理器，维护所有已知工具的映射，提供按名称查找、按服务器分组、排除列表过滤、计划模式适配等功能。

文件路径：`packages/core/src/tools/tool-registry.ts`

## 架构图（Mermaid）

```mermaid
graph TD
    subgraph 工具注册表核心
        TR[ToolRegistry<br/>工具注册表]
        TR -->|管理| ATM["allKnownTools<br/>Map&lt;string, AnyDeclarativeTool&gt;"]
    end

    subgraph 工具类型
        BDT[BaseDeclarativeTool<br/>声明式工具基类] -->|继承| DT[DiscoveredTool<br/>已发现工具]
        BTI[BaseToolInvocation<br/>工具调用基类] -->|继承| DTI[DiscoveredToolInvocation<br/>已发现工具调用]
        MCPTool[DiscoveredMCPTool<br/>MCP 工具]
    end

    subgraph 工具生命周期
        注册["registerTool()<br/>注册工具"] --> ATM
        发现["discoverAllTools()<br/>发现工具"] -->|执行外部命令| 解析["解析 JSON 输出"]
        解析 -->|创建 DiscoveredTool| 注册
        排序["sortTools()<br/>排序工具"] -->|内建 → 已发现 → MCP| ATM
        查询["getTool()<br/>查询工具"] -->|遗留别名回退| ATM
        过滤["getActiveTools()<br/>获取活跃工具"] -->|排除黑名单| ATM
    end

    subgraph 外部依赖
        Config[Config<br/>配置对象] -->|提供排除列表/发现命令| TR
        MB[MessageBus<br/>消息总线] -->|事件通信| TR
        Policy[策略引擎] -->|通过 Config 间接影响| 过滤
    end

    subgraph 输出
        TR -->|getFunctionDeclarations()| FD["FunctionDeclaration[]<br/>LLM 函数声明"]
        TR -->|getAllToolNames()| 名称列表["string[]<br/>工具名列表"]
        TR -->|getAllTools()| 工具列表["AnyDeclarativeTool[]<br/>工具实例列表"]
    end
```

## 核心组件

### 1. `DiscoveredToolInvocation` 类

继承自 `BaseToolInvocation<ToolParams, ToolResult>`，负责执行通过外部命令发现的工具。

#### 核心属性
- `config: Config` — 配置对象
- `originalToolName: string` — 工具的原始名称（不含前缀）

#### 核心方法

##### `getDescription(): string`
返回参数的 JSON 字符串表示，用于日志和展示。

##### `execute(signal, updateOutput?): Promise<ToolResult>`
执行已发现工具的核心逻辑：

1. 从配置中获取工具调用命令（`getToolCallCommand()`）。
2. 如果存在沙盒管理器（`sandboxManager`），则通过沙盒准备命令。
3. 使用 `child_process.spawn` 启动子进程。
4. 将工具参数以 JSON 格式写入子进程的 stdin。
5. 收集 stdout 和 stderr 输出。
6. 成功时返回 stdout 内容作为 `llmContent`。
7. 出错时（非零退出码、信号终止、stderr 有内容）返回包含详细错误信息的 `ToolResult`，错误类型为 `ToolErrorType.DISCOVERED_TOOL_EXECUTION_ERROR`。

### 2. `DiscoveredTool` 类

继承自 `BaseDeclarativeTool<ToolParams, ToolResult>`，是已发现工具的声明式定义。

#### 构造函数
- 接收 `config`, `originalName`, `prefixedName`, `description`, `parameterSchema`, `messageBus`。
- 自动在描述末尾追加发现命令和调用命令的说明文本。
- 将工具类型设为 `Kind.Other`。

#### `createInvocation(params, messageBus, toolName?, displayName?)`
工厂方法，创建 `DiscoveredToolInvocation` 实例。

### 3. `ToolRegistry` 类

工具系统的中央管理器。

#### 核心属性
- `allKnownTools: Map<string, AnyDeclarativeTool>` — 所有已知工具（含活跃和非活跃）
- `config: Config` — 配置对象
- `messageBus: MessageBus` — 消息总线
- `isMainRegistry: boolean` — 是否为主注册表（影响工具过滤逻辑）

#### 核心方法

##### `clone(): ToolRegistry`
创建注册表的浅拷贝，复制所有已知工具的 Map 引用。

##### `registerTool(tool: AnyDeclarativeTool): void`
注册一个工具。如果同名工具已存在，记录警告并覆盖。**排除的工具也会被注册**，以便后续会话中启用。

##### `unregisterTool(name: string): void`
按名称注销一个工具。

##### `sortTools(): void`
对工具进行稳定排序，优先级为：
1. 内建工具（优先级 0）
2. 已发现工具 `DiscoveredTool`（优先级 1）
3. MCP 工具 `DiscoveredMCPTool`（优先级 2），同优先级按服务器名排序

##### `removeMcpToolsByServer(serverName: string): void`
移除指定 MCP 服务器的所有工具。

##### `discoverAllTools(): Promise<void>`
执行工具发现流程：
1. 移除所有之前发现的工具。
2. 调用 `discoverAndRegisterToolsFromCommand()` 从外部命令发现新工具。

##### `discoverAndRegisterToolsFromCommand(): Promise<void>` (private)
外部命令工具发现的核心实现：
1. 获取发现命令（`config.getToolDiscoveryCommand()`）。
2. 使用 `shell-quote` 解析命令。
3. 可选通过沙盒管理器准备命令。
4. 启动子进程执行发现命令。
5. 收集输出（stdout/stderr 各有 **10MB** 大小限制）。
6. 解析 stdout 为 JSON 数组，支持三种格式：
   - 包含 `function_declarations` 字段的对象
   - 包含 `functionDeclarations` 字段的对象（驼峰命名）
   - 直接的 `FunctionDeclaration` 对象（含 `name` 字段）
7. 为每个发现的函数创建 `DiscoveredTool` 并注册。

##### `getActiveTools(): AnyDeclarativeTool[]` (private)
返回所有未被排除的工具。过滤逻辑：
1. 构建工具元数据（包括 MCP 工具的服务器名）。
2. 从配置获取排除列表并扩展遗留别名。
3. 对每个工具检查其名称和类名是否在排除列表中。

##### `expandExcludeToolsWithAliases(excludeTools)` (private)
将排除列表中的工具名扩展为包含所有遗留别名的完整集合。

##### `isActiveTool(tool, excludeTools?)` (private)
判断工具是否活跃（未被排除）。检查项包括：
- 工具名（`tool.name`）
- 标准化的类名（去除前导下划线）
- MCP 工具的限定名和非限定名

##### `getFunctionDeclarations(modelId?): FunctionDeclaration[]`
获取所有活跃工具的函数声明，供 LLM 使用。特殊处理：
- 对 MCP 工具使用完全限定名。
- 在**计划模式**下，`WriteFile` 和 `Edit` 工具的描述被修改为仅允许写入/更新计划目录中的 `.md` 文件。
- 如果是主注册表且配置了 `mainAgentTools`，只返回指定工具。
- 结果去重。

##### `getFunctionDeclarationsFiltered(toolNames, modelId?): FunctionDeclaration[]`
按名称列表获取指定工具的函数声明。

##### `getAllToolNames(): string[]`
返回所有活跃工具名（MCP 工具使用完全限定名），去重。

##### `getAllTools(): AnyDeclarativeTool[]`
返回所有活跃工具实例，按显示名排序，去重。

##### `getToolsByServer(serverName: string): AnyDeclarativeTool[]`
返回指定 MCP 服务器的所有工具，按名称排序。

##### `getTool(name: string): AnyDeclarativeTool | undefined`
按名称获取工具。查找逻辑：
1. 先从 `allKnownTools` 直接查找。
2. 未找到时检查遗留别名映射。
3. 找到后检查是否为活跃工具。

## 依赖关系

### 内部依赖

| 模块 | 导入内容 | 说明 |
|------|---------|------|
| `./tools.js` | `Kind`, `BaseDeclarativeTool`, `BaseToolInvocation`, `AnyDeclarativeTool`, `ToolResult`, `ToolInvocation` | 工具基类和类型定义 |
| `./tool-names.js` | `DISCOVERED_TOOL_PREFIX`, `TOOL_LEGACY_ALIASES`, `getToolAliases`, `WRITE_FILE_TOOL_NAME`, `EDIT_TOOL_NAME` | 工具名称常量和别名工具函数 |
| `./mcp-tool.js` | `DiscoveredMCPTool` | MCP 协议工具类 |
| `./tool-error.js` | `ToolErrorType` | 工具错误类型枚举 |
| `../config/config.js` | `Config` | 配置接口 |
| `../policy/types.js` | `ApprovalMode` | 审批模式枚举（用于计划模式判断） |
| `../utils/safeJsonStringify.js` | `safeJsonStringify` | 安全的 JSON 序列化 |
| `../confirmation-bus/message-bus.js` | `MessageBus` | 消息总线类型 |
| `../utils/debugLogger.js` | `debugLogger` | 调试日志工具 |
| `../utils/events.js` | `coreEvents` | 核心事件发射器 |

### 外部依赖

| 包名 | 导入内容 | 说明 |
|------|---------|------|
| `@google/genai` | `FunctionDeclaration` | Google GenAI SDK 的函数声明类型 |
| `node:child_process` | `spawn` | Node.js 子进程模块，用于执行发现命令和工具调用 |
| `node:string_decoder` | `StringDecoder` | Node.js 字符串解码器，用于将 Buffer 转为 UTF-8 字符串 |
| `shell-quote` | `parse` | Shell 命令解析库，用于安全拆分发现命令字符串 |

## 关键实现细节

1. **工具发现的安全机制**：
   - stdout 和 stderr 各有 **10MB** 的大小限制，超出后立即杀死子进程。
   - 支持通过 `sandboxManager` 在沙盒环境中执行外部命令。
   - 工具调用时参数通过 stdin 传入，避免命令行注入。

2. **多格式 JSON 解析**：工具发现命令的输出支持三种 JSON 格式（`function_declarations`、`functionDeclarations`、直接对象），提供了良好的兼容性。

3. **遗留别名的全面处理**：
   - `getTool()` 在查找失败时自动回退到遗留别名。
   - `getActiveTools()` 在构建排除列表时展开所有遗留别名。
   - 确保无论用户使用新旧名称都能正确工作。

4. **计划模式的写入限制**：在计划模式下，`WriteFile` 和 `Edit` 工具的描述被动态修改，限制其只能操作计划目录中的 `.md` 文件。这是通过修改 schema 描述实现的软限制（依赖 LLM 遵守）。

5. **主注册表与子注册表**：`isMainRegistry` 标志影响 `getFunctionDeclarations()` 的行为——主注册表会额外检查 `mainAgentTools` 配置来限制可用工具集。

6. **排序稳定性**：`sortTools()` 使用优先级 + 稳定排序，确保相同类型的工具保持注册顺序，MCP 工具按服务器名排序。

7. **`clone()` 的浅拷贝特性**：`clone()` 创建新的 `ToolRegistry` 实例和新的 `Map`，但 `Map` 中的工具对象本身是共享引用。修改克隆注册表中的工具列表不影响原始注册表，但修改工具对象本身会相互影响。

8. **MCP 工具的名称处理**：MCP 工具在注册表中以短名存储，但在对外暴露时使用完全限定名（包含服务器前缀）。`isActiveTool()` 会同时检查限定名和非限定名。
