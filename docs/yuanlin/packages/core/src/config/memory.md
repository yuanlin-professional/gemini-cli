# memory.ts

## 概述

`memory.ts` 定义了**分层记忆（Hierarchical Memory）**的数据结构和工具函数。该模块实现了一种三层记忆体系（全局 / 扩展 / 项目），并提供了将分层记忆扁平化为单一字符串的功能，用于在模型对话中展示或兼容旧版系统。该文件非常精简，仅包含一个接口和一个函数。

## 架构图（Mermaid）

```mermaid
graph TD
    subgraph 分层记忆结构
        A[HierarchicalMemory 接口]
        B[global<br/>全局记忆]
        C[extension<br/>扩展记忆]
        D[project<br/>项目记忆]
    end

    A --> B
    A --> C
    A --> D

    subgraph 扁平化处理
        E[flattenMemory 函数]
        F{输入类型判断}
        G[返回空字符串]
        H[直接返回字符串]
        I[拼接各层记忆]
    end

    E --> F
    F -->|undefined/falsy| G
    F -->|string 类型| H
    F -->|HierarchicalMemory 类型| I

    B -->|非空时拼接| I
    C -->|非空时拼接| I
    D -->|非空时拼接| I

    I --> J[格式化输出<br/>"--- 层名 ---<br/>内容"]

    style A fill:#4a9eff,color:#fff
    style E fill:#2ecc71,color:#fff
    style F fill:#ff9f43,color:#fff
```

## 核心组件

### 1. `HierarchicalMemory` 接口

定义了三层可选的记忆结构：

| 字段 | 类型 | 可选 | 描述 |
|------|------|------|------|
| `global` | `string` | 是 | 全局记忆，跨所有项目和扩展共享的记忆内容 |
| `extension` | `string` | 是 | 扩展记忆，与特定扩展关联的记忆内容 |
| `project` | `string` | 是 | 项目记忆，与当前项目关联的记忆内容 |

所有字段均为可选（`?`），允许部分层级为空。

### 2. `flattenMemory` 函数

将分层记忆或字符串类型的记忆扁平化为单一字符串输出。

```typescript
function flattenMemory(memory?: string | HierarchicalMemory): string
```

**参数**: `memory` - 可以是以下类型之一：
- `undefined` / `null` / `''`（falsy 值）
- `string`（旧版纯字符串记忆）
- `HierarchicalMemory`（新版分层记忆对象）

**返回值**: `string` - 扁平化后的记忆文本

**处理逻辑**:

1. **空值处理**: 如果 `memory` 为 falsy，返回空字符串 `''`
2. **字符串直通**: 如果 `memory` 是字符串类型，直接返回（兼容旧版）
3. **分层合并**: 如果是 `HierarchicalMemory` 对象：
   - 按 `global` -> `extension` -> `project` 的固定顺序遍历各层
   - 对每层内容进行 `trim()` 处理
   - 跳过空内容的层
   - 非空层格式化为 `--- 层名 ---\n内容` 的格式
   - 各层之间用 `\n\n`（双换行）分隔

**输出示例**:

```
--- Global ---
全局配置信息...

--- Extension ---
扩展相关指令...

--- Project ---
项目特定上下文...
```

## 依赖关系

### 内部依赖

无内部依赖。此文件是一个独立的纯工具模块。

### 外部依赖

无外部依赖。

## 关键实现细节

1. **向后兼容设计**: `flattenMemory` 的参数类型是 `string | HierarchicalMemory`，当接收到旧版的纯字符串记忆时直接透传，实现了新旧数据格式的平滑过渡。

2. **固定的层级优先顺序**: 扁平化输出始终按 `Global -> Extension -> Project` 的顺序排列。这一顺序隐含了作用域从大到小的层级关系：全局最宽泛，项目最具体。

3. **防御性 trim 处理**: 每个层级的内容都经过 `trim()` 处理且检查非空，确保不会因为空白字符产生无意义的输出段落。

4. **清晰的分隔标记**: 使用 `--- 层名 ---` 的分隔符格式，易于人类和模型识别不同记忆层的边界。这种格式在 Markdown 中也具有良好的可读性。

5. **纯函数设计**: `flattenMemory` 是一个无副作用的纯函数，不依赖任何外部状态，输入确定时输出完全确定，便于测试和复用。

6. **轻量级模块**: 整个文件不到 35 行代码，无任何外部依赖，体现了单一职责原则。它只关注"记忆数据的结构定义和格式化"这一件事。
