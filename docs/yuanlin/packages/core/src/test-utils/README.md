# packages/core/src/test-utils

## 概述

`test-utils` 目录提供了 core 包的**测试辅助工具集**，包含可复用的 mock 对象和工厂函数，供整个 core 包的单元测试使用。基于 **Vitest** 测试框架构建，提供了 Mock 工具（Tool）、Mock 消息总线（MessageBus）和 Mock 配置（Config）等核心测试基础设施。

## 目录结构

```
test-utils/
├── index.ts                  # 入口文件，导出 mock-tool
├── config.ts                 # Mock 配置工厂
├── mock-tool.ts              # Mock 工具类（MockTool、MockModifiableTool）
├── mock-message-bus.ts       # Mock 消息总线
└── mockWorkspaceContext.ts   # Mock 工作区上下文
```

## 核心组件

### 1. mock-tool.ts — Mock 工具类

提供两个高度可配置的 Mock 工具实现，用于测试工具调用链、策略引擎和调度器：

- **`MockTool`**：继承 `BaseDeclarativeTool`，支持自定义 `execute`（执行逻辑）和 `shouldConfirmExecute`（确认逻辑）回调。默认执行返回成功消息，默认无需确认。
- **`MockModifiableTool`**：实现 `ModifiableDeclarativeTool` 接口，支持模拟文件修改类工具的行为，提供 `getModifyContext` 方法返回文件路径、当前内容、提议内容和参数更新逻辑。
- **`MockToolInvocation` / `MockModifiableToolInvocation`**：对应的调用实例类，继承 `BaseToolInvocation`。
- **`MOCK_TOOL_SHOULD_CONFIRM_EXECUTE`**：预定义的确认回调，返回一个 `exec` 类型的确认详情。

### 2. mock-message-bus.ts — Mock 消息总线

模拟 `MessageBus`（确认总线）行为，用于测试工具确认流程和钩子执行：

- **`MockMessageBus`**：实现完整的发布/订阅模式，使用 Vitest 的 `vi.fn()` 创建可追踪的 mock 方法。
  - `publish`：捕获所有发布的消息到 `publishedMessages` 数组，并自动处理工具确认请求（根据 `defaultToolDecision` 返回 allow/deny/ask_user 响应）。
  - `subscribe` / `unsubscribe`：管理消息订阅。
  - `clear`：清除所有消息和订阅，实现测试隔离。
- **`createMockMessageBus()`**：工厂函数，创建 MockMessageBus 实例并转型为 MessageBus 接口。
- **`getMockMessageBusInstance()`**：辅助函数，从 MessageBus 接口还原 MockMessageBus 实例以访问测试专属属性。

### 3. config.ts — Mock 配置工厂

提供测试用的 `Config` 对象构建工具：

- **`DEFAULT_CONFIG_PARAMETERS`**：默认配置参数，包含 `usageStatisticsEnabled: true`、`debugMode: false`、`model: 'gemini-9001-super-duper'`、`targetDir: '/'` 等。
- **`makeFakeConfig(overrides?)`**：工厂函数，基于默认参数创建 `Config` 实例，支持通过 `Partial<ConfigParameters>` 覆盖任意字段。

### 4. mockWorkspaceContext.ts — Mock 工作区上下文

提供模拟的工作区上下文对象，用于测试依赖工作区环境信息的组件。

## 依赖关系

- **内部依赖**：
  - `mock-tool.ts` 依赖 `../tools/tools.js`（BaseDeclarativeTool、BaseToolInvocation 等基类）和 `../tools/modifiable-tool.js`（ModifiableDeclarativeTool 接口）。
  - `mock-message-bus.ts` 依赖 `../confirmation-bus/message-bus.js` 和 `../confirmation-bus/types.js`（MessageBus 类型和 MessageBusType 枚举）。
  - `config.ts` 依赖 `../config/config.js`（Config 类和 ConfigParameters 类型）。

- **外部依赖**：`vitest`（`vi.fn()` 用于创建可追踪的 mock 函数）。

- **导出方式**：`index.ts` 仅导出 `mock-tool.js`；`config.ts` 和 `mock-message-bus.ts` 需要单独导入。
