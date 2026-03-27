# fsErrorMessages.ts

## 概述

`fsErrorMessages.ts` 是 Gemini CLI 核心包中的文件系统错误消息转换模块。该模块的核心职责是将 Node.js 底层的文件系统错误码（如 `EACCES`、`ENOENT` 等）转换为用户友好的、具有可操作指导意义的错误提示消息。

该文件位于 `packages/core/src/utils/fsErrorMessages.ts`，共 86 行代码，设计简洁且职责单一。

## 架构图（Mermaid）

```mermaid
graph TB
    subgraph fsErrorMessages["fsErrorMessages.ts 模块"]
        errorMessageGenerators["errorMessageGenerators<br/>错误码 → 消息生成器映射表"]
        getFsErrorMessage["getFsErrorMessage<br/>文件系统错误消息转换（导出）"]
    end

    subgraph 错误码映射["支持的 Node.js 错误码"]
        EACCES["EACCES<br/>权限被拒绝"]
        ENOENT["ENOENT<br/>文件或目录不存在"]
        ENOSPC["ENOSPC<br/>磁盘空间不足"]
        EISDIR["EISDIR<br/>路径是目录"]
        EROFS["EROFS<br/>只读文件系统"]
        EPERM["EPERM<br/>操作不允许"]
        EEXIST["EEXIST<br/>文件已存在"]
        EBUSY["EBUSY<br/>资源忙/被锁定"]
        EMFILE["EMFILE<br/>进程打开文件数过多"]
        ENFILE["ENFILE<br/>系统打开文件数过多"]
    end

    subgraph 内部依赖模块["内部依赖"]
        isNodeError["isNodeError<br/>Node错误类型守卫"]
        getErrorMessage["getErrorMessage<br/>通用错误消息提取"]
    end

    getFsErrorMessage -->|查询映射表| errorMessageGenerators
    getFsErrorMessage -->|类型检查| isNodeError
    getFsErrorMessage -->|回退处理| getErrorMessage
    errorMessageGenerators --> 错误码映射

    subgraph 调用方["调用方场景"]
        文件读取["文件读取工具"]
        文件写入["文件写入工具"]
        文件操作["其他文件系统操作"]
    end

    调用方 -->|捕获异常后调用| getFsErrorMessage

    style fsErrorMessages fill:#fff3e0
```

## 核心组件

### 1. `errorMessageGenerators`（内部常量）

- **类型**：`Record<string, (path?: string) => string>`
- **功能**：错误码到消息生成函数的映射表。每个生成函数接收可选的文件路径参数，返回包含路径信息和操作建议的用户友好消息。
- **支持的错误码详解**：

| 错误码 | 含义 | 消息模板（有路径） | 操作建议 |
|---|---|---|---|
| `EACCES` | 权限被拒绝 | `Permission denied: cannot access '{path}'.` | 检查文件权限或以提升权限运行 |
| `ENOENT` | 文件/目录不存在 | `File or directory not found: '{path}'.` | 检查路径是否存在及拼写 |
| `ENOSPC` | 磁盘空间不足 | （无路径信息） | 释放磁盘空间后重试 |
| `EISDIR` | 路径是目录而非文件 | `Path is a directory, not a file: '{path}'.` | 提供文件路径而非目录路径 |
| `EROFS` | 只读文件系统 | （无路径信息） | 确保文件系统允许写操作 |
| `EPERM` | 操作不允许 | `Operation not permitted: '{path}'.` | 确保具有执行操作的权限 |
| `EEXIST` | 文件/目录已存在 | `File or directory already exists: '{path}'.` | 使用不同的名称或路径 |
| `EBUSY` | 资源忙或被锁定 | `Resource busy or locked: '{path}'.` | 关闭可能正在使用该文件的程序 |
| `EMFILE` | 进程打开文件数过多 | （无路径信息） | 关闭一些未使用的文件或应用 |
| `ENFILE` | 系统打开文件数过多 | （无路径信息） | 关闭一些未使用的文件或应用 |

### 2. `getFsErrorMessage(error: unknown, defaultMessage?: string): string`

- **功能**：将任意错误对象转换为用户友好的文件系统错误消息
- **参数**：
  - `error` - 任意类型的错误对象
  - `defaultMessage` - 可选的默认消息，默认值为 `'An unknown error occurred'`
- **返回**：格式化后的错误消息字符串
- **处理流程**：

```mermaid
flowchart TD
    输入["接收 error 参数"] --> 空值检查{"error == null ?"}
    空值检查 -->|是| 返回默认["返回 defaultMessage"]
    空值检查 -->|否| Node错误检查{"isNodeError(error) ?"}
    Node错误检查 -->|是| 提取错误码["提取 error.code 和 error.path"]
    Node错误检查 -->|否| 通用处理["调用 getErrorMessage(error)"]
    提取错误码 --> 已知错误码{"code 在映射表中?"}
    已知错误码 -->|是| 生成消息["调用对应的消息生成器<br/>传入 path"]
    已知错误码 -->|否| 有错误码{"code 存在?"}
    有错误码 -->|是| 附加错误码["返回 message + (code)"]
    有错误码 -->|否| 通用处理
    生成消息 --> 返回结果["返回用户友好消息"]
    附加错误码 --> 返回结果
    通用处理 --> 返回结果
    返回默认 --> 返回结果
```

## 依赖关系

### 内部依赖

| 依赖模块 | 导入内容 | 用途 |
|---|---|---|
| `./errors.js` | `isNodeError` | 类型守卫函数，判断错误对象是否为 Node.js 风格的错误（具有 `code`、`path` 等属性） |
| `./errors.js` | `getErrorMessage` | 通用错误消息提取函数，从任意类型的错误对象中提取消息字符串 |

### 外部依赖

无。该模块不依赖任何外部第三方库，仅使用 JavaScript 内置功能。

## 关键实现细节

1. **三层回退策略**：`getFsErrorMessage` 实现了优雅的三层错误处理回退：
   - 第一层：对于已知的 Node.js 文件系统错误码，使用映射表生成详细且友好的消息
   - 第二层：对于未知的 Node.js 错误码，将原始错误消息与错误码组合返回
   - 第三层：对于非 Node.js 错误（普通 Error 或其他类型），委托给通用的 `getErrorMessage` 处理

2. **路径感知的消息生成**：大部分错误码的消息生成器都支持路径参数。当路径可用时，消息中会包含具体的文件路径信息，帮助用户快速定位问题；路径不可用时，仍能提供通用的错误描述和建议。

3. **可操作的错误提示**：每条错误消息都包含两部分 -- 问题描述和解决建议。这种设计遵循了良好的用户体验原则，不仅告诉用户"出了什么问题"，还告诉用户"如何解决"。

4. **类型安全的错误处理**：函数参数类型为 `unknown`，配合 `isNodeError` 类型守卫，实现了安全的类型缩窄。这意味着调用方无需预先判断错误类型，可以直接将 `catch` 块中的错误传入。

5. **`null` 安全**：函数对 `null` 和 `undefined` 输入进行了显式处理，返回默认消息而非抛出异常，增强了鲁棒性。

6. **使用 `Object.hasOwn` 而非 `in` 操作符**：使用更现代的 `Object.hasOwn()` 方法检查映射表中是否存在对应的错误码，避免了原型链上属性的干扰。

7. **系统级 vs 进程级错误区分**：`EMFILE`（进程级文件描述符耗尽）和 `ENFILE`（系统级文件描述符耗尽）被分别处理，虽然给出的建议相似，但区分了问题的范围。而 `ENOSPC` 和 `EROFS` 等与特定路径无关的系统级错误，其消息生成器忽略了路径参数。
