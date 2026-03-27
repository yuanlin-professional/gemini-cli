# skill-creator

## 概述

Gemini CLI 的内置技能（Skill），用于指导用户创建新的技能包。当用户希望创建或更新扩展 Gemini CLI 能力的技能时，该技能被激活，提供完整的技能开发生命周期指导：从理解需求、规划资源、初始化模板、编辑内容、打包分发到安装使用。

## 目录结构

```
skill-creator/
├── SKILL.md    # 技能定义文件，包含 YAML 元数据和 Markdown 指导说明
└── scripts/    # 技能创建相关的工具脚本
    ├── init_skill.cjs       # 初始化新技能模板
    ├── package_skill.cjs    # 打包技能为 .skill 分发文件
    └── validate_skill.cjs   # 验证技能结构和内容
```

## 核心组件

- **SKILL.md**：技能的核心定义文件。YAML 前言定义了名称（`skill-creator`）和描述（指导创建有效技能）。正文详细阐述了技能的设计原则和创建流程：
  - **设计原则**：简洁优先（节省上下文窗口）、按任务脆弱性设置自由度（高/中/低）、渐进式披露（元数据 -> SKILL.md 正文 -> 捆绑资源三级加载）。
  - **技能结构**：必需的 `SKILL.md` + 可选的 `scripts/`（可执行脚本）、`references/`（参考文档）、`assets/`（输出资源）。
  - **创建流程七步法**：(1) 通过具体示例理解技能用途；(2) 规划可复用资源；(3) 运行 `init_skill.cjs` 初始化；(4) 编辑实现；(5) 运行 `package_skill.cjs` 打包；(6) 安装并重载；(7) 基于实际使用迭代改进。
