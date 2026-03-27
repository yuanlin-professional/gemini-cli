# QuotaDisplay.tsx

## 概述

`QuotaDisplay` 是一个 React (Ink) 组件，用于在终端 CLI 界面中显示 API 配额（Quota）的使用状态。它根据已使用百分比动态展示不同颜色和文案的配额警告信息，帮助用户及时了解 API 调用限额的消耗情况。

该组件的核心能力：
- 根据 `remaining`（剩余量）和 `limit`（总限额）计算已使用百分比
- 默认仅在使用率达到警告阈值时才显示（可通过 `forceShow` 强制显示）
- 根据使用率级别（正常/警告/严重）显示不同颜色
- 支持简洁模式（`terse`）和详细模式两种文案风格
- 在配额耗尽时显示"Limit reached"及重置时间
- 支持小写模式（`lowercase`）

## 架构图（Mermaid）

```mermaid
graph TD
    A[QuotaDisplay 组件] --> B{remaining 或 limit<br/>是否为 undefined/0?}
    B -- 是 --> C[返回 null，不渲染]
    B -- 否 --> D[计算已使用百分比<br/>usedPercentage]
    D --> E{非强制显示且<br/>使用率 < 警告阈值?}
    E -- 是 --> F[返回 null，不渲染]
    E -- 否 --> G[获取状态颜色<br/>getUsedStatusColor]
    G --> H{remaining === 0?}
    H -- 是 --> I[生成"Limit reached"文案<br/>可附带重置时间]
    H -- 否 --> J[生成百分比文案<br/>如"75% used"]
    I --> K{lowercase 模式?}
    J --> K
    K -- 是 --> L[转换为小写]
    K -- 否 --> M[保持原样]
    L --> N[渲染带颜色的 Text 组件]
    M --> N

    subgraph 颜色逻辑
        G --> O[正常: 绿色]
        G --> P[警告: 黄色]
        G --> Q[严重: 红色]
    end

    style A fill:#4A90D9,color:#fff
    style C fill:#D0021B,color:#fff
    style F fill:#D0021B,color:#fff
    style N fill:#7ED321,color:#fff
    style O fill:#50E3C2,color:#000
    style P fill:#F5A623,color:#fff
    style Q fill:#D0021B,color:#fff
```

## 核心组件

### 接口定义

```typescript
interface QuotaDisplayProps {
  remaining: number | undefined;  // 剩余配额数量
  limit: number | undefined;      // 总配额限额
  resetTime?: string;             // 配额重置时间（ISO 格式字符串）
  terse?: boolean;                // 简洁模式，默认 false
  forceShow?: boolean;            // 强制显示（忽略阈值判断），默认 false
  lowercase?: boolean;            // 小写模式，默认 false
}
```

### Props 详解

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `remaining` | `number \| undefined` | - | 剩余可用配额数量。为 `undefined` 时组件不渲染 |
| `limit` | `number \| undefined` | - | 配额总限额。为 `undefined` 或 `0` 时组件不渲染 |
| `resetTime` | `string` (可选) | - | 配额重置时间的字符串表示，传递给 `formatResetTime` 格式化 |
| `terse` | `boolean` | `false` | 简洁模式：仅显示百分比数值（如 `75%`），不附加说明文字 |
| `forceShow` | `boolean` | `false` | 强制显示：即使使用率未达警告阈值也显示 |
| `lowercase` | `boolean` | `false` | 小写模式：将最终文案转换为全小写 |

### QuotaDisplay 组件

**类型**: `React.FC<QuotaDisplayProps>`

**渲染逻辑**:

1. **数据有效性校验**: 当 `remaining` 或 `limit` 为 `undefined`，或 `limit` 为 `0` 时，返回 `null`。

2. **计算已使用百分比**:
   ```typescript
   const usedPercentage = 100 - (remaining / limit) * 100;
   ```

3. **阈值守卫**: 如果非强制显示模式，且使用率低于 `QUOTA_USED_WARNING_THRESHOLD`，返回 `null`。即默认情况下配额充足时不显示任何内容。

4. **颜色计算**: 调用 `getUsedStatusColor()` 根据使用百分比和阈值配置获取对应的显示颜色。

5. **文案生成**:
   - **配额耗尽** (`remaining === 0`):
     - 简洁模式: `"Limit reached"`
     - 详细模式: `"Limit reached, resets in X"`（附带重置时间）
   - **配额未耗尽**:
     - 简洁模式: `"75%"`
     - 详细模式: `"75% used (Limit resets in X)"`（附带重置时间）

6. **大小写处理**: 如果 `lowercase` 为 `true`，对最终文案执行 `.toLowerCase()`。

7. **渲染输出**: 使用 `<Text color={color}>` 渲染带颜色的文案。

## 依赖关系

### 内部依赖

| 模块路径 | 导入项 | 用途 |
|----------|--------|------|
| `../utils/displayUtils.js` | `getUsedStatusColor` | 根据使用百分比和阈值返回对应的终端颜色值 |
| `../utils/displayUtils.js` | `QUOTA_USED_WARNING_THRESHOLD` | 配额使用率的警告阈值常量 |
| `../utils/displayUtils.js` | `QUOTA_USED_CRITICAL_THRESHOLD` | 配额使用率的严重阈值常量 |
| `../utils/formatters.js` | `formatResetTime` | 将重置时间字符串格式化为人类可读的时间描述（如 "2 hours"） |

### 外部依赖

| 依赖包 | 导入项 | 用途 |
|--------|--------|------|
| `react` | `React` (类型) | 仅导入类型定义，用于 `React.FC` 类型注解 |
| `ink` | `Text` | Ink 框架的文本渲染组件，支持颜色属性 |

## 关键实现细节

1. **渐进式显示策略**: 组件默认不显示（配额充裕时），只在使用率达到警告阈值后才开始显示。这避免了在配额充足时向用户展示无意义的信息，同时在配额紧张时及时提醒。`forceShow` 参数提供了覆盖此行为的能力，适用于需要始终展示配额信息的场景（如设置页面或状态面板）。

2. **三级颜色体系**: 通过 `getUsedStatusColor` 函数结合 `QUOTA_USED_WARNING_THRESHOLD` 和 `QUOTA_USED_CRITICAL_THRESHOLD` 两个阈值常量，实现了三级颜色体系：
   - 使用率低于警告阈值：正常颜色（通常为绿色）
   - 使用率介于警告和严重阈值之间：警告颜色（通常为黄色）
   - 使用率超过严重阈值或已耗尽：严重颜色（通常为红色）

3. **百分比计算公式**: `usedPercentage = 100 - (remaining / limit) * 100`。这是从"剩余量"推算"已使用量"的计算方式，而非直接使用"已用量"。这与 API 通常返回 `remaining` 字段的设计一致。

4. **重置时间的条件展示**: 重置时间 (`resetTime`) 是可选的，仅在提供时才附加到文案中。且在简洁模式下，配额耗尽时仍然不展示重置时间，而在详细模式下会展示。这说明简洁模式被设计用于空间受限的场景（如状态栏）。

5. **小写模式的应用场景**: `lowercase` 参数将文案转为小写，可能用于嵌入到句子中间的场景，避免文案开头大写导致排版不协调。

6. **除零保护**: 通过 `limit === 0` 的检查，避免了除零错误（`remaining / limit` 中 `limit` 为 0 的情况）。
