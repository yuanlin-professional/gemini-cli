# themes-example

## 概述

`themes-example` 是一个 Gemini CLI 扩展示例，演示如何通过扩展添加自定义 UI 主题。该扩展在 `gemini-extension.json` 中定义了一套名为 `shades-of-green`（绿色调）的完整主题配色方案，用户可通过设置文件或 `/theme` 命令切换到此主题，改变 CLI 界面的视觉风格。

## 目录结构

```
themes-example/
├── gemini-extension.json    # 扩展清单文件（含主题定义）
└── README.md                # 扩展使用说明
```

## 核心组件

- **gemini-extension.json** - 扩展清单兼主题定义文件。声明扩展名称为 `themes-example`，版本 `1.0.0`。核心配置为 `themes` 数组，定义了 `shades-of-green` 主题，包含以下配色分类：
  - `background.primary` (`#1a362a`) - 深绿色背景
  - `text.primary` (`#a6e3a1`) - 浅绿色主文字
  - `text.secondary` (`#6e8e7a`) - 灰绿色次要文字
  - `text.link` (`#89e689`) - 亮绿色链接
  - `status.success/warning/error` - 状态指示色
  - `border.default` (`#4a6c5a`) - 边框色
  - `ui.comment` (`#6e8e7a`) - 注释色

- **README.md** - 使用指南，说明了通过 `gemini extensions link` 链接扩展、在 `settings.json` 中设置主题名称、以及通过 `/theme` 命令交互式切换主题的三步使用流程。
