# dark

## 概述

CLI 内置暗色主题集合目录，包含多种暗色代码高亮主题定义。每个主题文件导出一个 `Theme` 实例，配置了 highlight.js 各语法元素（关键字、字符串、注释、类型等）的颜色映射，用于终端中代码块的语法着色渲染。

## 目录结构

```
dark/
├── ansi-dark.ts               # ANSI 暗色主题
├── atom-one-dark.ts           # Atom One Dark 风格主题
├── ayu-dark.ts                # Ayu Dark 风格主题
├── default-dark.ts            # 默认暗色主题
├── dracula-dark.ts            # Dracula 风格暗色主题
├── github-dark.ts             # GitHub Dark 风格主题
├── holiday-dark.ts            # 节日暗色主题
├── shades-of-purple-dark.ts   # Shades of Purple 风格暗色主题
└── solarized-dark.ts          # Solarized Dark 风格主题
```

## 核心组件

- **default-dark.ts**：默认暗色主题，通过 `new Theme('Default', 'dark', {...}, darkTheme)` 创建。使用 `darkTheme` 预定义调色板（如 `AccentBlue`、`AccentCyan`、`AccentGreen`、`AccentYellow`、`AccentRed`、`AccentPurple` 等）为 highlight.js 的各类语法 token 分配颜色：关键字/符号为蓝色，内置类型为青色，数字为绿色，字符串为黄色，正则为红色，注释为斜体灰色，变量为紫色，属性为浅蓝色。同时配置了 diff 增删行的背景色。
