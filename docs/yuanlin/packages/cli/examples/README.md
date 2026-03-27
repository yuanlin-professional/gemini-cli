# examples

## 概述

CLI 包的示例演示目录，包含基于 Ink/React 的交互式 UI 组件演示程序。这些示例展示了如何使用 Gemini CLI 的核心 UI 组件（如用户对话框、可滚动列表等），可作为开发者参考和组件调试工具。

## 目录结构

```
examples/
├── ask-user-dialog-demo.tsx   # AskUserDialog 组件演示，展示多种问题类型（单选、多选、文本输入、是/否）
└── scrollable-list-demo.tsx   # 可滚动列表组件演示
```

## 核心组件

- **ask-user-dialog-demo.tsx**：演示 `AskUserDialog` 组件的完整用法。定义了四种问题类型的示例数据（项目类型单选、功能多选、项目名称文本输入、Git 初始化是/否确认），通过 `KeypressProvider` 上下文和 Ink 的 `render` 方法将对话框渲染到终端，并展示提交结果和取消操作的处理逻辑。
- **scrollable-list-demo.tsx**：可滚动列表的独立演示程序。
