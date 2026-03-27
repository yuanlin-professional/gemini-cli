# skills 目录

## 概述

`skills/` 目录实现了 Gemini CLI 的 **Agent 技能（Skills）管理系统**，提供技能的安装、卸载、链接、列表和启用/禁用功能。Skills 是可复用的 Agent 能力模块，可以从 Git 仓库或本地路径安装，支持 `user`（全局）和 `workspace`（工作区）两种作用域。

## 目录结构

```
skills/
├── install.ts          # 安装技能（Git URL / 本地路径）
├── install.test.ts     # install 测试
├── uninstall.ts        # 卸载技能
├── uninstall.test.ts   # uninstall 测试
├── link.ts             # 链接本地开发技能
├── link.test.ts        # link 测试
├── list.ts             # 列出已发现的技能
├── list.test.ts        # list 测试
├── enable.ts           # 启用技能
├── enable.test.ts      # enable 测试
├── disable.ts          # 禁用技能
└── disable.test.ts     # disable 测试
```

## 架构图

```mermaid
graph TD
    入口["skills.tsx<br/>顶级命令注册"] --> |"defer 延迟加载"| 子命令集合

    subgraph 子命令集合["子命令模块"]
        Install["install<br/>安装技能"]
        Uninstall["uninstall<br/>卸载技能"]
        Link["link<br/>链接本地技能"]
        List["list<br/>列出技能"]
        Enable["enable<br/>启用技能"]
        Disable["disable<br/>禁用技能"]
    end

    Install --> skillUtils["utils/skillUtils.js<br/>installSkill"]
    Uninstall --> skillUtils
    Link --> skillUtils

    Enable --> skillSettings["utils/skillSettings.js<br/>enableSkill / disableSkill"]
    Disable --> skillSettings

    List --> 配置加载["loadCliConfig<br/>加载完整 CLI 配置"]
    配置加载 --> 初始化["config.initialize()<br/>触发扩展加载与技能发现"]
    初始化 --> SkillManager["SkillManager<br/>技能管理器"]

    Install --> 同意流程["consent 安全确认<br/>requestConsentNonInteractive"]

    subgraph 安装作用域["安装作用域"]
        User["user（全局）"]
        Workspace["workspace（工作区）"]
    end

    Install --> 安装作用域
```

## 核心组件

### 1. install.ts - 安装技能

支持从两种来源安装技能：

| 来源 | 示例 |
|------|------|
| Git 仓库 URL | `gemini skills install https://github.com/user/skill-repo` |
| 本地路径 | `gemini skills install ./my-skill` |

关键参数：
```typescript
interface InstallArgs {
  source: string;              // Git URL 或本地路径
  scope?: 'user' | 'workspace'; // 安装作用域（默认 user）
  path?: string;               // Git 仓库内的子路径
  consent?: boolean;           // 跳过安全确认提示
}
```

安装前会触发安全确认流程，展示即将安装的技能定义（`SkillDefinition`），用户确认后才执行安装。

### 2. list.ts - 列出技能

通过完整的 CLI 配置初始化流程来发现所有技能：

1. `loadCliConfig()` 加载配置
2. `config.initialize()` 触发扩展加载与技能发现
3. `skillManager.getAllSkills()` 获取全部技能

输出信息包括：
- 技能名称
- 启用/禁用状态（彩色标记）
- 是否为内置技能
- 描述信息
- 安装位置

排序规则：非内置技能优先显示，同类按名称字母序排列。`--all` 参数可显示内置技能。

### 3. enable.ts / disable.ts - 启用/禁用

通过 `skillSettings.js` 工具模块修改技能的启用状态，并通过 `renderSkillActionFeedback()` 提供格式化的操作反馈。操作直接写入 settings 配置文件。

### 4. link.ts - 链接本地技能

类似 `npm link`，将本地开发目录的技能链接到 Gemini CLI 配置中，便于开发调试。

## 依赖关系

```mermaid
graph LR
    skills["skills/ 子命令"] --> yargs["yargs (CommandModule)"]
    skills --> core["@google/gemini-cli-core<br/>debugLogger, SkillDefinition"]
    skills --> skillUtils["utils/skillUtils.js<br/>installSkill, renderSkillActionFeedback"]
    skills --> skillSettings["utils/skillSettings.js<br/>enableSkill, disableSkill"]
    skills --> settings["config/settings.js<br/>loadSettings"]
    skills --> config["config/config.js<br/>loadCliConfig"]
    skills --> consent["config/extensions/consent.js<br/>requestConsentNonInteractive"]
    skills --> chalk["chalk (终端着色)"]
    skills --> utils["../utils.js (exitCli)"]
```

## 数据流

```mermaid
sequenceDiagram
    participant 用户
    participant Install命令
    participant 同意模块
    participant skillUtils
    participant 文件系统

    Note over 用户,文件系统: 安装技能流程
    用户->>Install命令: gemini skills install <source> --scope user
    Install命令->>skillUtils: installSkill(source, scope, subpath)
    skillUtils->>skillUtils: 解析来源（Git / 本地路径）

    alt 需要安全确认
        skillUtils->>同意模块: skillsConsentString() 构建提示
        同意模块->>用户: 显示即将安装的技能信息
        用户-->>同意模块: 确认安装
    end

    skillUtils->>文件系统: 克隆仓库 / 复制文件
    skillUtils->>文件系统: 写入技能配置
    skillUtils-->>Install命令: 已安装技能列表
    Install命令->>用户: 输出成功信息（名称、作用域、位置）

    Note over 用户,文件系统: 列出技能流程
    用户->>Install命令: gemini skills list --all
    Install命令->>Install命令: loadCliConfig() + initialize()
    Install命令->>Install命令: skillManager.getAllSkills()
    Install命令->>Install命令: 排序：非内置优先，字母序
    Install命令->>用户: 显示技能列表（名称/状态/描述/位置）
```
