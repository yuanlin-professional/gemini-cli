# statsCommand.ts

## 概述

`statsCommand.ts` 实现了 `/stats` 斜杠命令（别名 `/usage`），用于查看当前会话的统计信息。该命令是一个具有子命令的复合命令，提供三个维度的统计视图：会话统计（session）、模型统计（model）和工具统计（tools）。会话统计包含持续时间、认证信息、用户层级、当前模型、配额使用等详细数据；模型统计聚焦于当前模型和配额信息；工具统计展示工具调用相关的使用数据。该命令支持在 AI 代理繁忙时安全并发执行。

**文件路径**: `packages/cli/src/ui/commands/statsCommand.ts`

## 架构图（Mermaid）

```mermaid
flowchart TD
    A[用户执行 /stats 或 /usage] --> B{子命令解析}
    B -- 无子命令 --> C[默认: defaultSessionView]
    B -- session --> C
    B -- model --> D[模型统计视图]
    B -- tools --> E[工具统计视图]

    C --> C1[计算会话时长]
    C1 --> C2[获取用户身份信息]
    C2 --> C3[获取当前模型]
    C3 --> C4[构建 HistoryItemStats]
    C4 --> C5{配置对象存在?}
    C5 -- 是 --> C6[并行刷新配额和可用额度]
    C6 --> C7[附加配额信息到统计项]
    C7 --> C8[addItem 渲染统计信息]
    C5 -- 否 --> C8

    D --> D1[获取用户身份信息]
    D1 --> D2[获取当前模型和配额]
    D2 --> D3[构建 HistoryItemModelStats]
    D3 --> D4[addItem 渲染模型统计]

    E --> E1[构建 HistoryItemToolStats]
    E1 --> E2[addItem 渲染工具统计]

    subgraph getUserIdentity 函数
        F1[获取认证类型] --> F5[返回身份信息]
        F2[获取缓存的 Google 账户邮箱] --> F5
        F3[获取用户层级名称] --> F5
        F4[获取 G1 信用余额] --> F5
    end

    C2 --> getUserIdentity 函数
    D1 --> getUserIdentity 函数

    subgraph 子命令结构
        G1[session - 会话统计]
        G2[model - 模型统计]
        G3[tools - 工具统计]
    end
```

## 核心组件

### 1. 辅助函数

#### `getUserIdentity(context: CommandContext)`
- **可见性**: 模块内私有
- **参数**: `context` - 命令上下文
- **返回值**: `{ selectedAuthType, userEmail, tier, creditBalance }` 对象
- **功能**: 获取当前用户的身份和账户信息
- **实现细节**:
  1. 从合并后的设置中获取选定的认证类型（`settings.merged.security.auth.selectedType`），默认为空字符串
  2. 实例化 `UserAccountManager` 并调用 `getCachedGoogleAccount()` 获取缓存的 Google 账户邮箱
  3. 通过 `config.getUserTierName()` 获取用户层级名称（如免费、付费等）
  4. 通过 `config.getUserPaidTier()` 获取付费层级，再用 `getG1CreditBalance()` 计算 G1 信用余额

#### `defaultSessionView(context: CommandContext): Promise<void>`
- **可见性**: 模块内私有（但被主命令和 session 子命令共用）
- **功能**: 构建并渲染完整的会话统计视图
- **实现逻辑**:
  1. 计算会话持续时间：`当前时间 - sessionStartTime`
  2. 如果 `sessionStartTime` 不可用，显示错误并返回
  3. 调用 `getUserIdentity()` 获取用户身份信息
  4. 获取当前使用的模型（`config.getModel()`）
  5. 构建 `HistoryItemStats` 对象，包含格式化后的时长（`formatDuration`）、认证类型、邮箱、层级、模型、信用余额
  6. 如果配置对象存在，**并行**刷新用户配额和可用额度（`refreshUserQuota()` 和 `refreshAvailableCredits()`）
  7. 如果配额数据可用，附加配额信息（`quotas`）、池化剩余量（`pooledRemaining`）、池化上限（`pooledLimit`）、配额重置时间（`pooledResetTime`）
  8. 将构建好的统计项添加到 UI 历史记录

### 2. `statsCommand` 命令对象

- **类型**: `SlashCommand`（已导出）
- **属性**:

| 属性 | 值 | 说明 |
|------|-----|------|
| `name` | `'stats'` | 命令主名称 |
| `altNames` | `['usage']` | 别名，用户可通过 `/usage` 触发 |
| `description` | 查看会话统计的使用说明 | 命令描述 |
| `kind` | `CommandKind.BUILT_IN` | 内置命令 |
| `autoExecute` | `false` | 不自动执行，允许用户输入子命令 |
| `isSafeConcurrent` | `true` | 可在代理繁忙时安全执行 |
| `action` | `defaultSessionView` | 默认执行会话统计视图 |

### 3. 子命令

#### `session` 子命令
- **功能**: 显示会话级使用统计
- **action**: 调用 `defaultSessionView`（与主命令相同）
- **autoExecute**: `true`
- **isSafeConcurrent**: `true`
- **渲染类型**: `MessageType.STATS` -> `HistoryItemStats`

#### `model` 子命令
- **功能**: 显示模型级使用统计
- **action**: 同步函数，构建 `HistoryItemModelStats`
- **autoExecute**: `true`
- **isSafeConcurrent**: `true`
- **渲染数据**: 认证类型、邮箱、层级、当前模型、池化配额（剩余/上限/重置时间）
- **注意**: 不会像 `defaultSessionView` 那样主动刷新配额，而是使用已缓存的配额数据

#### `tools` 子命令
- **功能**: 显示工具级使用统计
- **action**: 同步函数，构建 `HistoryItemToolStats`
- **autoExecute**: `true`
- **isSafeConcurrent**: `true`
- **渲染数据**: 仅传递 `type: MessageType.TOOL_STATS`，具体的工具统计数据由渲染组件从其他上下文中获取

## 依赖关系

### 内部依赖

| 模块路径 | 导入内容 | 用途 |
|---------|---------|------|
| `../types.js` | `HistoryItemStats`, `HistoryItemModelStats`, `HistoryItemToolStats`, `MessageType` | UI 历史记录项类型定义和消息类型枚举 |
| `../utils/formatters.js` | `formatDuration` | 将毫秒时长格式化为人类可读的持续时间字符串 |
| `./types.js` | `CommandContext`, `SlashCommand`, `CommandKind` | 命令类型定义和接口 |
| `@google/gemini-cli-core` | `UserAccountManager`, `getG1CreditBalance` | 用户账户管理器（获取缓存的 Google 账户信息）和 G1 信用余额计算 |

### 外部依赖

无直接的外部第三方依赖。通过 `@google/gemini-cli-core` 间接使用核心功能。

## 关键实现细节

1. **并发安全标记**: 所有命令和子命令均设置 `isSafeConcurrent: true`，表明它们可以在 AI 代理正在流式生成响应时安全执行。这是因为统计命令仅读取数据和刷新配额，不会修改代理的运行状态。这使得用户可以在等待 AI 响应时随时查看使用情况。

2. **并行配额刷新**: `defaultSessionView` 使用 `Promise.all` 同时执行 `refreshUserQuota()` 和 `refreshAvailableCredits()` 两个异步操作。解构赋值 `const [quota]` 仅提取第一个结果（配额数据），因为可用额度的刷新结果不需要直接使用——它会更新内部缓存状态，后续通过 `getQuotaRemaining()` 等方法访问。

3. **三种统计视图的差异**:
   - **session**: 最完整的视图，包含时长、身份、模型、信用余额和实时刷新的配额数据
   - **model**: 轻量级视图，使用缓存的配额数据而不主动刷新，因此是同步操作
   - **tools**: 最简单的视图，仅发送消息类型标记，实际统计数据由 UI 渲染层自行从上下文中获取

4. **UserAccountManager 实例化策略**: 每次调用 `getUserIdentity` 都会创建新的 `UserAccountManager` 实例。这是因为该类主要用于读取缓存的账户数据（`getCachedGoogleAccount()`），是一个轻量级操作，无需缓存管理器实例。

5. **会话时间验证**: `defaultSessionView` 在计算前检查 `sessionStartTime` 是否可用。如果不可用（例如会话上下文异常），会显示明确的错误消息并提前返回，而不是产生无效的统计数据。

6. **配额系统数据结构**: 配额信息包含多个层面：
   - `quotas`: 从服务端刷新获取的原始配额数据
   - `pooledRemaining`: 池化配额剩余量（组织级共享配额）
   - `pooledLimit`: 池化配额总量上限
   - `pooledResetTime`: 配额重置时间点
   - `creditBalance`: G1 信用余额（通过付费层级计算）

7. **别名设计**: `/usage` 作为 `/stats` 的别名，语义上更直观地表达"查看使用量"的意图。两个命令名称适用于不同的用户心智模型——`stats` 偏向技术统计，`usage` 偏向资源消耗视角。
