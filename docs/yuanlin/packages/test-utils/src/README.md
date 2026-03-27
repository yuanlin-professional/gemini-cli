# packages/test-utils/src

## 概述

测试工具包的核心源码目录，包含集成测试脚手架、文件系统辅助、Mock 工具、MCP 服务器构建器和测试夹具。

## 目录结构

```
src/
├── index.ts                     # 模块导出入口
├── test-rig.ts                  # TestRig + InteractiveRun 核心类
├── file-system-test-helpers.ts  # 临时文件系统创建/清理
├── mock-utils.ts                # Mock 辅助工具
├── test-mcp-server.ts           # MCP 测试服务器构建器
└── fixtures/
    └── agents.ts                # 预定义测试 Agent 夹具
```

## 核心组件

### TestRig 核心特性

| 方法 | 说明 |
|------|------|
| `setup(name, options)` | 初始化测试环境 |
| `run(options)` | 非交互式运行 CLI |
| `runInteractive(options)` | 交互式运行（PTY） |
| `runCommand(args)` | 执行 CLI 命令 |
| `createFile(name, content)` | 在测试目录创建文件 |
| `addTestMcpServer(name, config)` | 添加测试 MCP 服务器 |
| `readToolLogs()` | 读取工具调用遥测 |
| `waitForToolCall(name)` | 等待工具调用完成 |
| `readHookLogs()` | 读取 Hook 调用遥测 |
| `cleanup()` | 清理测试环境 |

### FileSystemStructure

声明式文件系统结构类型，支持：
- 字符串值 -> 创建文件（内容为该字符串）
- 对象值 -> 创建目录（递归创建子结构）
- 数组值 -> 创建目录（字符串元素为空文件，对象元素为子目录）

### TestMcpServerBuilder

链式 API，通过 `addTool()` 添加工具定义，`build()` 输出 `TestMcpConfig`。

## 依赖关系

### 内部依赖
- `@google/gemini-cli-core` - 常量定义

### 外部依赖
- `@lydell/node-pty` - PTY 终端模拟
- `strip-ansi` - ANSI 清理
- `vitest` - 测试断言
