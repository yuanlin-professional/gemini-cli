# packages/

## 概述

这是 Gemini CLI 项目的 monorepo 包目录，采用多包架构组织代码。包含 7 个子包，涵盖 CLI 主程序、核心引擎、SDK、A2A 服务器、开发者工具、VS Code 扩展和测试工具。各子包通过 `file:` 协议引用本地依赖，均要求 Node.js >= 20，统一使用 ESModule（`"type": "module"`）。

## 目录结构

```
packages/
├── cli/                    # @google/gemini-cli — CLI 主程序（可执行入口 gemini）
├── core/                   # @google/gemini-cli-core — 核心引擎库
├── sdk/                    # @google/gemini-cli-sdk — 开发者 SDK
├── a2a-server/             # @google/gemini-cli-a2a-server — A2A（Agent-to-Agent）服务器
├── devtools/               # @google/gemini-cli-devtools — 开发者调试工具（WebSocket + Web 客户端）
├── test-utils/             # @google/gemini-cli-test-utils — 测试工具库（私有包）
└── vscode-ide-companion/   # gemini-cli-vscode-ide-companion — VS Code IDE 伴侣扩展
```

## 核心组件

### cli — CLI 主程序
- 包名：`@google/gemini-cli`，提供 `gemini` 可执行命令
- 基于 React + Ink 构建终端交互 UI，支持交互式和非交互式两种模式
- 集成代码高亮（highlight.js/lowlight）、渐变渲染（ink-gradient/tinygradient）、剪贴板（clipboardy）等终端增强能力
- 依赖 `@google/gemini-cli-core` 作为核心引擎，`@google/genai` 作为 AI SDK
- 支持沙箱运行模式（通过 Docker 镜像隔离）

### core — 核心引擎
- 包名：`@google/gemini-cli-core`，是所有其他包的基础依赖
- 提供 AI 模型交互（`@google/genai`）、MCP 协议支持（`@modelcontextprotocol/sdk`）、A2A 协议（`@a2a-js/sdk`）
- 集成 OpenTelemetry 全链路可观测性（traces、metrics、logs），支持 gRPC 和 HTTP 两种导出方式
- 内置文件搜索（ripgrep）、diff 对比、YAML/TOML/JSON 解析、Git 操作（simple-git）等工具链
- 支持伪终端（node-pty）、浏览器自动化（puppeteer-core）、代理（https-proxy-agent）等高级功能

### sdk — 开发者 SDK
- 包名：`@google/gemini-cli-sdk`，为扩展和插件开发者提供编程接口
- 轻量级封装，仅依赖 `@google/gemini-cli-core` 和 Zod（schema 校验）

### a2a-server — A2A 服务器
- 包名：`@google/gemini-cli-a2a-server`，实现 Agent-to-Agent 通信协议
- 基于 Express 5 构建 HTTP 服务，集成 Google Cloud Storage 和 Winston 日志
- 支持通过 tar 打包传输数据

### devtools — 开发者调试工具
- 包名：`@google/gemini-cli-devtools`，提供 WebSocket 服务和 Web 客户端
- 使用 React/ReactDOM 构建调试界面，通过 esbuild 打包客户端资源

### test-utils — 测试工具
- 包名：`@google/gemini-cli-test-utils`，私有包不发布到 npm
- 提供基于 vitest 的测试辅助工具，集成 node-pty 用于终端模拟测试和 strip-ansi 用于输出清理

### vscode-ide-companion — VS Code 伴侣扩展
- 包名：`gemini-cli-vscode-ide-companion`，发布者为 google
- 为 Gemini CLI 提供 IDE 集成能力：diff 查看/接受/取消、直接运行 CLI
- 基于 MCP 协议（`@modelcontextprotocol/sdk`）与 CLI 通信，通过 Express 提供本地服务
- 支持快捷键绑定（Ctrl/Cmd+S 接受 diff）

## 依赖关系

```
cli ──────────> core <──────── a2a-server
                 ^
                 |
sdk ─────────────┘
                 ^
                 |
test-utils ──────┘

devtools        (独立，无内部依赖)
vscode-ide-companion (独立，无内部依赖)
```

- **core** 是基础包，被 `cli`、`sdk`、`a2a-server`、`test-utils` 依赖
- **cli** 是最终面向用户的可执行包，依赖 `core`，开发时依赖 `test-utils`
- **devtools** 和 **vscode-ide-companion** 相对独立，不依赖其他内部包
- 所有包统一使用 `vitest` 作为测试框架，`typescript` 进行类型检查
