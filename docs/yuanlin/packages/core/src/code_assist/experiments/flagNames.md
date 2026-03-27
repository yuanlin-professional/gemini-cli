# flagNames.ts

## 概述

`flagNames.ts` 是实验系统的 Flag 名称常量注册表，定义了 Gemini CLI 所有已知的实验 Flag 及其对应的数字 ID。每个 Flag 代表一个可以被服务端远程控制的功能开关或配置参数。该文件还导出了一个联合类型 `ExperimentFlagName`，用于在 TypeScript 中提供类型安全的 Flag ID 引用。

该文件是实验系统中最频繁被修改的文件之一，每当新增或移除实验功能时都需要在此处更新常量。

## 架构图（Mermaid）

```mermaid
flowchart TD
    A[flagNames.ts] -->|导出常量| B[ExperimentFlags 对象]
    A -->|导出类型| C[ExperimentFlagName 联合类型]

    B --> D[CONTEXT_COMPRESSION_THRESHOLD<br/>上下文压缩阈值<br/>ID: 45740197]
    B --> E[USER_CACHING<br/>用户缓存<br/>ID: 45740198]
    B --> F[BANNER_TEXT_NO_CAPACITY_ISSUES<br/>无容量问题横幅文本<br/>ID: 45740199]
    B --> G[BANNER_TEXT_CAPACITY_ISSUES<br/>有容量问题横幅文本<br/>ID: 45740200]
    B --> H[ENABLE_PREVIEW<br/>启用预览功能<br/>ID: 45740196]
    B --> I[ENABLE_NUMERICAL_ROUTING<br/>启用数值路由<br/>ID: 45750526]
    B --> J[CLASSIFIER_THRESHOLD<br/>分类器阈值<br/>ID: 45750527]
    B --> K[ENABLE_ADMIN_CONTROLS<br/>启用管理员控制<br/>ID: 45752213]
    B --> L[MASKING_PROTECTION_THRESHOLD<br/>遮蔽保护阈值<br/>ID: 45758817]
    B --> M[MASKING_PRUNABLE_THRESHOLD<br/>遮蔽可裁剪阈值<br/>ID: 45758818]
    B --> N[MASKING_PROTECT_LATEST_TURN<br/>遮蔽保护最新轮次<br/>ID: 45758819]
    B --> O[GEMINI_3_1_PRO_LAUNCHED<br/>Gemini 3.1 Pro 已上线<br/>ID: 45760185]
    B --> P[PRO_MODEL_NO_ACCESS<br/>Pro 模型无权限<br/>ID: 45768879]
    B --> Q[GEMINI_3_1_FLASH_LITE_LAUNCHED<br/>Gemini 3.1 Flash Lite 已上线<br/>ID: 45771641]

    C -->|类型等于| R[所有 Flag ID 值的联合类型<br/>45740196 | 45740197 | ... | 45771641]

    style A fill:#e1f5fe
    style B fill:#c8e6c9
    style C fill:#fff9c4
```

## 核心组件

### 1. `ExperimentFlags` 常量对象

```typescript
export const ExperimentFlags = {
  CONTEXT_COMPRESSION_THRESHOLD: 45740197,
  USER_CACHING: 45740198,
  // ...
} as const;
```

使用 `as const` 断言，确保所有属性值被推断为字面量类型而非宽泛的 `number` 类型。这使得 TypeScript 能够精确追踪每个 Flag 的具体 ID 值。

#### Flag 清单详解

| 常量名 | ID | 功能说明 |
|--------|-----|----------|
| `CONTEXT_COMPRESSION_THRESHOLD` | 45740197 | 上下文压缩阈值，控制何时对会话上下文进行压缩以节省 token |
| `USER_CACHING` | 45740198 | 用户级缓存功能开关，可能用于缓存用户的请求结果 |
| `BANNER_TEXT_NO_CAPACITY_ISSUES` | 45740199 | 无容量问题时显示的横幅文本（如公告、提示等） |
| `BANNER_TEXT_CAPACITY_ISSUES` | 45740200 | 有容量问题时显示的横幅文本（如服务繁忙提示） |
| `ENABLE_PREVIEW` | 45740196 | 启用预览功能，可能控制是否开放预览版特性 |
| `ENABLE_NUMERICAL_ROUTING` | 45750526 | 启用数值路由功能，可能用于请求分发/负载均衡策略 |
| `CLASSIFIER_THRESHOLD` | 45750527 | 分类器阈值参数，可能用于请求分类或意图识别的灵敏度调节 |
| `ENABLE_ADMIN_CONTROLS` | 45752213 | 启用管理员控制面板，为管理员用户提供额外功能 |
| `MASKING_PROTECTION_THRESHOLD` | 45758817 | 遮蔽保护阈值，控制上下文遮蔽（masking）时的保护级别 |
| `MASKING_PRUNABLE_THRESHOLD` | 45758818 | 遮蔽可裁剪阈值，控制哪些上下文内容可以被裁剪 |
| `MASKING_PROTECT_LATEST_TURN` | 45758819 | 遮蔽时是否保护最新一轮对话，防止最新内容被裁剪 |
| `GEMINI_3_1_PRO_LAUNCHED` | 45760185 | Gemini 3.1 Pro 模型是否已上线的开关 |
| `PRO_MODEL_NO_ACCESS` | 45768879 | Pro 模型无访问权限的标志，可能用于控制用户访问级别 |
| `GEMINI_3_1_FLASH_LITE_LAUNCHED` | 45771641 | Gemini 3.1 Flash Lite 模型是否已上线的开关 |

### 2. `ExperimentFlagName` 类型

```typescript
export type ExperimentFlagName =
  (typeof ExperimentFlags)[keyof typeof ExperimentFlags];
```

这是一个从 `ExperimentFlags` 对象的所有值中提取出的联合类型，等价于：

```typescript
type ExperimentFlagName = 45740196 | 45740197 | 45740198 | 45740199 | 45740200 | 45750526 | 45750527 | 45752213 | 45758817 | 45758818 | 45758819 | 45760185 | 45768879 | 45771641;
```

该类型用于在代码中引用 Flag ID 时提供类型安全，确保只能使用已注册的 Flag ID。

## 依赖关系

### 内部依赖

无。该文件是纯常量/类型定义文件，不依赖任何其他内部模块。

### 外部依赖

无。该文件不引用任何外部包。

## 关键实现细节

1. **`as const` 断言的作用**：`as const` 使得整个对象成为深度只读（deeply readonly），并且每个属性值被推断为字面量类型（如 `45740197` 而非 `number`）。这是构建类型安全的 Flag 系统的关键。

2. **数字 ID 作为值**：Flag 使用数字 ID 而非字符串名称作为标识符。这些 ID 与服务端实验系统中的 Flag ID 一一对应，确保客户端和服务端使用相同的标识符通信。

3. **Flag 分类**：从命名上可以将这些 Flag 分为几个功能域：
   - **上下文管理**：`CONTEXT_COMPRESSION_THRESHOLD`、`MASKING_*` 系列
   - **用户体验**：`BANNER_TEXT_*` 系列、`ENABLE_PREVIEW`
   - **模型控制**：`GEMINI_3_1_PRO_LAUNCHED`、`GEMINI_3_1_FLASH_LITE_LAUNCHED`、`PRO_MODEL_NO_ACCESS`
   - **路由与分类**：`ENABLE_NUMERICAL_ROUTING`、`CLASSIFIER_THRESHOLD`
   - **权限与缓存**：`USER_CACHING`、`ENABLE_ADMIN_CONTROLS`

4. **ID 值范围**：所有 Flag ID 都在 `4574xxxx` 到 `4577xxxx` 的范围内，表明它们是由同一个实验管理系统分配的递增 ID。

5. **类型导出模式**：`ExperimentFlagName` 类型使用了 TypeScript 的映射类型（Mapped Types）技巧——`(typeof Obj)[keyof typeof Obj]`——从常量对象中自动提取所有值的联合类型。这意味着新增 Flag 时只需在 `ExperimentFlags` 对象中添加一行，`ExperimentFlagName` 类型会自动更新，无需手动维护。
