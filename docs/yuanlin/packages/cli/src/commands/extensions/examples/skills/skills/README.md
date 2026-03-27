# skills

## 概述

`skills` 是技能定义的根目录，位于 skills-example 扩展内部。每个子目录代表一个独立的技能，目录名即为技能名称。Gemini CLI 扫描此目录，自动注册所有包含 `SKILL.md` 文件的子目录作为可用技能。

## 目录结构

```
skills/
└── greeter/        # greeter 技能
    └── SKILL.md    # 技能定义文件
```

## 核心组件

- **greeter/** - 友好问候技能目录。包含 `SKILL.md` 定义文件，该技能在用户发出问候或请求打招呼时激活，以预定义的友好方式回应。这是技能系统的最小化示例，展示了如何通过 Markdown 前置元数据和 prompt 内容定义一个完整技能。
