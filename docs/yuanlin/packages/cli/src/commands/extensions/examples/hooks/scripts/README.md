# scripts

## 概述

`scripts` 是 hooks 扩展的脚本执行目录，存放由钩子事件触发时实际运行的脚本文件。这些脚本通过 `hooks.json` 中的配置被关联到特定的生命周期事件，在事件触发时由 Node.js 运行时执行。

## 目录结构

```
scripts/
└── on-start.js    # 会话启动时执行的脚本
```

## 核心组件

- **on-start.js** - 会话启动钩子脚本。由 `SessionStart` 事件触发执行，功能为在控制台输出一条提示信息：`Session Started! This is running from a script in the hooks-example extension.`。该脚本作为示例，展示了扩展如何在 CLI 会话开始时注入自定义行为，实际应用中可替换为环境初始化、配置检查、依赖验证等逻辑。
