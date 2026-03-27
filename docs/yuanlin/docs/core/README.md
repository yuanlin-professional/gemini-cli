# docs/core/ - Core 包文档

## 概述

`docs/core/` 目录描述 Gemini CLI 核心包（`packages/core`）的架构和功能。Core 包是 Gemini CLI 的后端部分，负责与 Gemini API 通信、管理工具、处理来自 CLI 前端（`packages/cli`）的请求。

## 目录结构

```
core/
├── index.md                    # Core 包概述和导航
├── subagents.md                # 子 Agent 系统（实验性功能）
├── local-model-routing.md      # 本地模型路由（实验性，使用本地 Gemma 模型）
└── remote-agents.md            # 远程 Agent 支持
```

## 架构图

```mermaid
graph TB
    subgraph Core 包职责
        API[Gemini API 交互<br/>安全通信与响应处理]
        Prompt[提示词工程<br/>构建有效提示]
        ToolMgmt[工具管理与编排<br/>注册/解释/执行/返回]
        Session[会话与状态管理<br/>对话历史和上下文]
        Config[配置管理<br/>API Key/模型/工具设置]
    end

    subgraph 高级功能
        SubAgents[subagents.md<br/>子 Agent 系统]
        LocalRouting[local-model-routing.md<br/>本地模型路由]
        RemoteAgents[remote-agents.md<br/>远程 Agent]
    end

    subgraph 相关参考
        ToolsRef[../reference/tools.md<br/>工具参考]
        PolicyRef[../reference/policy-engine.md<br/>策略引擎]
        MemportRef[../reference/memport.md<br/>记忆导入]
    end

    Core包职责 --> 高级功能
    API --> SubAgents
    API --> RemoteAgents
    ToolMgmt --> ToolsRef
    ToolMgmt --> PolicyRef
    Config --> LocalRouting
```

## 核心组件

| 文档 | 描述 |
|------|------|
| `index.md` | Core 包概述，描述其五大核心职责：API 交互、提示词工程、工具编排、会话管理、配置管理 |
| `subagents.md` | 子 Agent 系统：创建和使用专门化的子 Agent 处理复杂任务（实验性） |
| `local-model-routing.md` | 使用本地 Gemma 模型进行模型路由决策（实验性） |
| `remote-agents.md` | 远程 Agent 支持，允许与外部 Agent 通信 |

## 依赖关系

### 内部引用

- `index.md` 引用 `../reference/tools.md`（工具参考）
- `index.md` 引用 `../reference/policy-engine.md`（策略引擎）
- `index.md` 引用 `../reference/memport.md`（记忆导入）
- 被 `docs/index.md` 间接引用
