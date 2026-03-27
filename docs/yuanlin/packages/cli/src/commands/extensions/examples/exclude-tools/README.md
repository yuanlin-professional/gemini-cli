# exclude-tools

## 概述

`exclude-tools` 是一个 Gemini CLI 扩展示例，演示如何通过扩展配置排除（禁用）特定的工具调用。该扩展通过在 `gemini-extension.json` 中声明 `excludeTools` 字段，阻止模型调用指定的危险命令，从而增强安全性。

## 目录结构

```
exclude-tools/
└── gemini-extension.json    # 扩展配置文件，声明排除的工具
```

## 核心组件

- **gemini-extension.json** - 扩展清单文件。定义扩展名称为 `excludeTools`，版本 `1.0.0`。核心配置项 `excludeTools` 是一个字符串数组，当前排除了 `run_shell_command(rm -rf)`，即禁止模型通过 shell 命令工具执行 `rm -rf` 这一危险操作。这种机制允许扩展作者在不修改核心代码的前提下，限制模型可用的工具范围。
