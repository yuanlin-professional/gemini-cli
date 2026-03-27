# skills

## 概述

`skills` 是一个 Gemini CLI 扩展示例，演示如何通过扩展定义自定义技能（skills）。技能是一种可被用户通过斜杠命令（如 `/greeter`）调用的预定义行为模块，每个技能通过 Markdown 文件（`SKILL.md`）定义其名称、描述和 prompt 模板，使模型按照特定的角色或逻辑来响应。

## 目录结构

```
skills/
├── gemini-extension.json    # 扩展清单文件
└── skills/                  # 技能定义目录
    └── greeter/             # greeter 技能子目录
        └── SKILL.md         # greeter 技能的定义文件
```

## 核心组件

- **gemini-extension.json** - 扩展清单文件，声明扩展名称为 `skills-example`，版本 `1.0.0`。该文件作为扩展的入口标识，Gemini CLI 通过它识别和加载扩展。
- **skills/** - 技能定义子目录，包含所有技能的定义。每个技能占据一个独立子目录，目录名即为技能的调用名称。
