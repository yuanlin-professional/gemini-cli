# noninteractive - 非交互式 UI 模块

## 概述

`noninteractive` 目录提供了 Gemini CLI 在非交互式环境（如 CI/CD 管道、脚本自动化、无终端模式）下运行时的 UI 上下文实现。该模块通过创建一组空操作（no-op）函数来替代交互式 UI 组件，确保命令在无终端交互能力的场景下仍能正常执行，同时将关键信息（错误、警告、信息）通过标准输出/错误流进行输出。

## 目录结构

```
noninteractive/
└── nonInteractiveUi.ts    # 非交互式 UI 上下文工厂函数
```

## 架构图

```mermaid
graph TD
    subgraph 非交互式UI模块
        NUI[createNonInteractiveUI<br/>非交互式UI工厂]
    end

    subgraph 输出通道
        STDOUT[process.stdout<br/>标准输出]
        STDERR[process.stderr<br/>标准错误]
    end

    subgraph 命令系统
        CC[CommandContext<br/>命令上下文]
        UI_IFACE["CommandContext['ui']<br/>UI接口"]
    end

    subgraph 交互式模式
        INK[Ink React组件<br/>交互式UI]
    end

    CC -->|交互模式| INK
    CC -->|非交互模式| NUI
    NUI -->|实现| UI_IFACE
    INK -->|实现| UI_IFACE

    NUI -->|info消息| STDOUT
    NUI -->|error/warning| STDERR
```

## 核心组件

### nonInteractiveUi.ts - 非交互式 UI 工厂

导出 `createNonInteractiveUI()` 工厂函数，返回符合 `CommandContext['ui']` 接口的对象。

**有效输出行为（addItem）：**
- `error` 类型消息 → 写入 `process.stderr`，前缀 `Error: `
- `warning` 类型消息 → 写入 `process.stderr`，前缀 `Warning: `
- `info` 类型消息 → 写入 `process.stdout`

**空操作函数（no-op）：**

以下 UI 操作在非交互模式下不执行任何动作：
| 函数 | 原交互式用途 |
|------|-------------|
| `clear()` | 清屏 |
| `setDebugMessage()` | 设置调试信息 |
| `loadHistory()` | 加载历史记录 |
| `setPendingItem()` | 设置待处理条目 |
| `toggleCorgiMode()` | 切换柯基模式 |
| `toggleDebugProfiler()` | 切换调试分析器 |
| `toggleVimEnabled()` | 切换 Vim 模式（返回 false） |
| `reloadCommands()` | 重载命令 |
| `openAgentConfigDialog()` | 打开 Agent 配置对话框 |
| `setConfirmationRequest()` | 设置确认请求 |
| `removeComponent()` | 移除组件 |
| `toggleBackgroundShell()` | 切换后台 Shell |
| `toggleShortcutsHelp()` | 切换快捷键帮助 |
| `dispatchExtensionStateUpdate()` | 分发扩展状态更新 |
| `addConfirmUpdateExtensionRequest()` | 添加扩展更新确认 |

## 依赖关系

| 依赖 | 用途 |
|------|------|
| `../commands/types.js` | `CommandContext` 类型定义，确保 UI 接口契约一致 |
| `../state/extensions.js` | `ExtensionUpdateAction` 类型 |

## 数据流

```mermaid
sequenceDiagram
    participant 调用方 as CLI入口/命令系统
    participant 工厂 as createNonInteractiveUI()
    participant UI as 非交互式UI对象
    participant 命令 as SlashCommand.action
    participant STDOUT as process.stdout
    participant STDERR as process.stderr

    调用方->>工厂: 检测到非交互模式
    工厂-->>调用方: 返回UI对象(no-op实现)
    调用方->>命令: 执行命令(context含非交互UI)
    命令->>UI: addItem({type:'info', text:'...'})
    UI->>STDOUT: 写入信息文本
    命令->>UI: addItem({type:'error', text:'...'})
    UI->>STDERR: 写入错误文本
    命令->>UI: clear() / toggleCorgiMode() / ...
    Note over UI: 空操作，不执行任何动作
```
