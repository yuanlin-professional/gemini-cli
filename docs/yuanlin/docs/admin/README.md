# admin

## 概述

该目录存放面向企业管理员的文档，定义了 Gemini CLI 的企业级管控能力。管理员可通过 Management Console 集中管理安全策略和配置，这些控制在全局强制执行，用户无法在本地覆盖。

## 目录结构

```
admin/
└── enterprise-controls.md    # 企业管理员控制项文档
```

## 核心组件

- **enterprise-controls.md**：企业管控完整文档，涵盖以下控制项：
  - **Strict Mode**（默认启用）：启用后用户无法进入 yolo 模式
  - **Extensions**（默认禁用）：控制用户是否可以使用或安装扩展
  - **MCP 开关**（默认禁用）：控制用户是否可以使用 MCP 服务器
  - **MCP 服务器白名单**（preview）：管理员定义允许连接的 MCP 服务器列表，支持 `url`、`type`、`trust`、`includeTools`、`excludeTools` 字段，实施严格的合并逻辑（管理员配置覆盖本地 url/type/trust，自动清除本地执行字段）
  - **Required MCP Servers**（preview）：强制注入的 MCP 服务器，无论用户本地配置如何都会加载，支持 OAuth 认证与服务账户模拟
  - **Unmanaged Capabilities**（默认禁用）：控制 Agent Skills 等高级功能的可用性

  文档还详细说明了管理控制与系统设置（System Settings）的区别：管理控制不可被本地修改，而系统设置可被有权限的用户调整。
