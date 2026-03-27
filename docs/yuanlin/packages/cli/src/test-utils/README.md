# test-utils

## 概述

CLI 包的测试工具库，提供一整套用于编写和运行 UI 集成测试的基础设施。核心是 `AppRig` 类，它能模拟完整的 Gemini CLI 应用环境，包括配置、认证、工具调度、消息总线、Shell 命令模拟等，使测试能以程序化方式与应用交互。

## 目录结构

```
test-utils/
├── AppRig.tsx                    # 核心测试装置类，模拟完整应用环境
├── AppRig.test.tsx               # AppRig 自身的单元测试
├── async.ts                      # 异步辅助工具
├── createExtension.ts            # 扩展创建辅助函数
├── customMatchers.ts             # 自定义 Vitest 匹配器
├── fixtures/                     # 测试数据夹具（假响应文件等）
├── mockCommandContext.ts         # 命令上下文模拟
├── mockCommandContext.test.ts    # 命令上下文模拟的测试
├── mockConfig.ts                 # 配置模拟
├── mockDebugLogger.ts            # 调试日志模拟
├── MockShellExecutionService.ts  # Shell 执行服务模拟，可拦截/模拟 Shell 命令
├── persistentStateFake.ts        # 持久化状态伪实现
├── render.tsx                    # 带 Provider 的渲染辅助函数
├── render.test.tsx               # 渲染辅助的测试
├── settings.ts                   # 设置模拟辅助
└── svg.ts                        # SVG 相关辅助
```

## 核心组件

- **AppRig.tsx**：测试工具库的核心类。它封装了完整的应用生命周期管理，包括：创建隔离的临时测试目录、mock 认证流程（跳过真实登录）、mock `StreamingContext` 追踪状态变化、mock `ShellExecutionService` 拦截 Shell 命令、配置策略引擎以控制工具审批行为。提供丰富的 API 接口：`initialize()`/`render()`/`unmount()` 管理生命周期；`type()`/`pressEnter()`/`sendMessage()` 模拟用户输入；`waitForOutput()`/`waitForIdle()`/`waitForPendingConfirmation()` 等待特定状态；`resolveTool()`/`setBreakpoint()`/`drainBreakpointsUntilIdle()` 控制工具执行流程；`addUserHint()` 注入模型引导。清理时自动删除临时目录并重置全局状态。
