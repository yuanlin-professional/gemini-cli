# mcp-server -- MCP 服务器扩展示例

## 概述

本目录是一个 Gemini CLI 扩展示例，演示如何将 MCP（Model Context Protocol）服务器作为扩展集成到 Gemini CLI 中。该扩展名为 `mcp-server-example`（版本 1.0.0），展示了通过 MCP 协议向 Gemini CLI 暴露自定义工具（Tool）和提示词（Prompt）的完整流程。MCP 是一种标准化协议，允许外部服务器为 AI 模型提供工具调用和提示词模板能力。

## 目录结构

```
mcp-server/
├── .gitignore                # Git 忽略规则
├── gemini-extension.json     # 扩展清单文件，含 MCP 服务器配置
├── package.json              # npm 包描述，声明依赖
├── example.js                # MCP 服务器主入口文件
└── README.md                 # 原始英文说明文档
```

## 核心组件

### `gemini-extension.json`

扩展清单文件，通过 `mcpServers` 字段声明 MCP 服务器配置：

```json
{
  "name": "mcp-server-example",
  "version": "1.0.0",
  "mcpServers": {
    "nodeServer": {
      "command": "node",
      "args": ["${extensionPath}${/}example.js"],
      "cwd": "${extensionPath}"
    }
  }
}
```

- **`mcpServers.nodeServer`**：定义了一个名为 `nodeServer` 的 MCP 服务器实例。
- **`command` / `args`**：通过 `node example.js` 启动服务器进程。
- **`${extensionPath}`**：内置路径变量，运行时替换为扩展实际安装路径。
- **`${/}`**：跨平台路径分隔符变量。

### `example.js`

MCP 服务器主入口，基于 `@modelcontextprotocol/sdk` 实现，注册了两个能力：

1. **工具 `fetch_posts`**：从公共 API (`jsonplaceholder.typicode.com/posts`) 获取帖子列表，返回前 5 条数据。无需输入参数。
2. **提示词 `poem-writer`**：生成俳句（Haiku）的提示词模板，接受 `title`（必填）和 `mood`（可选）两个参数，生成符合 5-7-5 音节格式的俳句写作指令。

服务器通过 `StdioServerTransport` 使用标准输入/输出进行通信，这是 Gemini CLI 与 MCP 服务器交互的标准方式。

### `package.json`

声明运行时依赖：

- `@modelcontextprotocol/sdk` (^1.23.0)：MCP 协议的官方 SDK。
- `zod` (^3.22.4)：用于定义工具输入和提示词参数的 schema 验证。

## 依赖关系

- 依赖 Gemini CLI 扩展系统的 MCP 服务器集成机制（通过 `gemini-extension.json` 中的 `mcpServers` 配置）。
- 依赖 `@modelcontextprotocol/sdk` npm 包提供 MCP 服务器框架。
- 依赖 `zod` npm 包进行输入参数的 schema 定义与验证。
- 依赖 Node.js 运行时（ES Module 模式，`"type": "module"`）。
- 使用前需执行 `npm install` 安装依赖。
