# hooks -- 生命周期钩子扩展示例

## 概述

本目录是一个 Gemini CLI 扩展示例，演示如何利用生命周期钩子（Hooks）在特定事件发生时执行自定义逻辑。该扩展名为 `hooks-example`（版本 1.0.0），展示了最基础的钩子用法：在会话启动（`SessionStart`）时运行一个 Node.js 脚本。钩子机制允许扩展在 Gemini CLI 的关键生命周期节点注入自定义行为，例如初始化环境、加载配置或输出欢迎信息。

## 目录结构

```
hooks/
├── .gitignore                # Git 忽略规则
├── gemini-extension.json     # 扩展清单文件
├── hooks/                    # 钩子配置目录
│   └── hooks.json            # 钩子事件与处理器的映射配置
└── scripts/                  # 钩子脚本目录
    └── on-start.js           # SessionStart 事件的处理脚本
```

## 核心组件

### `gemini-extension.json`

扩展清单文件：

```json
{
  "name": "hooks-example",
  "version": "1.0.0"
}
```

### `hooks/hooks.json`

钩子配置文件，定义了事件到处理器的映射关系：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ${extensionPath}/scripts/on-start.js"
          }
        ]
      }
    ]
  }
}
```

- **`SessionStart`**：Gemini CLI 会话启动时触发的生命周期事件。
- **`type: "command"`**：处理器类型为外部命令。
- **`${extensionPath}`**：内置变量，运行时自动替换为扩展的实际安装路径。

### `scripts/on-start.js`

会话启动时执行的 Node.js 脚本，功能为输出一条欢迎消息：

```javascript
console.log(
  'Session Started! This is running from a script in the hooks-example extension.',
);
```

## 依赖关系

- 依赖 Gemini CLI 扩展系统的钩子机制（自动扫描 `hooks/hooks.json`）。
- 依赖 Node.js 运行时（用于执行 `on-start.js` 脚本）。
- 依赖 `${extensionPath}` 内置路径变量的解析能力。
- 无任何 npm 包依赖。
