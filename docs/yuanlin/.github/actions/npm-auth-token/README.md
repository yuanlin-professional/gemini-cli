# npm-auth-token

## 概述

`npm-auth-token` 是一个 GitHub Actions 复合动作，用于根据包名称动态选择正确的 NPM 认证令牌。它支持多个包的发布认证，能够区分私有 GitHub 包（`@google-gemini/` 作用域）和公共 npm 包（`@google/gemini-cli`、`@google/gemini-cli-core`、`@google/gemini-cli-a2a-server`），为每个包返回对应的发布令牌。

## 目录结构

```
.github/actions/npm-auth-token/
└── action.yml    # 动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 必填 | 说明 |
|--------|------|------|
| `package-name` | 是 | 要发布的包名称 |
| `github-token` | 是 | GitHub Token（用于私有 GitHub 包） |
| `wombat-token-core` | 是 | `@google/gemini-cli-core` 包的 npm 发布令牌 |
| `wombat-token-cli` | 是 | `@google/gemini-cli` 包的 npm 发布令牌 |
| `wombat-token-a2a-server` | 是 | `@google/gemini-cli-a2a-server` 包的 npm 发布令牌 |

### 输出参数（Outputs）

| 参数名 | 说明 |
|--------|------|
| `auth-token` | 根据包名称选定的 NPM 认证令牌 |

### 令牌选择逻辑

| 包名称模式 | 使用的令牌 |
|-----------|-----------|
| `@google-gemini/*`（私有包） | `github-token` |
| `@google/gemini-cli` | `wombat-token-cli` |
| `@google/gemini-cli-core` | `wombat-token-core` |
| `@google/gemini-cli-a2a-server` | `wombat-token-a2a-server` |

## 依赖关系

- **无外部依赖**：纯 shell 脚本实现，无需第三方 Action。
- **被依赖方**：被 `publish-release` 动作多次调用，分别为 core、cli 和 a2a-server 三个包获取对应的发布令牌。
