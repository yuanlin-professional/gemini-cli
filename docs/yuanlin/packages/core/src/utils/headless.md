# headless.ts

## 概述

`headless.ts` 是 Gemini CLI 核心包中的一个实用工具模块，负责检测 CLI 是否运行在 **无头模式（Headless Mode）**，即非交互式环境下。无头模式意味着没有用户在终端前实时输入，常见于 CI/CD 流水线、管道命令、脚本自动化等场景。

该模块导出一个接口 `HeadlessModeOptions` 和一个核心检测函数 `isHeadlessMode`，通过多重判断策略（环境变量、TTY 检测、命令行参数）来综合判定当前运行环境是否为无头模式。

## 架构图（Mermaid）

```mermaid
flowchart TD
    A[调用 isHeadlessMode] --> B{是否为集成测试?}
    B -- 是 GEMINI_CLI_INTEGRATION_TEST=true --> C[跳过 CI 检测]
    B -- 否 --> D{CI 环境变量检测}
    D -- CI=true 或 GITHUB_ACTIONS=true --> E[返回 true: 无头模式]
    D -- 否 --> C
    C --> F{TTY 检测}
    F -- stdin 非 TTY 或 stdout 非 TTY --> E
    F -- 均为 TTY --> G{选项参数检测}
    G -- options.prompt 或 options.query 存在 --> E
    G -- 均不存在 --> H{进程参数回退检测}
    H -- argv 包含 -p 或 --prompt --> E
    H -- 均不包含 --> I[返回 false: 交互模式]
```

## 核心组件

### 1. `HeadlessModeOptions` 接口

```typescript
export interface HeadlessModeOptions {
  prompt?: string | boolean;
  query?: string | boolean;
}
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `prompt` | `string \| boolean` | 显式的提示字符串或标志。当通过 `-p` 或 `--prompt` 传入时，表明用户以非交互方式提供了输入 |
| `query` | `string \| boolean` | 初始查询位置参数。当直接在命令行传入查询内容时，表明不需要交互式输入 |

两个属性均为可选，当任一存在且为真值时，即认定为无头模式。

### 2. `isHeadlessMode` 函数

```typescript
export function isHeadlessMode(options?: HeadlessModeOptions): boolean
```

核心检测函数，通过以下 **四层判断策略** 逐步判定：

#### 第一层：CI 环境检测（可被集成测试绕过）

```typescript
if (process.env['GEMINI_CLI_INTEGRATION_TEST'] !== 'true') {
  const isCI =
    process.env['CI'] === 'true' || process.env['GITHUB_ACTIONS'] === 'true';
  if (isCI) {
    return true;
  }
}
```

- 检查环境变量 `CI` 是否为 `'true'`，或 `GITHUB_ACTIONS` 是否为 `'true'`
- 特殊处理：当 `GEMINI_CLI_INTEGRATION_TEST` 为 `'true'` 时，跳过 CI 检测。这是为了让集成测试能在 CI 环境中以交互模式运行，避免因 CI 环境自动触发无头模式而干扰测试

#### 第二层：TTY 检测

```typescript
const isNotTTY =
  (!!process.stdin && !process.stdin.isTTY) ||
  (!!process.stdout && !process.stdout.isTTY);
```

- 检查 `process.stdin` 和 `process.stdout` 是否为 TTY（终端）
- 当标准输入或标准输出不是 TTY 时（例如通过管道传输），判定为无头模式
- 使用 `!!process.stdin` 先确保流对象存在，避免空引用

#### 第三层：选项参数检测

```typescript
if (isNotTTY || !!options?.prompt || !!options?.query) {
  return true;
}
```

- 检查调用方传入的 `options` 对象中 `prompt` 或 `query` 是否存在且为真值
- 当用户通过命令行传入了提示或查询内容时，说明不需要交互式输入

#### 第四层：进程参数回退检测

```typescript
return process.argv.some((arg) => arg === '-p' || arg === '--prompt');
```

- 作为最终回退策略，直接扫描 `process.argv` 原始参数数组
- 查找 `-p` 或 `--prompt` 标志
- 这层检测覆盖了 `options` 参数未被正确传递但命令行确实包含相关标志的边界情况

## 依赖关系

### 内部依赖

无。该模块是一个纯工具函数，不依赖项目内其他模块。

### 外部依赖

| 依赖 | 说明 |
|------|------|
| `node:process` | Node.js 内置模块，用于访问环境变量 (`process.env`)、标准流 (`process.stdin`, `process.stdout`) 及命令行参数 (`process.argv`) |

## 关键实现细节

1. **集成测试豁免机制**：通过 `GEMINI_CLI_INTEGRATION_TEST` 环境变量，允许集成测试在 CI 环境中绕过无头模式检测。这是一个精心设计的机制——CI 环境中 `CI=true` 通常会自动被设置，但集成测试可能需要模拟交互式行为，因此需要一个逃逸通道。

2. **多层防御式检测**：函数采用从环境变量到 TTY 状态再到命令行参数的渐进式检测策略，确保在各种运行场景下都能正确判定。每一层都是上一层的补充，形成完整的覆盖。

3. **双重 TTY 检查**：同时检查 `stdin` 和 `stdout` 的 TTY 状态。例如，当命令通过管道链接时（如 `echo "query" | gemini-cli`），`stdin` 不是 TTY；当输出被重定向时（如 `gemini-cli > output.txt`），`stdout` 不是 TTY。两种情况都应触发无头模式。

4. **回退策略的必要性**：第四层直接检查 `process.argv` 看似冗余，但当 `isHeadlessMode` 被调用时未传入 `options`（或 `options` 解析出错），该回退能确保 `-p`/`--prompt` 标志仍然被正确识别。

5. **严格字符串比较**：环境变量检测使用 `=== 'true'` 而非简单的真值检查，避免环境变量被设置为其他非 `'true'` 值时的误判。
