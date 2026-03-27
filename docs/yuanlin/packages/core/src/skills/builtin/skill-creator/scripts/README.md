# scripts

## 概述

skill-creator 技能的工具脚本目录，包含技能开发生命周期中使用的三个 CommonJS 脚本：初始化模板、验证结构和打包分发。这些脚本由 Gemini CLI 代理在技能创建流程中调用执行。

## 目录结构

```
scripts/
├── init_skill.cjs       # 技能初始化脚本，从模板创建新技能
├── package_skill.cjs    # 技能打包脚本，生成 .skill 分发文件
└── validate_skill.cjs   # 技能验证脚本，检查结构和内容合规性
```

## 核心组件

- **init_skill.cjs**：技能初始化脚本。用法为 `node init_skill.cjs <skill-name> --path <output-directory>`。执行时会在指定路径下创建以技能名命名的目录，自动生成：带 YAML 前言和 TODO 占位符的 `SKILL.md` 模板文件、`scripts/`（含示例脚本 `example_script.cjs`）、`references/`（含示例参考文档）、`assets/`（含占位资源文件）三个资源子目录。脚本内置路径遍历防护（禁止名称含路径分隔符）和目录存在性检查。模板中使用 `{skill_name}` 和 `{skill_title}` 占位符，由 `titleCase` 函数将连字符名称转换为首字母大写标题。
- **package_skill.cjs**：打包脚本，将技能目录压缩为 `.skill` 文件（实质为 zip 格式），打包前自动运行验证。
- **validate_skill.cjs**：验证脚本，检查 YAML 前言格式、必填字段、命名规范、目录结构、描述完整性等。
