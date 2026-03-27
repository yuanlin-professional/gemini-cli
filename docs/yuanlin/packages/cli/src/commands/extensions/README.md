# extensions 目录

## 概述

`extensions/` 目录实现了 Gemini CLI 的**扩展管理系统**，提供扩展的完整生命周期管理能力，包括安装、卸载、更新、启用/禁用、链接、创建、验证和配置。所有子命令均通过 yargs `CommandModule` 规范导出，由上层 `extensions.tsx` 统一注册。

## 目录结构

```
extensions/
├── install.ts          # 安装扩展（支持 Git URL 和本地路径）
├── install.test.ts     # install 测试
├── uninstall.ts        # 卸载扩展
├── uninstall.test.ts   # uninstall 测试
├── list.ts             # 列出已安装扩展（支持 text/json 格式）
├── list.test.ts        # list 测试
├── update.ts           # 更新扩展
├── update.test.ts      # update 测试
├── enable.ts           # 启用扩展
├── enable.test.ts      # enable 测试
├── disable.ts          # 禁用扩展
├── disable.test.ts     # disable 测试
├── link.ts             # 链接本地开发扩展
├── link.test.ts        # link 测试
├── new.ts              # 创建新扩展脚手架
├── new.test.ts         # new 测试
├── validate.ts         # 验证扩展配置
├── validate.test.ts    # validate 测试
├── configure.ts        # 配置扩展设置项
├── configure.test.ts   # configure 测试
└── utils.ts            # 共享工具函数
```

## 架构图

```mermaid
graph TD
    入口["extensions.tsx<br/>顶级命令注册"] --> |"defer 延迟加载"| 子命令集合

    subgraph 子命令集合["子命令模块"]
        Install["install<br/>安装扩展"]
        Uninstall["uninstall<br/>卸载扩展"]
        List["list<br/>列出扩展"]
        Update["update<br/>更新扩展"]
        Enable["enable<br/>启用扩展"]
        Disable["disable<br/>禁用扩展"]
        Link["link<br/>链接本地扩展"]
        New["new<br/>创建新扩展"]
        Validate["validate<br/>验证扩展"]
        Configure["configure<br/>配置设置"]
    end

    子命令集合 --> ExtManager["ExtensionManager<br/>扩展管理器核心"]
    子命令集合 --> Utils["utils.ts<br/>共享工具函数"]

    ExtManager --> 配置层["配置层<br/>settings / trustedFolders / consent"]
    Install --> 信任检查["FolderTrustDiscoveryService<br/>文件夹信任检查"]
```

## 核心组件

### 1. install.ts - 安装扩展

最复杂的子命令，支持两种安装来源：
- **Git 仓库 URL**：从远程克隆
- **本地路径**：本地目录安装

安装流程包含**文件夹信任检查**：对本地路径会通过 `FolderTrustDiscoveryService` 扫描扩展内容（Commands、MCP servers、Hooks、Skills 等），提示用户确认信任后才允许安装。

```typescript
// 关键参数
interface InstallArgs {
  source: string;         // Git URL 或本地路径
  ref?: string;           // Git ref（分支/标签）
  autoUpdate?: boolean;   // 自动更新
  allowPreRelease?: boolean; // 允许预发布版本
  consent?: boolean;      // 跳过确认提示
  skipSettings?: boolean; // 跳过配置过程
}
```

### 2. list.ts - 列出扩展

支持两种输出格式：
- `text`（默认）：人类可读的文本格式
- `json`：结构化 JSON 输出，便于脚本处理

### 3. utils.ts - 共享工具函数

提供扩展子命令间的复用逻辑：

| 函数 | 功能 |
|------|------|
| `getExtensionManager()` | 创建并初始化 ExtensionManager 实例 |
| `getExtensionAndManager()` | 按名称查找已安装扩展 |
| `configureSpecificSetting()` | 配置单个扩展设置项 |
| `configureExtension()` | 配置扩展的全部设置 |
| `configureAllExtensions()` | 批量配置所有扩展 |
| `configureExtensionSettings()` | 带作用域的设置配置流程 |
| `getFormattedSettingValue()` | 格式化设置值（敏感值脱敏） |

设置支持**两种作用域**：`USER`（用户级）和 `WORKSPACE`（工作区级），工作区设置优先于用户设置。

## 依赖关系

```mermaid
graph LR
    ext["extensions/ 子命令"] --> yargs["yargs (CommandModule)"]
    ext --> core["@google/gemini-cli-core<br/>debugLogger, FolderTrustDiscoveryService"]
    ext --> extMgr["config/extension-manager.js<br/>ExtensionManager"]
    ext --> settings["config/settings.js<br/>loadSettings"]
    ext --> trust["config/trustedFolders.js<br/>isWorkspaceTrusted, loadTrustedFolders"]
    ext --> consent["config/extensions/consent.js<br/>requestConsentNonInteractive"]
    ext --> extSettings["config/extensions/extensionSettings.js<br/>promptForSetting, updateSetting"]
    ext --> chalk["chalk (终端着色)"]
    ext --> prompts["prompts (交互式输入)"]
    ext --> utils_cmd["../utils.js (exitCli)"]
```

## 数据流

```mermaid
sequenceDiagram
    participant 用户
    participant Install命令
    participant 信任服务
    participant 扩展管理器
    participant 文件系统

    用户->>Install命令: gemini extensions install <source>
    Install命令->>Install命令: inferInstallMetadata() 推断安装类型

    alt 本地路径安装
        Install命令->>信任服务: FolderTrustDiscoveryService.discover()
        信任服务-->>Install命令: 扫描结果（Commands/MCP/Hooks/Skills）
        Install命令->>用户: 显示扫描结果，请求信任确认
        用户-->>Install命令: 确认信任
        Install命令->>文件系统: 保存信任设置
    end

    Install命令->>用户: 请求安装许可
    用户-->>Install命令: 确认
    Install命令->>扩展管理器: installOrUpdateExtension()
    扩展管理器->>文件系统: 写入扩展配置
    扩展管理器-->>Install命令: 安装成功
    Install命令->>用户: 输出成功信息
```
