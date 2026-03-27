# greeter

## 概述

`greeter` 是 skills-example 扩展中定义的一个友好问候技能。当用户说 "hello" 或请求问候时，该技能会指导模型以特定的友好方式回应。技能通过 `SKILL.md` 文件定义，使用 YAML 前置元数据声明技能的名称和描述，正文部分则是发送给模型的 system prompt。

## 目录结构

```
greeter/
└── SKILL.md    # 技能定义文件
```

## 核心组件

- **SKILL.md** - greeter 技能的完整定义文件。YAML 前置元数据声明 `name: greeter` 和 `description: A friendly greeter skill`。正文内容是 prompt 模板，指示模型扮演友好问候者的角色，在用户发出问候时回复 "Greetings from the skills-example extension!"。该文件展示了 Gemini CLI 技能系统的标准定义格式：Markdown + YAML frontmatter。
