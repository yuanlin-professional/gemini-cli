# ui/themes/builtin/

## 概述

该目录包含 Gemini CLI 的全部内置终端主题定义。主题系统为代码高亮（基于 highlight.js 的 CSS 类名映射）和 UI 配色提供统一的样式方案。内置主题分为深色（dark）、浅色（light）和无色（no-color）三大类别，每个主题通过 `Theme` 类实例化，包含 hljs 语法高亮样式、颜色调色板（`ColorsTheme`）和语义化颜色标记（`SemanticColors`）。

## 目录结构

```
builtin/
├── no-color.ts                   # 无色主题，所有颜色值为空字符串，适用于无色终端或辅助功能场景
├── dark/                         # 深色主题集合
│   ├── default-dark.ts           # 默认深色主题
│   ├── ansi-dark.ts              # ANSI 深色主题
│   ├── atom-one-dark.ts          # Atom One Dark 风格
│   ├── ayu-dark.ts               # Ayu Dark 风格
│   ├── dracula-dark.ts           # Dracula 风格
│   ├── github-dark.ts            # GitHub Dark 风格
│   ├── holiday-dark.ts           # 节日主题深色版
│   ├── shades-of-purple-dark.ts  # Shades of Purple 风格
│   └── solarized-dark.ts         # Solarized Dark 风格
└── light/                        # 浅色主题集合
    ├── default-light.ts          # 默认浅色主题
    ├── ansi-light.ts             # ANSI 浅色主题
    ├── ayu-light.ts              # Ayu Light 风格
    ├── github-light.ts           # GitHub Light 风格
    ├── googlecode-light.ts       # Google Code 风格
    ├── solarized-light.ts        # Solarized Light 风格
    └── xcode-light.ts            # Xcode 风格
```

## 核心组件

### Theme 类结构
每个主题文件导出一个 `Theme` 实例，构造参数包含：
1. **名称**（如 `'Default'`、`'Dracula'`）
2. **模式**（`'dark'` 或 `'light'`）
3. **hljs 样式映射**：为 `hljs-keyword`、`hljs-string`、`hljs-comment` 等语法高亮类名定义 `color`、`fontStyle`、`fontWeight`、`backgroundColor` 等 CSS 属性
4. **ColorsTheme 调色板**：定义 `Background`、`Foreground`、`AccentBlue`、`AccentGreen`、`AccentYellow`、`AccentRed`、`DiffAdded`、`DiffRemoved` 等基础颜色值
5. **SemanticColors 语义标记**：定义 `text.primary`、`background.message`、`status.error`、`ui.gradient` 等高层语义化颜色

### no-color.ts — 无色主题
- 所有 `ColorsTheme` 和 `SemanticColors` 的颜色值均为空字符串
- 仅保留 `fontStyle` 和 `fontWeight` 等非颜色样式属性
- 适用于 `NO_COLOR` 环境变量启用或无色终端环境

### dark/default-dark.ts — 默认深色主题（示例）
- 引用 `darkTheme` 预定义调色板，将语法高亮类映射到对应颜色：关键字蓝色、字符串黄色、注释灰色斜体、变量紫色等
- diff 新增行背景色为深绿色（`#144212`），删除行为深红色（`#600`）

## 依赖关系

- **../theme.js**: 提供 `Theme` 类和 `ColorsTheme` 类型定义，以及预设调色板（`darkTheme`、`lightTheme`）
- **../semantic-tokens.js**: 提供 `SemanticColors` 类型定义，定义语义化颜色接口（text、background、border、ui、status）
