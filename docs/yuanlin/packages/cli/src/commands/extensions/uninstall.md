# uninstall.ts

## 概述

`uninstall.ts` 实现了 Gemini CLI 扩展系统中的 **卸载扩展（uninstall）** 命令。该命令支持三种卸载模式：

1. **按名称卸载**：指定一个或多个扩展名称/源路径进行卸载。
2. **全部卸载**：使用 `--all` 标志一次性卸载所有已安装的扩展。
3. **批量卸载**：传入多个扩展名称，依次逐个卸载，并汇总报告错误。

该文件导出两个核心成员：
- `handleUninstall` 异步函数：卸载逻辑的核心处理器
- `uninstallCommand` 对象：符合 yargs `CommandModule` 接口的命令定义

## 架构图（Mermaid）

```mermaid
graph TD
    A[用户执行 gemini extensions uninstall] --> B[yargs 解析命令参数]
    B --> C{参数校验 check}
    C -- "无 --all 且无 names" --> C1[抛出错误: 请指定扩展名或使用 --all]
    C -- 校验通过 --> D[uninstallCommand handler]
    D --> E[handleUninstall 函数]
    E --> F[创建 ExtensionManager 实例]
    F --> G[加载已安装扩展 loadExtensions]
    G --> H{确定卸载列表}
    H -- "--all 标志" --> I[获取所有已安装扩展名称]
    H -- "指定 names" --> J[去重后使用名称列表]
    I --> K{卸载列表是否为空?}
    J --> K
    K -- "空且是 --all" --> L[输出 '无已安装扩展' 提示]
    K -- 空且非 --all --> L2[静默返回]
    K -- 非空 --> M[遍历卸载列表]
    M --> N[逐个调用 uninstallExtension]
    N --> O{单个卸载结果}
    O -- 成功 --> P[输出成功消息]
    O -- 失败 --> Q[收集错误到 errors 数组]
    P --> R{还有下一个?}
    Q --> R
    R -- 是 --> N
    R -- 否 --> S{errors 数组是否有内容?}
    S -- 有错误 --> T[逐个输出错误消息并 exit 1]
    S -- 无错误 --> U[调用 exitCli 正常退出]

    style A fill:#e1f5fe
    style P fill:#c8e6c9
    style T fill:#ffcdd2
    style L fill:#fff9c4
```

## 核心组件

### 1. `UninstallArgs` 接口

```typescript
interface UninstallArgs {
  names?: string[]; // 可以是扩展名称或源 URL
  all?: boolean;
}
```

- **`names`**（可选）：要卸载的扩展名称或源路径数组。支持同时传入多个。
- **`all`**（可选）：布尔值，为 `true` 时卸载所有已安装的扩展。

### 2. `handleUninstall(args: UninstallArgs)` 异步函数

这是卸载命令的核心业务逻辑。执行流程如下：

**第一阶段 -- 初始化：**
1. 获取当前工作目录，创建 `ExtensionManager` 实例。
2. 调用 `loadExtensions()` 加载已安装扩展列表。

**第二阶段 -- 确定卸载目标：**
1. 若 `args.all` 为 `true`：从 `extensionManager.getExtensions()` 获取所有已安装扩展的名称。
2. 若指定了 `args.names`：使用 `new Set(args.names)` 去重后转为数组，避免重复卸载同一扩展。

**第三阶段 -- 空列表处理：**
- 若卸载列表为空且使用了 `--all`，输出友好提示 `'No extensions currently installed.'` 后返回。
- 若卸载列表为空且未使用 `--all`，静默返回（理论上不会触发此分支，因为 yargs check 已拦截）。

**第四阶段 -- 逐个卸载：**
1. 遍历 `namesToUninstall` 数组，对每个名称调用 `extensionManager.uninstallExtension(name, false)`。
2. 成功时输出成功消息。
3. 失败时将错误信息收集到 `errors` 数组中，**不立即中断**，继续处理剩余扩展。

**第五阶段 -- 错误汇总：**
- 若 `errors` 数组非空，逐条输出所有失败的卸载错误，然后以 `process.exit(1)` 退出。
- 外层还有一个 try-catch 捕获初始化阶段的异常。

### 3. `uninstallCommand: CommandModule` 对象

| 属性 | 值 | 说明 |
|------|-----|------|
| `command` | `'uninstall [names..]'` | 命令格式，`[names..]` 为可选的可变长度位置参数 |
| `describe` | `'Uninstalls one or more extensions.'` | 命令描述 |
| `builder` | 函数 | 配置 `names` 位置参数和 `--all` 选项，含参数校验 |
| `handler` | 异步函数 | 解析参数后调用 `handleUninstall`，然后 `exitCli()` |

**命令行参数详情：**

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `names` | `string[]` | 否（位置参数，可变长度） | - | 要卸载的扩展名称或源路径 |
| `--all` | `boolean` | 否 | `false` | 卸载所有已安装扩展 |

**参数校验逻辑（`.check()`）：**
- 若 `--all` 未设置 **且** `names` 为空或未提供，抛出错误提示用户必须指定至少一个扩展名或使用 `--all` 标志。
- 确保命令不会在没有任何卸载目标的情况下执行。

## 依赖关系

### 内部依赖

| 模块路径 | 导入内容 | 用途 |
|----------|----------|------|
| `@google/gemini-cli-core` | `debugLogger`, `getErrorMessage` | 调试日志输出、错误消息安全提取 |
| `../../config/extensions/consent.js` | `requestConsentNonInteractive` | 非交互式同意请求函数（ExtensionManager 构造参数） |
| `../../config/extension-manager.js` | `ExtensionManager` | 扩展管理器，负责加载/卸载扩展 |
| `../../config/settings.js` | `loadSettings` | 加载项目与全局合并设置 |
| `../../config/extensions/extensionSettings.js` | `promptForSetting` | 提示用户输入扩展所需配置项 |
| `../utils.js` | `exitCli` | CLI 退出清理函数 |

### 外部依赖

| 包名 | 导入内容 | 用途 |
|------|----------|------|
| `yargs` | `CommandModule`（类型） | yargs 命令模块类型定义 |

## 关键实现细节

1. **容错式批量卸载**：与许多 CLI 工具"遇到第一个错误就停止"的行为不同，`handleUninstall` 采用了"尽可能多卸载"的策略。即使某个扩展卸载失败，也会继续尝试卸载其余扩展，最后才汇总报告所有错误。这在批量操作中非常有用，避免因单个扩展的问题导致整个操作中断。

2. **名称去重处理**：`[...new Set(args.names)]` 确保用户即使重复传入同一扩展名称，也只会尝试卸载一次，避免出现"扩展已被卸载"的二次错误。

3. **双重身份标识**：`names` 参数既可以是扩展名称也可以是源路径/URL（如注释所述 `can be extension names or source URLs`），提供了灵活的标识方式。

4. **`uninstallExtension` 的第二个参数**：调用 `extensionManager.uninstallExtension(name, false)` 时传入 `false` 作为第二个参数，该参数可能控制是否同时移除扩展的配置/数据文件（非强制清理模式）。

5. **参数互斥校验**：`builder` 中的 `.check()` 验证器确保 `--all` 和 `names` 参数之间至少有一个被提供。这种校验在 yargs 层面就阻止了无效调用，提供了清晰的错误提示。

6. **yargs 可变长度参数**：命令格式中的 `[names..]` 使用了 yargs 的可变参数语法（variadic），允许用户在命令行中传入任意数量的扩展名称，例如 `gemini extensions uninstall ext1 ext2 ext3`。

7. **分层错误处理**：代码有两层 try-catch。内层 try-catch 在 `for` 循环中捕获单个扩展卸载的异常并收集；外层 try-catch 捕获初始化阶段（如创建 ExtensionManager 或加载扩展时）的全局异常。
