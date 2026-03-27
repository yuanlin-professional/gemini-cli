# tutorials

## 概述

该目录包含 Gemini CLI 的用户教程集合，涵盖从基础操作到高级功能的完整使用指南。教程以实际操作为导向，帮助用户掌握任务规划、会话管理、文件操作、Shell 命令、MCP 配置、自动化等核心能力。

## 目录结构

```
tutorials/
├── automation.md              # 自动化工作流教程
├── file-management.md         # 文件管理操作教程
├── mcp-setup.md               # MCP 服务器配置教程
├── memory-management.md       # 记忆管理教程（持久化偏好设置）
├── plan-mode-steering.md      # 计划模式引导教程
├── session-management.md      # 会话管理教程（保存与恢复）
├── shell-commands.md          # Shell 命令使用教程
├── skills-getting-started.md  # Agent Skills 入门教程
├── task-planning.md           # 任务规划教程（使用 todo 列表）
└── web-tools.md               # Web 工具使用教程
```

## 核心组件

- **task-planning.md**：任务规划教程，教用户如何利用内置的 todo 列表管理复杂任务：
  - **背景**：解释 LLM 上下文窗口有限，任务规划提供可见性、聚焦性和韧性
  - **创建计划**：通过明确提示（如"请先制定计划"）触发 `write_todos` 工具生成结构化列表
  - **审查与迭代**：支持动态增删步骤、调整顺序
  - **执行与监控**：实时显示当前进行中的任务，`Ctrl+T` 切换完整 todo 视图
  - **处理变更**：支持运行中取消或跳过步骤，计划作为"活文档"动态调整

- **session-management.md**：会话保存与恢复教程
- **memory-management.md**：持久化偏好记忆教程
- **mcp-setup.md**：MCP 服务器集成配置指南
- **automation.md**：自动化工作流搭建教程
- **file-management.md**：文件读写与管理操作指南
- **shell-commands.md**：Shell 命令执行教程
- **skills-getting-started.md**：Agent Skills 快速入门
- **plan-mode-steering.md**：计划模式的高级引导技巧
- **web-tools.md**：Web 工具（搜索、抓取）使用教程
