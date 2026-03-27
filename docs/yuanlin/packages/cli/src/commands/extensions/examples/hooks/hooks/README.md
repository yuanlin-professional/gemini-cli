# hooks

## 概述

`hooks` 是 hooks 扩展的钩子定义目录，包含钩子事件的 JSON 配置文件。钩子机制允许扩展在 Gemini CLI 生命周期的特定时机（如会话开始、工具调用前后等）自动执行自定义逻辑。配置文件将生命周期事件映射到要执行的命令。

## 目录结构

```
hooks/
└── hooks.json    # 钩子事件配置文件
```

## 核心组件

- **hooks.json** - 钩子配置文件。定义了 `SessionStart` 事件的处理器，当 Gemini CLI 会话启动时，自动执行 `node ${extensionPath}/scripts/on-start.js` 命令。配置中使用 `${extensionPath}` 变量引用扩展的安装路径，确保脚本路径在不同环境下始终正确。钩子类型为 `command`，表示直接执行 shell 命令。
