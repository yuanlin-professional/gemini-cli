# custom-commands -- 自定义命令扩展示例

## 概述

本目录是一个 Gemini CLI 扩展示例，演示如何通过 TOML 配置文件定义自定义斜杠命令。该扩展名为 `custom-commands`（版本 1.0.0），展示了命令模板系统的核心能力：使用参数占位符 `{{args}}`、嵌入 shell 命令 `!{...}` 以及自定义提示词（prompt）。通过这种方式，用户可以将常用的代码搜索、分析等工作流封装为简洁的斜杠命令。

## 目录结构

```
custom-commands/
├── .gitignore                # Git 忽略规则（node_modules、dist、.env 等）
├── gemini-extension.json     # 扩展清单文件，声明扩展名称和版本
└── commands/                 # 自定义命令目录
    └── fs/                   # "fs" 命令命名空间
        └── grep-code.toml   # "/fs grep-code" 命令定义
```

## 核心组件

### `gemini-extension.json`

扩展清单文件，结构极简：

```json
{
  "name": "custom-commands",
  "version": "1.0.0"
}
```

Gemini CLI 通过扫描 `commands/` 目录自动发现命令定义。

### `commands/fs/grep-code.toml`

定义了一个名为 `/fs grep-code` 的自定义命令。核心内容：

```toml
prompt = """
Please summarize the findings for the pattern `{{args}}`.

Search Results:
!{grep -r {{args}} .}
"""
```

- **`{{args}}`**：参数模板占位符，运行时被用户输入的参数替换。
- **`!{grep -r {{args}} .}`**：嵌入式 shell 命令，执行后其输出会内联到提示词中。
- 整体流程：先执行 `grep` 搜索，将搜索结果嵌入提示词，再由 Gemini 模型对结果进行摘要分析。

### 命令命名规则

目录层级映射为命令路径：`commands/fs/grep-code.toml` 对应斜杠命令 `/fs grep-code`。

## 依赖关系

- 依赖 Gemini CLI 扩展系统的命令发现机制（自动扫描 `commands/` 目录下的 TOML 文件）。
- 依赖系统 `grep` 命令（由 shell 命令嵌入功能 `!{...}` 调用）。
- 无任何 npm 包依赖。
