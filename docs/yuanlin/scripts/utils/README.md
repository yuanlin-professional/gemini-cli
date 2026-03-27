# scripts/utils/

## 概述

该目录包含项目构建和代码生成过程中使用的工具函数。当前仅有一个文件 `autogen.ts`，提供用于自动生成代码和文档的通用工具函数，包括 Prettier 格式化、字符串处理和标记注入等功能。这些工具函数被项目的各类代码生成脚本所共享使用。

## 目录结构

```
scripts/utils/
└── autogen.ts    # 自动代码生成工具函数集合
```

## 核心组件

### autogen.ts — 自动生成工具函数

提供以下 5 个工具函数：

- **formatWithPrettier(content, filePath)**: 使用 Prettier 格式化生成的代码内容。自动解析目标文件路径对应的 Prettier 配置，确保生成的代码符合项目代码风格。
- **normalizeForCompare(content)**: 将内容标准化用于对比——统一换行符为 `\n` 并去除尾部空白。用于判断生成的内容是否与现有文件一致，避免不必要的文件覆写。
- **escapeBackticks(value)**: 转义字符串中的反斜杠和反引号，使其可安全嵌入 JavaScript/TypeScript 模板字面量（template literal）中。
- **formatDefaultValue(value, options?)**: 将任意类型的默认值格式化为人类可读的字符串表示。支持 `undefined`、`null`、字符串（可选是否加引号）、数字、布尔值、数组和对象。用于在生成的文档或代码中展示配置项的默认值。
- **injectBetweenMarkers(options)**: 在文档中定位两个标记（marker）之间的区域，将其内容替换为新生成的内容。用于保持文档的非生成部分不变，仅更新自动生成的区域（如 `<!-- BEGIN AUTO-GENERATED -->` 和 `<!-- END AUTO-GENERATED -->` 之间的内容）。

## 依赖关系

- **prettier**: 代码格式化引擎，用于确保生成的代码符合项目配置的代码风格规范
