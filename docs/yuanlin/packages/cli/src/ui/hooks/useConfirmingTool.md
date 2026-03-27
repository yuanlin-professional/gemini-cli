# useConfirmingTool.ts

## 概述

`useConfirmingTool` 是一个 React 自定义 Hook，用于从待处理的历史项队列中**选取队首（Head）需要用户确认的工具调用**。它实现了一种"确认队列"模式 --- 当多个工具调用同时等待审批时，该 Hook 只返回队列中第一个等待确认的工具，供 UI 逐个展示确认对话框。

该 Hook 本身非常轻量，核心逻辑委托给 `getConfirmingToolState` 工具函数完成，自身仅负责从 UI 状态上下文中读取数据并进行 `useMemo` 记忆化。

**文件路径**: `packages/cli/src/ui/hooks/useConfirmingTool.ts`

## 架构图（Mermaid）

```mermaid
graph TD
    subgraph 数据来源
        A[UIStateContext<br/>全局 UI 状态上下文] -->|useUIState| B[pendingHistoryItems<br/>待处理历史项列表]
    end

    subgraph useConfirmingTool Hook
        B --> C[useMemo 记忆化计算]
        C -->|委托| D[getConfirmingToolState<br/>确认工具状态计算函数]
        D --> E[ConfirmingToolState | null<br/>队首待确认工具]
    end

    subgraph getConfirmingToolState 内部逻辑
        D --> F[getAllToolCalls<br/>提取所有工具调用]
        F --> G[筛选 AwaitingApproval 状态]
        G --> H{是否有待确认工具?}
        H -->|否| I[返回 null]
        H -->|是| J[取第一个待确认工具<br/>计算其在全列表中的位置]
        J --> K[返回 ConfirmingToolState<br/>tool + index + total]
    end

    subgraph 消费方
        L[工具确认 UI 组件] -->|调用 Hook| C
        L -->|显示| E
    end
```

## 核心组件

### 1. `useConfirmingTool` 函数

Hook 的主体，签名：`() => ConfirmingToolState | null`

**执行流程**:
1. 通过 `useUIState()` 获取全局 UI 状态，解构出 `pendingHistoryItems`
2. 使用 `useMemo` 包裹 `getConfirmingToolState(pendingHistoryItems)` 的调用
3. 依赖项为 `[pendingHistoryItems]`，仅在待处理列表引用变化时重新计算
4. 返回 `ConfirmingToolState` 对象或 `null`

### 2. `ConfirmingToolState` 接口（来自 `confirmingTool.ts`）

```typescript
interface ConfirmingToolState {
  tool: IndividualToolCallDisplay;  // 队首待确认的工具调用显示对象
  index: number;                     // 该工具在全部待处理工具列表中的位置（1-based）
  total: number;                     // 全部待处理工具调用的总数
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `tool` | `IndividualToolCallDisplay` | 队首等待审批的工具调用对象，包含工具名称、参数、状态等完整信息 |
| `index` | `number` | 该工具在所有待处理工具中的位置编号（从 1 开始），用于显示 "第 X/Y 个" |
| `total` | `number` | 所有待处理工具调用的总数，用于显示 "第 X/Y 个" |

### 3. `getConfirmingToolState` 函数（核心计算逻辑）

位于 `../utils/confirmingTool.ts`，是纯函数，便于单独测试。

**执行逻辑**:
1. 调用 `getAllToolCalls(pendingHistoryItems)` 从所有待处理历史项中提取全部工具调用（扁平化）
2. 从提取结果中筛选状态为 `CoreToolCallStatus.AwaitingApproval` 的工具调用
3. 如果没有待确认工具，返回 `null`
4. 取筛选结果的第一个元素作为"队首"（`head`）
5. 使用 `callId` 在原始全列表中查找 `head` 的位置（`findIndex`）
6. 返回 `{ tool: head, index: headIndexInFullList + 1, total: allPendingTools.length }`

**注意**: `index` 字段使用 1-based 索引（加了 1），方便直接用于 UI 显示（如 "确认工具 2/5"）。

### 4. 类型重导出

```typescript
export type { ConfirmingToolState } from '../utils/confirmingTool.js';
```

Hook 文件同时重导出了 `ConfirmingToolState` 类型，使消费方无需直接依赖 `utils/confirmingTool` 模块。

## 依赖关系

### 内部依赖

| 依赖 | 来源路径 | 导入内容 |
|------|----------|----------|
| `UIStateContext` | `../contexts/UIStateContext.js` | `useUIState` Hook |
| `confirmingTool` | `../utils/confirmingTool.js` | `getConfirmingToolState` 函数、`ConfirmingToolState` 类型 |

#### `confirmingTool.ts` 的间接依赖

| 依赖 | 来源路径 | 导入内容 |
|------|----------|----------|
| `@google/gemini-cli-core` | 外部包 | `CoreToolCallStatus` 枚举 |
| `types` | `../types.js` | `HistoryItemWithoutId`、`IndividualToolCallDisplay` 类型 |
| `historyUtils` | `./historyUtils.js` | `getAllToolCalls` 函数 |

### 外部依赖

| 依赖 | 导入内容 |
|------|----------|
| `react` | `useMemo` |

## 关键实现细节

1. **确认队列模式（Queue Head Selection）**: 该 Hook 实现了一种队列化的工具确认流程。即使同时有多个工具等待审批，UI 也只显示第一个，用户确认或拒绝后，队列自动推进到下一个。这避免了同时弹出多个确认对话框造成的混乱。

2. **pendingHistoryItems 的数据来源**: 注释中明确说明使用 `pendingHistoryItems`（而非其他状态源）是为了同时捕获来自 **Gemini 模型响应** 和 **斜杠命令（Slash commands）** 两个来源的工具调用。这是一个重要的设计决策，确保确认机制的完整覆盖。

3. **逻辑分层**: Hook 层（`useConfirmingTool.ts`）和计算层（`confirmingTool.ts`）分离。Hook 负责 React 生命周期管理（状态读取 + 记忆化），纯计算逻辑放在独立的工具函数中，使其可以在非 React 环境下测试和复用。

4. **位置追踪（index / total）**: 返回值中包含 `index` 和 `total`，使 UI 能展示类似 "正在确认第 2 个工具（共 5 个）" 的进度信息，帮助用户了解还有多少确认请求待处理。

5. **轻量级 Hook**: 整个 Hook 只有 10 行代码，体现了单一职责原则 --- 它只做一件事：从全局状态中选取队首待确认工具。复杂的筛选和排序逻辑全部委托给工具函数。

6. **useMemo 优化**: 由于 `getConfirmingToolState` 涉及数组遍历、筛选和查找操作，使用 `useMemo` 避免在每次渲染时重复计算，仅当 `pendingHistoryItems` 引用变化时才重新执行。
