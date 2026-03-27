# resolver.ts

## 概述

`resolver.ts` 是工具声明解析器模块，负责根据模型标识符（modelId）解析并返回最终的工具函数声明（`FunctionDeclaration`）。该模块实现了一种**覆盖（override）模式**：每个工具定义包含一个基础声明（`base`），以及一个可选的覆盖函数（`overrides`），解析器会根据传入的模型 ID 决定是否对基础声明进行属性合并覆盖。

该文件仅导出一个纯函数 `resolveToolDeclaration`，设计简洁，职责单一。

## 架构图（Mermaid）

```mermaid
flowchart TD
    A[调用者传入 ToolDefinition 和 modelId] --> B{modelId 存在且 definition.overrides 存在？}
    B -- 否 --> C[返回 definition.base 基础声明]
    B -- 是 --> D[调用 definition.overrides(modelId) 获取覆盖对象]
    D --> E{覆盖对象存在？}
    E -- 否 --> C
    E -- 是 --> F[将基础声明与覆盖对象浅合并]
    F --> G[返回合并后的 FunctionDeclaration]
```

## 核心组件

### `resolveToolDeclaration(definition, modelId?)`

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `definition` | `ToolDefinition` | 是 | 工具定义对象，包含 `base` 基础声明和可选的 `overrides` 覆盖函数 |
| `modelId` | `string` | 否 | 模型标识符，用于获取针对特定模型的声明覆盖 |

**返回值**: `FunctionDeclaration` -- 最终要发送到 Google GenAI API 的函数声明对象。

**解析逻辑**（按优先级）：

1. **无 modelId 或无 overrides 函数**：直接返回 `definition.base`，即基础声明。
2. **有 modelId 且有 overrides 函数**：调用 `definition.overrides(modelId)` 获取覆盖对象。
   - 若覆盖对象为 `null`/`undefined`（即该模型无需特殊处理），返回 `definition.base`。
   - 若覆盖对象存在，使用**展开运算符（spread）**进行浅合并：`{ ...definition.base, ...override }`，覆盖对象中的属性会替换基础声明中同名的属性。

## 依赖关系

### 内部依赖

| 模块路径 | 导入内容 | 用途 |
|----------|----------|------|
| `./types.js` | `ToolDefinition`（类型） | 工具定义的类型接口，定义了 `base` 和 `overrides` 的结构 |

### 外部依赖

| 包名 | 导入内容 | 用途 |
|------|----------|------|
| `@google/genai` | `FunctionDeclaration`（类型） | Google GenAI SDK 中定义的函数声明类型，作为返回值类型 |

## 关键实现细节

1. **浅合并策略**：使用 JavaScript 展开运算符 `{ ...base, ...override }` 进行对象合并。这意味着只有顶层属性会被覆盖，嵌套对象（如 `parameters` 中的 `properties`）会被整体替换而非深度合并。调用方的 `overrides` 函数需要注意这一点，如果需要修改嵌套属性，应提供完整的嵌套结构。

2. **纯函数设计**：`resolveToolDeclaration` 是一个无副作用的纯函数，不修改输入参数，不依赖外部状态，输出完全由输入决定，便于测试和推理。

3. **延迟求值**：`overrides` 是一个函数而非静态对象，这意味着覆盖逻辑可以在运行时根据 `modelId` 动态决定返回不同的覆盖内容，甚至返回 `null` 表示无需覆盖。这为不同模型之间的工具声明差异化提供了灵活性。

4. **防御性编程**：函数在多个层级进行了空值检查（`!modelId`、`!definition.overrides`、`!override`），确保在任何参数缺失的情况下都能安全地回退到基础声明。

5. **类型安全**：所有导入均为 TypeScript 类型导入（`type` 关键字），不会引入运行时开销。返回类型明确为 `FunctionDeclaration`，保证与 Google GenAI API 的契约一致。
