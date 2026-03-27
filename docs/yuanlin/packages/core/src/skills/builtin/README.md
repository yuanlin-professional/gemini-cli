# packages/core/src/skills/builtin

## 概述

`skills/builtin` 目录包含 Gemini CLI 的**内置技能（Built-in Skills）**。目前仅包含一个内置技能 `skill-creator`，它是一个"元技能"——用于指导 Gemini CLI 创建新技能的技能。技能（Skill）是 Gemini CLI 的模块化扩展机制，通过 `SKILL.md` 文件和可选的捆绑资源（脚本、参考文档、资产），将 Gemini CLI 从通用 Agent 转变为特定领域的专业 Agent。

## 目录结构

```
builtin/
└── skill-creator/
    ├── SKILL.md                          # 技能定义文件（YAML 前置元数据 + Markdown 指南）
    └── scripts/
        ├── init_skill.cjs                # 技能初始化脚本
        ├── package_skill.cjs             # 技能打包脚本
        └── validate_skill.cjs            # 技能校验脚本
```

## 核心组件

### 1. SKILL.md — 技能创建指南

这是 `skill-creator` 技能的核心定义文件，包含：

- **YAML 前置元数据**：`name` 和 `description` 字段，用于技能触发匹配。当用户请求创建或更新技能时自动激活。
- **技能体系说明**：定义了技能的组成结构——必需的 `SKILL.md` + 可选的 `scripts/`（可执行脚本）、`references/`（参考文档）、`assets/`（输出资产）。
- **核心设计原则**：
  - **简洁至上**：上下文窗口是公共资源，只添加模型不具备的知识。
  - **自由度分级**：根据任务脆弱性选择高自由度（文本指导）、中自由度（伪代码）或低自由度（具体脚本）。
  - **渐进式披露**：三级加载——元数据（始终在上下文中，约100词）-> SKILL.md 正文（触发后加载，<5k词）-> 捆绑资源（按需加载）。
- **7 步创建流程**：理解需求 -> 规划资源 -> 初始化 -> 编辑实现 -> 打包 -> 安装 -> 迭代。

### 2. scripts/init_skill.cjs — 技能初始化脚本

命令行工具，用于创建新技能的模板目录结构：

```bash
node init_skill.cjs <skill-name> --path <output-directory>
```

功能：
- 创建技能目录及子目录（`scripts/`、`references/`、`assets/`）。
- 生成带 YAML 前置元数据和 TODO 占位符的 `SKILL.md` 模板。
- 创建示例文件（`example_script.cjs`、`example_reference.md`、`example_asset.txt`）。
- 内置路径遍历防护，防止恶意技能名称。

### 3. scripts/package_skill.cjs — 技能打包脚本

将开发完成的技能目录打包为 `.skill` 分发文件（本质上是 `.zip` 文件）。打包前自动执行校验，检查 YAML 格式、必需字段、命名规范、描述质量等。

### 4. scripts/validate_skill.cjs — 技能校验脚本

独立的校验工具，检查技能是否满足所有规范要求，可在打包前单独运行。

## 依赖关系

- **被加载方**：内置技能由 `../skillLoader.ts` 和 `../skillManager.ts` 加载和管理。
- **运行时依赖**：脚本使用 Node.js 内置模块（`fs`、`path`），无外部依赖。
- **与技能系统的关系**：`skill-creator` 通过 `activate_skill` 工具被触发，触发后 SKILL.md 的正文内容加载到 LLM 上下文中，指导 Agent 完成技能创建流程。
- **安装目标**：创建的技能通过 `gemini skills install` 命令安装到工作区（workspace）或用户（user）级别。
