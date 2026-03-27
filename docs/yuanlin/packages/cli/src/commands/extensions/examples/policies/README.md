# policies -- 安全策略引擎扩展示例

## 概述

本目录是一个 Gemini CLI 扩展示例，演示如何通过策略引擎（Policy Engine）为 Gemini CLI 添加安全规则和安全检查器。该扩展名为 `policy-example`（版本 1.0.0），展示了三种不同的安全控制模式：用户确认提示（ask_user）、请求拒绝（deny）以及安全检查器（safety_checker）。扩展运行在 Tier 2（扩展层），出于安全考虑，扩展贡献的 `allow` 决策和 `yolo` 模式配置会被 Gemini CLI 忽略，确保扩展只能加强安全控制而不能绕过用户确认。

## 目录结构

```
policies/
├── gemini-extension.json     # 扩展清单文件
├── README.md                 # 原始英文说明文档
└── policies/                 # 策略规则目录
    └── policies.toml         # TOML 格式的策略规则定义
```

## 核心组件

### `gemini-extension.json`

扩展清单文件：

```json
{
  "name": "policy-example",
  "version": "1.0.0",
  "description": "An example extension demonstrating Policy Engine support."
}
```

### `policies/policies.toml`

策略规则定义文件，使用 TOML 格式声明了三条规则：

**规则 1 -- 危险命令确认（ask_user）**

```toml
[[rule]]
toolName = "run_shell_command"
commandPrefix = "rm -rf"
decision = "ask_user"
priority = 100
```

当模型调用 `run_shell_command` 工具且命令以 `rm -rf` 开头时，强制弹出用户确认提示。

**规则 2 -- 敏感文件搜索拒绝（deny）**

```toml
[[rule]]
toolName = "grep_search"
argsPattern = "(\.env|id_rsa|passwd)"
decision = "deny"
priority = 200
denyMessage = "Access to sensitive credentials or system files is restricted by the policy-example extension."
```

当模型调用 `grep_search` 工具且参数匹配 `.env`、`id_rsa` 或 `passwd` 等敏感文件模式时，直接拒绝请求并显示自定义拒绝消息。

**规则 3 -- 写操作路径验证（safety_checker）**

```toml
[[safety_checker]]
toolName = ["write_file", "replace"]
priority = 300
[safety_checker.checker]
type = "in-process"
name = "allowed-path"
required_context = ["environment"]
```

对所有 `write_file` 和 `replace` 工具调用应用 `allowed-path` 安全检查器，验证目标文件路径是否在允许范围内。检查器以进程内（in-process）模式运行，需要 `environment` 上下文信息。

### 规则优先级

三条规则的优先级递增（100 < 200 < 300），高优先级规则优先匹配。

## 依赖关系

- 依赖 Gemini CLI 内置的 Policy Engine 策略引擎（自动扫描 `policies/` 目录下的 TOML 文件）。
- 依赖 Gemini CLI 的 Tier 2 安全模型（扩展层安全约束）。
- 无任何 npm 包依赖，纯声明式配置。
