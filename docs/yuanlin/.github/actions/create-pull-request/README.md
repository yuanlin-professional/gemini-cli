# create-pull-request

## 概述

`create-pull-request` 是一个 GitHub Actions 复合动作，用于自动创建 Pull Request 并启用自动合并。它封装了 `gh pr create` 和 `gh pr merge --auto` 命令，适用于发布流程中需要自动提交合并请求的场景（如将发布分支合并回主分支）。支持 dry-run 模式，在模拟运行时跳过实际创建操作。

## 目录结构

```
.github/actions/create-pull-request/
└── action.yml    # 动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 必填 | 默认值 | 说明 |
|--------|------|--------|------|
| `branch-name` | 是 | - | 创建 PR 的源分支名称 |
| `pr-title` | 是 | - | Pull Request 标题 |
| `pr-body` | 是 | - | Pull Request 正文内容 |
| `base-branch` | 是 | `main` | 目标合并分支 |
| `github-token` | 是 | - | 用于创建 PR 的 GitHub Token |
| `dry-run` | 否 | `false` | 是否为模拟运行模式 |
| `working-directory` | 否 | `.` | 命令执行的工作目录 |

### 执行步骤

1. **Print Inputs** -- 将所有输入参数以 JSON 格式打印到日志。
2. **Creates a Pull Request** -- 当 `dry-run` 不为 `true` 时执行：
   - 先验证远程分支是否存在（`git ls-remote`），不存在则报错退出。
   - 使用 `gh pr create` 创建 Pull Request。
   - 使用 `gh pr merge --auto` 对新创建的 PR 启用自动合并。

## 依赖关系

- **GitHub CLI (`gh`)** -- 依赖 GitHub 运行器内置的 `gh` 命令行工具来创建和合并 PR。
- **GitHub Token** -- 需要具有创建 PR 和启用自动合并权限的 Token。
- **被依赖方** -- 通常在发布工作流中被调用，用于将版本变更合并回主分支。
