# packages/vscode-ide-companion/src/utils

## 概述

VS Code 伴侣扩展的工具函数目录，包含条件日志工具。

## 目录结构

```
utils/
└── logger.ts    # 条件日志创建函数
```

## 核心组件

### createLogger (`logger.ts`)

创建一个条件日志函数，仅在以下情况下输出日志：

1. **开发模式** - `context.extensionMode === ExtensionMode.Development`
2. **日志配置启用** - `gemini-cli.debug.logging.enabled` 设置为 `true`

日志输出到 VS Code 的 Output Channel（`Gemini CLI IDE Companion`）。

```typescript
const log = createLogger(context, logger);
log('IDE server started');  // 仅在开发模式或日志启用时输出
```

## 依赖关系

### 外部依赖
- `vscode` - VS Code 扩展 API（ExtensionContext, OutputChannel, workspace.getConfiguration）
