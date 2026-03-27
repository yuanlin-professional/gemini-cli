# policies

## 概述

`policies` 是 policy-example 扩展的策略规则定义目录，包含 TOML 格式的策略配置文件。策略引擎允许扩展在 Tier 2（扩展层级）运行安全规则，控制模型对工具的访问权限。需注意扩展层级中 `allow` 决策和 `yolo` 模式配置会被忽略，仅支持 `ask_user`（询问用户）和 `deny`（拒绝）决策。

## 目录结构

```
policies/
└── policies.toml    # 策略规则定义文件
```

## 核心组件

- **policies.toml** - 策略规则配置文件，定义了三类安全规则：
  1. **危险命令拦截规则** (`priority=100`) - 当模型试图执行 `rm -rf` 前缀的 shell 命令时，要求用户确认（`ask_user`）。
  2. **敏感文件访问拒绝规则** (`priority=200`) - 禁止通过 grep 工具搜索 `.env`、`id_rsa`、`passwd` 等敏感文件，直接拒绝并返回自定义提示信息。
  3. **写操作路径安全检查器** (`priority=300`) - 对 `write_file` 和 `replace` 工具应用 `allowed-path` 进程内安全检查器，基于环境上下文验证写入路径的合法性。
