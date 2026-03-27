# calculate-vars

## 概述

`calculate-vars` 是一个 GitHub Actions 复合动作（composite action），用于在发布流程中计算和标准化常用变量。其主要功能是解析 `dry_run` 输入参数，并输出一个统一的布尔标志 `is_dry_run`，供下游步骤判断当前运行是否为模拟发布（dry run）。

## 目录结构

```
.github/actions/calculate-vars/
└── action.yml    # 动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `dry_run` | boolean | 否 | 是否为模拟运行 |

### 输出参数（Outputs）

| 参数名 | 说明 |
|--------|------|
| `is_dry_run` | 布尔标志，指示当前运行是否为模拟发布。当输入为空或 `"false"` 时输出 `"false"`，否则输出 `"true"` |

### 执行步骤

1. **Print inputs** -- 将所有输入参数以 JSON 格式打印到日志，便于调试。
2. **Set vars for simplified logic** -- 解析 `dry_run` 输入，将结果标准化为 `"true"` 或 `"false"` 并写入 `GITHUB_OUTPUT`。

## 依赖关系

- **无外部依赖**：该动作为纯 shell 脚本实现的复合动作，不依赖任何第三方 Action 或额外工具。
- **被依赖方**：该动作通常被发布流程中的其他工作流调用，用于在流水线早期确定是否执行真实发布操作。
