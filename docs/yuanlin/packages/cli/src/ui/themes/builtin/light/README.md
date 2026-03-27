# light

## 概述

CLI 内置亮色主题集合目录，包含多种亮色代码高亮主题定义。每个主题文件导出一个 `Theme` 实例，配置了 highlight.js 各语法元素的颜色映射，适用于浅色背景终端环境下的代码语法着色。

## 目录结构

```
light/
├── ansi-light.ts          # ANSI 亮色主题
├── ayu-light.ts           # Ayu Light 风格主题
├── default-light.ts       # 默认亮色主题
├── github-light.ts        # GitHub Light 风格主题
├── googlecode-light.ts    # Google Code 风格亮色主题
├── solarized-light.ts     # Solarized Light 风格主题
└── xcode-light.ts         # Xcode 风格亮色主题
```

## 核心组件

- **default-light.ts**：默认亮色主题，通过 `new Theme('Default Light', 'light', {...}, lightTheme)` 创建。使用 `lightTheme` 预定义调色板为语法 token 分配颜色：关键字/标签/内置类型为蓝色，字符串/标题/属性/模板变量为红色，选择器/符号/链接为青色，增加行为绿色，删除行为红色，注释为灰色调。整体配色方案适配浅色终端背景，确保在高对比度下的可读性。
