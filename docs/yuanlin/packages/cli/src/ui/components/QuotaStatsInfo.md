# QuotaStatsInfo.tsx

## 概述

`QuotaStatsInfo` 是一个 React (Ink) 组件，用于在终端 CLI 界面中展示 API 配额的详细统计信息面板。与 `QuotaDisplay` 组件不同，`QuotaStatsInfo` 是一个更完整的信息面板，除了显示使用百分比和状态颜色外，还额外展示配额上限数值、重置规则说明以及配额耗尽时的操作指引。

该组件的核心能力：
- 计算并以颜色编码显示配额使用百分比
- 显示具体的配额上限数值（如 `Usage limit: 10,000`）
- 说明配额跨会话共享且每日重置的规则
- 在配额耗尽时提示用户通过 `/auth` 升级或切换到 API Key
- 支持通过 `showDetails` 参数控制详细信息的显示与隐藏

## 架构图（Mermaid）

```mermaid
graph TD
    A[QuotaStatsInfo 组件] --> B{remaining 或 limit<br/>是否为 undefined/0?}
    B -- 是 --> C[返回 null，不渲染]
    B -- 否 --> D[计算已使用百分比]
    D --> E[获取状态颜色<br/>getUsedStatusColor]
    E --> F[渲染纵向容器 Box]
    F --> G[第一行: 使用状态摘要]
    G --> H{remaining === 0?}
    H -- 是 --> I[显示 "Limit reached"<br/>附带重置时间]
    H -- 否 --> J[显示 "X% used"<br/>附带重置时间]
    F --> K{showDetails 为 true?}
    K -- 是 --> L[显示详细信息面板]
    K -- 否 --> M[不显示详细信息]
    L --> N[显示配额上限数值<br/>"Usage limit: X"]
    L --> O[显示重置规则说明]
    L --> P{remaining === 0?}
    P -- 是 --> Q[显示操作指引<br/>"/auth 升级或切换 API Key"]
    P -- 否 --> R[不显示操作指引]

    style A fill:#4A90D9,color:#fff
    style C fill:#D0021B,color:#fff
    style G fill:#7ED321,color:#fff
    style L fill:#BD10E0,color:#fff
    style Q fill:#F5A623,color:#fff
```

## 核心组件

### 接口定义

```typescript
interface QuotaStatsInfoProps {
  remaining: number | undefined;  // 剩余配额数量
  limit: number | undefined;      // 总配额限额
  resetTime?: string;             // 配额重置时间（可选）
  showDetails?: boolean;          // 是否显示详细信息面板，默认 true
}
```

### Props 详解

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `remaining` | `number \| undefined` | - | 剩余可用配额数量。为 `undefined` 时组件不渲染 |
| `limit` | `number \| undefined` | - | 配额总限额。为 `undefined` 或 `0` 时组件不渲染 |
| `resetTime` | `string` (可选) | - | 配额重置时间字符串，格式化后展示给用户 |
| `showDetails` | `boolean` | `true` | 控制是否展示详细信息（配额上限、规则说明、操作指引） |

### QuotaStatsInfo 组件

**类型**: `React.FC<QuotaStatsInfoProps>`

**渲染逻辑**:

1. **数据有效性校验**: 当 `remaining` 或 `limit` 为 `undefined`，或 `limit` 为 `0` 时，返回 `null`。

2. **计算已使用百分比**:
   ```typescript
   const usedPercentage = 100 - (remaining / limit) * 100;
   ```

3. **颜色计算**: 调用 `getUsedStatusColor()` 获取基于使用率的状态颜色。

4. **渲染结构**（纵向排列的 Box 容器）:

   **第一行 - 使用状态摘要**（带状态颜色）:
   - 配额耗尽时: `"Limit reached, resets in X"`
   - 配额未耗尽时: `"X% used (Limit resets in X)"`

   **详细信息面板**（仅当 `showDetails` 为 `true` 时显示）:
   - **配额上限**: `"Usage limit: 10,000"`（使用 `toLocaleString()` 格式化数字，添加千位分隔符）
   - **规则说明**: `"Usage limits span all sessions and reset daily."`
   - **操作指引**（仅当配额耗尽时显示）: `"Please /auth to upgrade or switch to an API key to continue."`

## 依赖关系

### 内部依赖

| 模块路径 | 导入项 | 用途 |
|----------|--------|------|
| `../semantic-colors.js` | `theme` | 语义化颜色主题对象，提供 `theme.text.primary` 等颜色值 |
| `../utils/formatters.js` | `formatResetTime` | 将重置时间字符串格式化为人类可读的简洁描述 |
| `../utils/displayUtils.js` | `getUsedStatusColor` | 根据使用百分比和阈值返回对应的终端颜色值 |
| `../utils/displayUtils.js` | `QUOTA_USED_WARNING_THRESHOLD` | 配额使用率的警告阈值常量 |
| `../utils/displayUtils.js` | `QUOTA_USED_CRITICAL_THRESHOLD` | 配额使用率的严重阈值常量 |

### 外部依赖

| 依赖包 | 导入项 | 用途 |
|--------|--------|------|
| `react` | `React` (类型) | 仅导入类型定义，用于 `React.FC` 类型注解 |
| `ink` | `Box` | Ink 框架的布局容器组件，提供 Flexbox 布局能力 |
| `ink` | `Text` | Ink 框架的文本渲染组件，支持颜色属性 |

## 关键实现细节

1. **与 QuotaDisplay 的区别**: `QuotaDisplay` 是一个紧凑的单行配额指示器，适合嵌入到状态栏或其他紧凑空间中；而 `QuotaStatsInfo` 是一个多行的详细信息面板，适合在专门的配额查看页面或 `/stats` 命令输出中使用。两者共享相同的百分比计算逻辑和颜色体系。

2. **始终显示策略**: 与 `QuotaDisplay` 不同，`QuotaStatsInfo` 没有阈值守卫逻辑——只要数据有效就会显示。这符合其"详细信息面板"的定位：用户主动查看配额信息时，无论使用率多低都应该看到完整的统计数据。

3. **数字本地化格式化**: 使用 `limit.toLocaleString()` 格式化配额上限数值，这会根据用户的区域设置添加千位分隔符（如英文环境下 `10000` 显示为 `10,000`），提升可读性。

4. **主题颜色的使用**: 详细信息部分使用 `theme.text.primary` 颜色而非状态颜色，保持了视觉层次：状态摘要行用颜色传达紧急程度，而说明性文字使用中性的主题颜色，不会让用户误以为所有内容都是警告。

5. **操作指引的条件展示**: 只有在配额完全耗尽（`remaining === 0`）时才显示"请通过 `/auth` 升级或切换 API Key"的操作指引。这避免了在配额尚有余量时过早引导用户执行升级操作。

6. **跨会话共享说明**: `"Usage limits span all sessions and reset daily."` 这条信息明确告知用户配额是全局性的（跨所有 CLI 会话共享），而非单个会话独立计算，且每日重置。这对于理解配额消耗模式至关重要。

7. **容器边距设置**: `marginTop={0} marginBottom={0}` 显式设置为零边距，确保该面板在被嵌入到其他布局中时不会引入意外的间距，由父组件全权控制布局间距。
