# packages/core/src/policy/policies

## 概述

`policy/policies` 目录存放了 Gemini CLI 策略引擎所使用的全部 **TOML 策略配置文件**。这些策略定义了不同审批模式（Approval Mode）下各个工具的执行权限，是 CLI 安全模型的核心。策略引擎根据优先级规则匹配来决定工具调用是被允许（allow）、需要用户确认（ask_user）还是被拒绝（deny）。

### 优先级体系

策略规则采用分层优先级机制，高优先级覆盖低优先级：

| 层级 | 优先级范围 | 说明 |
|------|-----------|------|
| Admin 策略 | 5.x | 管理员策略，最高优先级 |
| User 策略 | 4.x | 用户策略（含 UI 选择、CLI 参数、MCP 信任等） |
| Workspace 策略 | 3.x | 工作区级策略 |
| Extension 策略 | 2.x | 扩展级策略 |
| Default 策略 | 1.x | 默认策略（本目录中的 TOML 文件） |

## 目录结构

```
policies/
├── conseca.toml            # Conseca 模式策略
├── discovered.toml         # 已发现工具的策略
├── memory-manager.toml     # 内存管理器策略
├── non-interactive.toml    # 非交互模式策略
├── plan.toml               # 计划模式（Plan Mode）策略
├── read-only.toml          # 只读工具自动放行策略
├── sandbox-default.toml    # 沙箱默认模式配置
├── tracker.toml            # 追踪器策略
├── write.toml              # 写操作工具策略
└── yolo.toml               # YOLO 全自动模式策略
```

## 核心组件

### 1. yolo.toml — YOLO 全自动模式

YOLO 模式下几乎所有工具自动放行（priority=998），但有两个例外：
- **`ask_user`** 工具仍需用户交互（priority=999），确保模型在需要时仍能收集用户偏好。
- **`enter_plan_mode` / `exit_plan_mode`** 被拒绝（priority=999），因为计划模式需要人类审批，与 YOLO 的自主特性冲突。

### 2. write.toml — 写操作策略

定义写类工具（`replace`、`write_file`、`run_shell_command`、`save_memory`、`activate_skill`、`web_fetch`）的默认行为：
- 交互模式下默认 `ask_user`（priority=10），需要用户确认。
- `autoEdit` 模式下自动放行（priority=15），但写文件类操作附带 **safety_checker**（`allowed-path`）确保仅修改允许的路径。
- 非交互模式下直接 `deny`（priority=10），防止无人值守时执行危险操作。

### 3. read-only.toml — 只读工具放行

将只读工具（`glob`、`grep_search`、`list_directory`、`read_file`、`google_web_search`、`codebase_investigator` 等）设为自动放行（priority=50），无需用户确认。

### 4. plan.toml — 计划模式

计划模式的核心安全策略：
- 默认拒绝所有工具（catch-all deny，priority=60）。
- 显式允许只读工具（priority=70）。
- 允许对 `.gemini/tmp/*/plans/*.md` 路径的写入，用于保存计划文件。
- 管理 `enter_plan_mode` / `exit_plan_mode` 的状态转换逻辑。

### 5. non-interactive.toml — 非交互模式

极简策略：在非交互模式下禁止 `ask_user` 工具（priority=999），因为没有用户可以响应。

### 6. sandbox-default.toml — 沙箱默认配置

定义沙箱模式的多种子模式：
- **plan 模式**：只读、无网络、不允许覆盖。
- **default 模式**：只读、无网络、允许覆盖。
- **accepting_edits 模式**：可写，允许 `sed`、`grep`、`awk` 等编辑工具。

### 7. 其他策略文件

- **conseca.toml**：Conseca 安全模式相关策略。
- **discovered.toml**：动态发现的工具的默认策略。
- **memory-manager.toml**：内存管理工具的策略。
- **tracker.toml**：追踪器工具的策略。

## 依赖关系

- **被加载方**：这些 TOML 文件由 `../toml-loader.ts` 解析，交给 `../policy-engine.ts` 策略引擎处理。
- **规则结构**：每条规则包含 `toolName`（工具名/通配符）、`decision`（allow/ask_user/deny）、`priority`（优先级数值）、可选的 `modes`（适用模式）、`interactive`（是否交互环境）、`argsPattern`（参数模式匹配）、`safety_checker`（安全检查器）等字段。
- **与审批模式的关系**：上层通过配置选择加载哪些策略文件组合（如选择 YOLO 模式会同时加载 `yolo.toml` + `read-only.toml` + `write.toml` 等），策略引擎合并后按优先级匹配。
