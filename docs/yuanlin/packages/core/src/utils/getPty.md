# getPty.ts

## 概述

`getPty.ts` 是 Gemini CLI 核心包中的伪终端（PTY）获取工具模块。该模块的核心职责是**在运行时动态加载伪终端实现库**，为 CLI 工具提供底层的终端模拟能力。伪终端（Pseudo-Terminal）是一种允许程序像与真实终端交互一样运行子进程的机制，常用于命令行工具中执行 shell 命令并捕获其输出。

该模块采用**降级策略（Fallback Strategy）**：优先尝试加载 `@lydell/node-pty`，若失败则回退到 `node-pty`，若两者都不可用则返回 `null`，表示系统将使用普通的 `child_process` 替代方案。此外，还支持通过环境变量 `GEMINI_PTY_INFO` 强制跳过 PTY，直接使用 `child_process`。

## 架构图（Mermaid）

```mermaid
flowchart TD
    A[调用 getPty] --> B{环境变量<br/>GEMINI_PTY_INFO === 'child_process'?}
    B -- 是 --> C[返回 null<br/>使用 child_process]
    B -- 否 --> D[尝试动态导入<br/>@lydell/node-pty]
    D -- 成功 --> E[返回 PtyImplementation<br/>name: 'lydell-node-pty']
    D -- 失败 --> F[尝试动态导入<br/>node-pty]
    F -- 成功 --> G[返回 PtyImplementation<br/>name: 'node-pty']
    F -- 失败 --> H[返回 null<br/>使用 child_process]

    style A fill:#4CAF50,color:#fff
    style C fill:#FF9800,color:#fff
    style E fill:#2196F3,color:#fff
    style G fill:#2196F3,color:#fff
    style H fill:#FF9800,color:#fff
```

## 核心组件

### 1. 类型定义：`PtyImplementation`

```typescript
export type PtyImplementation = {
  module: any;
  name: 'lydell-node-pty' | 'node-pty';
} | null;
```

这是一个**联合类型**，表示 PTY 实现的加载结果：

| 字段 | 类型 | 说明 |
|------|------|------|
| `module` | `any` | 动态导入的 PTY 模块实例，类型为 `any` 因为两个库的 API 签名在类型层面未统一 |
| `name` | `'lydell-node-pty' \| 'node-pty'` | 标识当前加载的具体 PTY 实现名称，方便上层代码做差异化处理 |
| `null` | - | 表示没有可用的 PTY 实现，调用方应回退到 `child_process` |

### 2. 接口定义：`PtyProcess`

```typescript
export interface PtyProcess {
  readonly pid: number;
  onData(callback: (data: string) => void): void;
  onExit(callback: (e: { exitCode: number; signal?: number }) => void): void;
  kill(signal?: string): void;
}
```

`PtyProcess` 是对 PTY 进程的**统一抽象接口**，无论底层使用哪种 PTY 实现，上层代码都通过此接口与 PTY 进程交互：

| 成员 | 类型 | 说明 |
|------|------|------|
| `pid` | `readonly number` | 进程 ID，只读属性 |
| `onData` | `(callback: (data: string) => void) => void` | 注册数据回调，当 PTY 进程有输出时触发 |
| `onExit` | `(callback: (e: { exitCode: number; signal?: number }) => void) => void` | 注册退出回调，当进程退出时触发，携带退出码和可选的信号编号 |
| `kill` | `(signal?: string) => void` | 终止进程，可选传入信号名称（如 `'SIGTERM'`、`'SIGKILL'`） |

### 3. 核心函数：`getPty`

```typescript
export const getPty = async (): Promise<PtyImplementation> => { ... }
```

这是模块导出的唯一函数，是一个**异步函数**，返回 `Promise<PtyImplementation>`。

**执行流程：**

1. **环境变量检查**：检查 `process.env['GEMINI_PTY_INFO']` 是否为 `'child_process'`，若是则直接返回 `null`，跳过 PTY 加载。
2. **优先加载 `@lydell/node-pty`**：通过 `await import(lydell)` 动态导入。`@lydell/node-pty` 是 `node-pty` 的一个社区维护 fork，通常包含更新的修复和改进。
3. **回退加载 `node-pty`**：若第一步失败（库未安装或加载异常），则尝试加载原始的 `node-pty`。
4. **全部失败返回 null**：若两个库都无法加载，返回 `null`。

## 依赖关系

### 内部依赖

该模块不依赖项目内部的其他模块，是一个**纯工具型叶子模块**。

### 外部依赖

| 依赖 | 类型 | 说明 |
|------|------|------|
| `@lydell/node-pty` | 可选运行时依赖 | 首选的 PTY 实现库，Simon Lydell 维护的 node-pty fork |
| `node-pty` | 可选运行时依赖 | 备选的 PTY 实现库，由 Microsoft 维护，用于 VS Code 终端等场景 |
| `process.env` | Node.js 内置 | 读取环境变量 `GEMINI_PTY_INFO` |

> 注意：两个 PTY 库都是**可选依赖**，模块通过 `try/catch` 容错，即使都未安装也不会抛出异常。

## 关键实现细节

### 1. 动态导入（Dynamic Import）的变量化技巧

```typescript
const lydell = '@lydell/node-pty';
const module = await import(lydell);
```

代码将包名赋值给变量后再传入 `import()`，而非直接写 `import('@lydell/node-pty')`。这是一种**故意绕过打包器静态分析**的技巧。Webpack、Rollup 等打包工具会静态分析 `import()` 的字符串字面量参数，将其视为需要打包的依赖。将包名存入变量后，打包器无法静态推断导入目标，从而**避免将可选依赖打入产物包中**，保持其为运行时动态加载。

### 2. 降级策略的设计意图

优先使用 `@lydell/node-pty` 而非 `node-pty` 的原因：
- `@lydell/node-pty` 是社区活跃维护的 fork，修复了原版的一些已知问题
- 在某些平台上（特别是 macOS ARM64）兼容性更好
- 两个库的 API 兼容，上层代码无需做差异化处理

### 3. 环境变量控制机制

`GEMINI_PTY_INFO=child_process` 环境变量允许用户或系统强制禁用 PTY，常用场景包括：
- CI/CD 环境中不需要 PTY
- 调试时需要使用纯 `child_process` 以获得更简单的 I/O 模型
- PTY 库存在兼容性问题时的手动降级

### 4. ESLint 抑制注释

代码中多处出现 `// eslint-disable-next-line @typescript-eslint/no-unsafe-assignment`，这是因为动态 `import()` 的返回类型为 `any`，TypeScript 的 ESLint 规则会对 `any` 类型的赋值发出警告。由于这两个库没有统一的类型定义，使用 `any` 是合理的折中方案。
