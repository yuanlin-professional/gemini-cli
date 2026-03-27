# deepMerge.ts

## 概述

`deepMerge.ts` 是 Gemini CLI 的**可配置策略深度合并模块**。它实现了一个灵活的对象深度合并算法，核心特点是支持**按路径自定义合并策略**：调用方可以为配置树中不同的路径指定不同的合并行为（替换、数组拼接、数组去重合并、浅合并等）。

该模块主要用于合并多层级的设置配置（如系统默认设置、用户设置、工作区设置），在 `settings.ts` 中被调用来实现多来源配置的优先级合并。

**文件路径**: `packages/cli/src/utils/deepMerge.ts`
**许可证**: Apache-2.0 (Copyright 2025 Google LLC)

## 架构图（Mermaid）

```mermaid
graph TD
    A[调用方] -->|传入策略函数 + 多个源对象| B[customDeepMerge]
    B -->|创建空结果对象| C[结果对象 result]
    B -->|遍历每个源对象| D[mergeRecursively]

    D --> E{遍历 source 的每个键}
    E -->|跳过危险键| F["过滤 __proto__ / constructor / prototype"]
    E -->|跳过 undefined 值| G[继续下一个键]
    E -->|正常键| H[获取合并策略]

    H --> I{策略判断}

    I -->|SHALLOW_MERGE| J[浅合并: {...obj1, ...obj2}]
    I -->|CONCAT + 数组| K[数组拼接: arr1.concat arr2]
    I -->|UNION + 数组| L["数组去重合并: [...new Set(...)]"]
    I -->|默认 REPLACE| M{值类型判断}

    M -->|双方都是对象| N[递归合并]
    M -->|仅 source 是对象| O[创建空对象后递归]
    M -->|其他| P[直接覆盖赋值]

    Q[getMergeStrategyForPath] -->|根据路径返回策略| H

    style B fill:#4A90D9,color:#fff
    style D fill:#F5A623,color:#fff
    style Q fill:#7ED321,color:#fff
```

## 核心组件

### `Mergeable` 类型

```typescript
export type Mergeable =
  | string
  | number
  | boolean
  | null
  | undefined
  | object
  | Mergeable[];
```

递归类型定义，表示所有可被合并的值类型。涵盖了 JSON 中可能出现的所有数据类型。

---

### `MergeableObject` 类型

```typescript
export type MergeableObject = Record<string, Mergeable>;
```

可合并对象的类型定义：键为字符串，值为 `Mergeable` 类型。

---

### `isPlainObject` 函数（私有）

```typescript
function isPlainObject(item: unknown): item is MergeableObject
```

类型守卫函数，判断一个值是否为"普通对象"（非 `null`、非数组的 `object`）。用于在合并时区分对象和其他类型。

---

### `mergeRecursively` 函数（私有）

```typescript
function mergeRecursively(
  target: MergeableObject,
  source: MergeableObject,
  getMergeStrategyForPath: (path: string[]) => MergeStrategy | undefined,
  path: string[] = [],
): MergeableObject
```

递归合并的核心实现。将 `source` 的内容合并到 `target` 上（原地修改 `target`）。

#### 参数说明

| 参数 | 类型 | 描述 |
|------|------|------|
| `target` | `MergeableObject` | 合并目标对象，被原地修改 |
| `source` | `MergeableObject` | 合并源对象，提供要合并的数据 |
| `getMergeStrategyForPath` | `(path: string[]) => MergeStrategy \| undefined` | 策略查询函数，根据当前属性路径返回合并策略 |
| `path` | `string[]` | 当前递归的路径，默认为空数组（根层级） |

#### 合并策略决策流程

对于 `source` 中的每个键 `key`：

1. **安全检查**：跳过 `__proto__`、`constructor`、`prototype` 键（防止原型链污染攻击）
2. **undefined 跳过**：如果源值为 `undefined`，跳过（不覆盖目标中的已有值）
3. **查询策略**：调用 `getMergeStrategyForPath([...path, key])` 获取当前路径的合并策略
4. **按策略处理**：

| 策略 | 条件 | 行为 |
|------|------|------|
| `SHALLOW_MERGE` | target 和 source 值都存在 | 将两个值当作对象，用展开运算符浅合并 `{...obj1, ...obj2}` |
| `CONCAT` | target 值为数组 | 将 source 值（转为数组）拼接到 target 数组末尾 |
| `UNION` | target 值为数组 | 将两个数组合并后通过 `Set` 去重 |
| `REPLACE`（默认） | 双方都是普通对象 | 递归调用 `mergeRecursively` 深度合并 |
| `REPLACE`（默认） | 仅 source 是普通对象 | 在 target 中创建空对象，再递归合并 |
| `REPLACE`（默认） | 其他情况 | 直接用 source 值覆盖 target 值 |

---

### `customDeepMerge` 函数

```typescript
export function customDeepMerge(
  getMergeStrategyForPath: (path: string[]) => MergeStrategy | undefined,
  ...sources: MergeableObject[]
): MergeableObject
```

公开的深度合并入口函数。创建一个空结果对象，然后依次将每个源对象合并进去。

#### 参数说明

| 参数 | 类型 | 描述 |
|------|------|------|
| `getMergeStrategyForPath` | `(path: string[]) => MergeStrategy \| undefined` | 策略查询回调。接收属性路径数组（如 `['editor', 'theme', 'colors']`），返回该路径应使用的合并策略。返回 `undefined` 表示使用默认的替换/深度合并行为 |
| `...sources` | `MergeableObject[]` | 可变数量的源对象，按优先级从低到高排列。后面的源对象会覆盖前面的 |

#### 返回值

返回一个新的 `MergeableObject`，包含所有源对象合并后的结果。

#### 使用示例

```typescript
import { customDeepMerge } from './deepMerge.js';
import { MergeStrategy } from '../config/settingsSchema.js';

// 定义路径策略
const getStrategy = (path: string[]) => {
  const key = path.join('.');
  if (key === 'extensions.enabled') return MergeStrategy.UNION;
  if (key === 'history.commands') return MergeStrategy.CONCAT;
  if (key === 'agents.overrides') return MergeStrategy.SHALLOW_MERGE;
  return undefined; // 默认深度合并/替换
};

const systemDefaults = { theme: 'light', extensions: { enabled: ['a'] } };
const userSettings = { theme: 'dark', extensions: { enabled: ['b'] } };
const workspaceSettings = { editor: { fontSize: 14 } };

const merged = customDeepMerge(getStrategy, systemDefaults, userSettings, workspaceSettings);
// merged = {
//   theme: 'dark',                    // 后者覆盖（REPLACE）
//   extensions: { enabled: ['a', 'b'] }, // 去重合并（UNION）
//   editor: { fontSize: 14 }          // 深度合并
// }
```

### `MergeStrategy` 枚举（外部依赖类型）

```typescript
export enum MergeStrategy {
  REPLACE = 'replace',        // 用新值替换旧值（默认）
  CONCAT = 'concat',          // 拼接数组
  UNION = 'union',            // 合并数组并去重
  SHALLOW_MERGE = 'shallow_merge', // 浅合并对象
}
```

## 依赖关系

### 内部依赖

| 依赖模块 | 导入内容 | 用途 |
|----------|----------|------|
| `../config/settingsSchema.js` | `MergeStrategy` | 合并策略枚举，定义了四种合并行为 |

### 外部依赖

无外部第三方依赖。本模块是纯算法实现，不依赖任何运行时库。

## 关键实现细节

1. **原型链污染防护**：在遍历 `source` 的键时，显式跳过 `__proto__`、`constructor` 和 `prototype` 三个危险属性名。因为 `JSON.parse` 可以生成这些属性作为自有属性，如果不过滤可能导致原型链污染漏洞。这是一项重要的安全措施。

2. **undefined 语义**：`undefined` 值在合并中被忽略（不会覆盖 target 中的已有值）。这与 `null` 不同——`null` 会正常覆盖目标值。这一设计使得源对象中可以通过省略某个键来表示"不参与合并"。

3. **路径感知策略**：策略查询函数接收的是完整路径数组（如 `['agents', 'overrides', 'myAgent']`），使得调用方可以基于配置的 schema 定义为不同深度的不同键指定不同的合并行为。这比全局统一的合并策略灵活得多。

4. **SHALLOW_MERGE 的容错处理**：当策略为 `SHALLOW_MERGE` 时，如果 target 或 source 的值不是对象，会降级为空对象 `{}` 参与合并，避免因类型不匹配而抛出异常。

5. **数组的 CONCAT 与 UNION**：
   - `CONCAT`：简单拼接，保留重复元素。适用于命令历史等场景
   - `UNION`：通过 `new Set()` 去重。适用于启用的扩展列表等需要唯一性的场景
   - 当 source 值不是数组时，会被包装为单元素数组 `[srcValue]` 再处理

6. **新对象语义**：`customDeepMerge` 始终返回一个新创建的对象（`const result = {}`），不会修改任何传入的源对象。但注意内部的 `mergeRecursively` 会修改 `target`，所以这个保证仅限于顶层调用。

7. **合并顺序决定优先级**：`sources` 参数按低优先级到高优先级排列。后面的源对象会覆盖前面的。这与 CSS 的级联规则、Git 的配置层级（system < global < local）类似。在设置场景中通常是：SystemDefaults < User < Workspace。

8. **递归对象创建**：当 target 中某个键的值不是对象，但 source 中对应值是对象时，会在 target 中创建一个空对象 `{}`，然后递归合并。这确保了深层结构能被正确创建，而不是简单地用 source 对象替换。
