# examples -- Gemini CLI 扩展示例集合

## 概述

本目录是 Gemini CLI 扩展系统的官方示例集合，包含了多种扩展类型的参考实现。每个子目录都是一个独立的、可直接使用的扩展示例，展示了 Gemini CLI 扩展机制的不同能力：自定义命令、生命周期钩子、MCP 服务器集成以及安全策略引擎。这些示例通常由 `gemini extensions new` 命令在创建新扩展时作为模板使用。

## 目录结构

```
examples/
├── custom-commands/   # 自定义斜杠命令扩展示例
├── hooks/             # 生命周期钩子扩展示例
├── mcp-server/        # MCP (Model Context Protocol) 服务器扩展示例
├── policies/          # 安全策略引擎扩展示例
├── exclude-tools/     # 工具排除配置示例
├── skills/            # 技能扩展示例
└── themes-example/    # 主题自定义示例
```

## 核心组件

| 子目录 | 扩展名称 | 说明 |
|--------|---------|------|
| `custom-commands/` | `custom-commands` | 演示如何通过 TOML 文件定义自定义斜杠命令，支持参数模板和 shell 命令嵌入 |
| `hooks/` | `hooks-example` | 演示如何在会话生命周期事件（如 SessionStart）上执行自定义脚本 |
| `mcp-server/` | `mcp-server-example` | 演示如何将 MCP 服务器作为扩展集成，向 Gemini CLI 暴露工具和提示词 |
| `policies/` | `policy-example` | 演示如何通过策略规则和安全检查器增强 Gemini CLI 的安全控制 |
| `exclude-tools/` | - | 演示如何排除特定工具 |
| `skills/` | - | 演示如何定义可复用的技能 |
| `themes-example/` | - | 演示如何自定义 CLI 主题 |

## 依赖关系

- 所有示例扩展均依赖 Gemini CLI 扩展系统的核心规范，每个扩展根目录必须包含 `gemini-extension.json` 清单文件。
- `mcp-server/` 示例额外依赖 `@modelcontextprotocol/sdk` 和 `zod` npm 包。
- `hooks/` 示例依赖 Node.js 运行时来执行脚本。
- `policies/` 示例依赖 Gemini CLI 内置的 Policy Engine 来解析 TOML 策略规则。
- 各示例之间相互独立，无交叉依赖。
