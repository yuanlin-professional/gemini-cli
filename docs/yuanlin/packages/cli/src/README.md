# packages/cli/src/

## 概述

这是 Gemini CLI 的主入口源码目录。该目录包含 CLI 应用的完整启动流程、交互式 UI 渲染引擎、非交互式执行管道，以及配置管理、服务层和工具函数等核心模块。入口函数 `main()` 位于 `gemini.tsx` 中，负责整个应用的生命周期管理——从参数解析、认证、沙箱启动到 UI 渲染。

## 目录结构

```
src/
├── gemini.tsx                    # 应用主入口，包含 main() 函数和完整启动流程
├── interactiveCli.tsx            # 交互式 UI 启动器，使用 React/Ink 渲染终端界面
├── nonInteractiveCli.ts          # 非交互式（管道/脚本）模式执行逻辑
├── nonInteractiveCliCommands.ts  # 非交互式命令处理
├── validateNonInterActiveAuth.ts # 非交互式模式的认证校验
├── deferred.ts                   # 延迟命令执行
├── acp/                          # ACP（Admin Control Plane）客户端
├── commands/                     # CLI 命令定义
├── config/                       # 配置加载（settings、auth、sandbox、policy 等）
├── core/                         # 核心初始化逻辑（initializer）
├── patches/                      # 运行时补丁
├── services/                     # 服务层（如 SlashCommandConflictHandler）
├── ui/                           # 完整的终端 UI 系统（组件、主题、hooks、上下文）
├── utils/                        # 工具函数（cleanup、sandbox、session、relaunch 等）
├── integration-tests/            # 集成测试
└── test-utils/                   # 测试辅助工具
```

## 核心组件

### gemini.tsx — 应用主入口
- `main()` 函数是整个 CLI 的启动入口，执行以下关键步骤：
  1. 设置信号处理器和未捕获 Promise 拒绝处理器
  2. 加载用户设置（`loadSettings`）和 CLI 配置（`loadCliConfig`）
  3. 处理 DNS 解析顺序配置、认证刷新（OAuth/ADC/外部认证）
  4. 根据配置决定是否进入沙箱模式（`start_sandbox`），或在子进程中重新启动以获得更多内存
  5. 区分交互式/非交互式模式，分别调用 `startInteractiveUI` 或 `runNonInteractive`
  6. 管理会话恢复（`--resume`）、会话列表（`--list-sessions`）、会话删除等功能
- `initializeOutputListenersAndFlush()` 确保在非交互模式下标准输出/错误输出不丢失
- `setupAdminControlsListener()` 通过 IPC 监听来自父进程的管理控制设置

### interactiveCli.tsx — 交互式 UI 引擎
- 基于 React + Ink 渲染终端 UI
- 构建多层 Context Provider 树：SettingsContext -> KeyMatchersProvider -> KeypressProvider -> MouseProvider -> TerminalProvider -> ScrollProvider -> OverflowProvider -> SessionStatsProvider -> VimModeProvider -> AppContainer
- 支持备用屏幕缓冲区（alternate buffer）、鼠标事件、Kitty 键盘协议
- 集成自动更新检查、慢渲染监控（>200ms 阈值）、窗口标题动态管理

## 依赖关系

- **@google/gemini-cli-core**: 核心库，提供 Config、事件系统（coreEvents）、认证、日志、会话管理等基础能力
- **React / Ink**: 终端 UI 渲染框架（仅交互模式下动态加载）
- **内部模块依赖**: `config/` 提供配置和认证 -> `core/` 执行初始化 -> `ui/` 渲染界面 -> `utils/` 提供清理/沙箱/会话等支撑功能
- **node:crypto / node:os / node:dns / v8**: Node.js 原生模块，用于哈希、内存管理和 DNS 配置
