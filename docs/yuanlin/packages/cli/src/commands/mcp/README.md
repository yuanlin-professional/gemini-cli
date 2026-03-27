# mcp 目录

## 概述

`mcp/` 目录实现了 Gemini CLI 的 **MCP（Model Context Protocol）服务器管理系统**，提供 MCP 服务器的添加、删除、列表、启用和禁用功能。支持三种传输协议（stdio、SSE、HTTP），具备完善的安全控制（信任检查、管理员白名单、工具过滤）和连接状态检测能力。

## 目录结构

```
mcp/
├── add.ts              # 添加 MCP 服务器（支持 stdio/SSE/HTTP 传输）
├── add.test.ts         # add 测试
├── remove.ts           # 移除 MCP 服务器
├── remove.test.ts      # remove 测试
├── list.ts             # 列出所有 MCP 服务器及连接状态
├── list.test.ts        # list 测试
└── enableDisable.ts    # 启用/禁用 MCP 服务器（支持会话级别）
```

## 架构图

```mermaid
graph TD
    入口["mcp.ts<br/>顶级命令注册"] --> |"defer 延迟加载"| 子命令集合

    subgraph 子命令集合["子命令模块"]
        Add["add<br/>添加服务器"]
        Remove["remove<br/>移除服务器"]
        List["list<br/>列出服务器"]
        Enable["enable<br/>启用服务器"]
        Disable["disable<br/>禁用服务器"]
    end

    Add --> 设置系统["Settings 系统<br/>loadSettings / setValue"]
    Remove --> 设置系统
    List --> 设置系统
    Enable --> 启用管理器["McpServerEnablementManager"]
    Disable --> 启用管理器

    List --> 连接测试["testMCPConnection<br/>MCP 协议连接检测"]
    List --> 扩展管理器["ExtensionManager<br/>加载扩展提供的 MCP 服务器"]
    List --> 管理员白名单["applyAdminAllowlist<br/>管理员安全过滤"]

    连接测试 --> MCP客户端["MCP Client<br/>@modelcontextprotocol/sdk"]
    连接测试 --> 传输层["createTransport<br/>stdio / SSE / HTTP"]

    subgraph 传输协议["支持的传输协议"]
        STDIO["stdio<br/>本地进程通信"]
        SSE["SSE<br/>Server-Sent Events"]
        HTTP["HTTP<br/>Streamable HTTP"]
    end

    Add --> 传输协议
```

## 核心组件

### 1. add.ts - 添加 MCP 服务器

支持三种传输协议，根据 `--transport` 参数构建不同的服务器配置：

| 传输类型 | 必需参数 | 可选参数 |
|---------|---------|---------|
| `stdio`（默认） | `command`、`args` | `env`、`timeout`、`trust` |
| `sse` | `url` | `headers`、`timeout`、`trust` |
| `http` | `url` | `headers`、`timeout`、`trust` |

支持两种配置作用域：
- `project`（默认）：写入工作区 `.gemini/settings.json`
- `user`：写入用户级全局配置

其他关键选项：
- `--trust`：信任服务器，跳过工具调用确认
- `--include-tools` / `--exclude-tools`：工具白名单/黑名单过滤
- `--description`：服务器描述

### 2. list.ts - 列出 MCP 服务器

最复杂的子命令，核心功能：

**服务器发现**：合并两个来源的 MCP 服务器
1. `settings.mcpServers` — 用户直接配置的服务器
2. 已安装扩展提供的 MCP 服务器

**管理员安全过滤**：通过 `applyAdminAllowlist()` 过滤被管理员阻止的服务器。

**连接状态检测**：对每个服务器进行实际 MCP 协议连接测试，状态包括：

| 状态 | 图标 | 说明 |
|------|------|------|
| `CONNECTED` | 绿色勾 | 连接正常 |
| `CONNECTING` | 黄色省略号 | 连接中 |
| `BLOCKED` | 红色禁止 | 被管理员/白名单阻止 |
| `DISABLED` | 灰色圆圈 | 已禁用 |
| `DISCONNECTED` | 红色叉 | 连接失败 |

**安全特性**：stdio 类型服务器在不受信任的工作区中不会执行连接测试。

### 3. enableDisable.ts - 启用/禁用

通过 `McpServerEnablementManager` 单例管理服务器启用状态，支持：
- **持久化启用/禁用**：写入配置文件
- **会话级禁用**（`--session`）：仅对当前会话生效，不写入配置

启用前会检查多层安全策略：管理员限制 > 白名单 > 黑名单 > 用户启用状态。

## 依赖关系

```mermaid
graph LR
    mcp["mcp/ 子命令"] --> yargs["yargs (CommandModule)"]
    mcp --> core["@google/gemini-cli-core<br/>MCPServerConfig, MCPServerStatus,<br/>createTransport, applyAdminAllowlist"]
    mcp --> sdk["@modelcontextprotocol/sdk<br/>Client (MCP 协议客户端)"]
    mcp --> settings["config/settings.js<br/>loadSettings, SettingScope"]
    mcp --> mcpMgr["config/mcp/<br/>McpServerEnablementManager,<br/>canLoadServer"]
    mcp --> extMgr["config/extension-manager.js<br/>ExtensionManager"]
    mcp --> chalk["chalk (终端着色)"]
    mcp --> utils["../utils.js (exitCli)"]
```

## 数据流

```mermaid
sequenceDiagram
    participant 用户
    participant Add命令
    participant List命令
    participant 设置系统
    participant MCP客户端
    participant MCP服务器

    Note over 用户,MCP服务器: 添加服务器流程
    用户->>Add命令: gemini mcp add myserver npx server-pkg
    Add命令->>设置系统: loadSettings()
    Add命令->>Add命令: 根据 transport 构建 MCPServerConfig
    Add命令->>设置系统: setValue(scope, 'mcpServers', config)
    Add命令->>用户: 服务器已添加

    Note over 用户,MCP服务器: 列表与状态检测流程
    用户->>List命令: gemini mcp list
    List命令->>设置系统: 加载配置 + 扩展中的 MCP 服务器
    List命令->>List命令: applyAdminAllowlist() 安全过滤

    loop 逐个检测服务器
        List命令->>List命令: canLoadServer() 检查启用状态
        List命令->>MCP客户端: createTransport() 创建传输层
        MCP客户端->>MCP服务器: connect() + ping()
        MCP服务器-->>MCP客户端: 响应
        MCP客户端-->>List命令: 连接状态
    end

    List命令->>用户: 显示服务器列表及状态
```
