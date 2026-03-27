# ui/hooks/shell-completions/

## 概述

该目录实现了 Gemini CLI 交互式 Shell 中的命令参数自动补全功能。通过可插拔的 Provider 机制，为特定的 shell 命令（如 `git`、`npm`）提供上下文感知的参数补全建议。当用户在内置 Shell 中输入命令时，系统会根据当前输入的命令名匹配对应的 Provider，并异步返回补全建议列表。

## 目录结构

```
shell-completions/
├── index.ts              # 入口文件，注册所有 Provider 并导出 getArgumentCompletions 函数
├── types.ts              # 类型定义：ShellCompletionProvider 接口和 CompletionResult 结构
├── gitProvider.ts         # Git 命令补全 Provider（子命令 + 分支名）
├── gitProvider.test.ts    # Git Provider 单元测试
├── npmProvider.ts         # NPM 命令补全 Provider
└── npmProvider.test.ts    # NPM Provider 单元测试
```

## 核心组件

### index.ts — 入口与调度
- 维护一个 `providers` 数组（当前包含 `gitProvider` 和 `npmProvider`）
- 导出 `getArgumentCompletions(commandToken, tokens, cursorIndex, cwd, signal?)` 函数
- 根据 `commandToken`（用户输入的命令名）查找匹配的 Provider，调用其 `getCompletions` 方法返回补全结果
- 未匹配时返回 `null`，表示退回到默认的文件路径补全

### types.ts — 类型定义
- **CompletionResult**: 包含 `suggestions` 建议列表和 `exclusive` 标志（为 `true` 时阻止系统附加通用文件补全）
- **ShellCompletionProvider**: Provider 接口，定义 `command` 触发词和 `getCompletions` 异步方法

### gitProvider.ts — Git 命令补全
- 第一级补全：输入 `git ` 后补全子命令（`add`、`branch`、`checkout`、`commit`、`diff`、`merge`、`pull`、`push`、`rebase`、`status`、`switch`）
- 第二级补全：对 `checkout`、`switch`、`merge`、`branch` 子命令，通过执行 `git branch --format=%(refname:short)` 获取本地分支列表进行补全
- 支持 `AbortSignal` 用于取消正在执行的 git 命令
- 错误时静默返回空建议（如非 git 仓库目录）

### npmProvider.ts — NPM 命令补全
- 为 `npm` 命令提供参数补全支持

## 依赖关系

- **node:child_process / node:util**: 用于异步执行 `git branch` 等外部命令获取补全数据
- **../useShellCompletion.js**: 提供 `escapeShellPath` 工具函数，用于转义分支名中的特殊字符
- **../../components/SuggestionsDisplay.js**: 提供 `Suggestion` 类型定义，定义建议项的 `label`、`value`、`description` 结构
