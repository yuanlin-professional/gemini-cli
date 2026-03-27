# tag-npm-release

## 概述

`tag-npm-release` 是一个 GitHub 复合动作（Composite Action），用于为已发布的 npm 包版本添加分发标签（dist-tag）。该动作支持对三个包（`@google/gemini-cli`、`@google/gemini-cli-core` 和 `@google/gemini-cli-a2a-server`）同时进行标签操作，并支持 dry-run 模式进行模拟测试。

## 目录结构

```
.github/actions/tag-npm-release/
└── action.yml    # 复合动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 必填 | 默认值 | 说明 |
|--------|------|--------|------|
| `channel` | 是 | -- | NPM 分发标签名称（如 `latest`、`next`、`beta`） |
| `version` | 是 | -- | 要标记的版本号 |
| `dry-run` | 是 | -- | 是否为模拟运行模式 |
| `github-token` | 是 | -- | GitHub Token，用于创建发布和认证 |
| `wombat-token-core` | 是 | -- | `@google/gemini-cli-core` 的 Wombat npm 令牌 |
| `wombat-token-cli` | 是 | -- | `@google/gemini-cli` 的 Wombat npm 令牌 |
| `wombat-token-a2a-server` | 是 | -- | `@google/gemini-cli-a2a-server` 的 Wombat npm 令牌 |
| `cli-package-name` | 是 | -- | CLI 包名称 |
| `core-package-name` | 是 | -- | Core 包名称 |
| `a2a-package-name` | 是 | -- | A2A 包名称 |
| `working-directory` | 否 | `.` | 命令执行的工作目录 |

### 执行步骤

1. **打印输入参数** -- 以 JSON 格式输出所有输入参数。
2. **设置 Node.js** -- 使用 `actions/setup-node` 根据 `.nvmrc` 配置 Node.js 版本。
3. **配置 .npmrc** -- 调用 `.github/actions/setup-npmrc` 子动作配置注册中心认证。
4. **获取 Core 包令牌** -- 调用 `.github/actions/npm-auth-token` 获取 `core` 包的认证令牌。
5. **标记 Core 包** -- 非 dry-run 模式下，执行 `npm dist-tag add` 为 Core 包的指定版本添加标签。
6. **获取 CLI 包令牌** -- 调用 `.github/actions/npm-auth-token` 获取 `cli` 包的认证令牌。
7. **标记 CLI 包** -- 非 dry-run 模式下，执行 `npm dist-tag add` 为 CLI 包的指定版本添加标签。
8. **获取 A2A 包令牌** -- 调用 `.github/actions/npm-auth-token` 获取 `a2a` 包的认证令牌。
9. **标记 A2A 包** -- 非 dry-run 模式下，执行 `npm dist-tag add` 为 A2A 包的指定版本添加标签。
10. **记录 dry-run** -- dry-run 模式下，输出模拟操作的日志信息。

### npm dist-tag 命令

该动作的核心操作为：

```bash
npm dist-tag add <package-name>@<version> <channel>
```

此命令将指定包的特定版本关联到指定的分发渠道标签上，例如将 `@google/gemini-cli@1.2.3` 标记为 `latest`。

## 依赖关系

### 内部 Actions

- `.github/actions/setup-npmrc` -- 配置 npm 注册中心认证
- `.github/actions/npm-auth-token` -- 获取各个包的 npm 认证令牌

### 外部 Actions

- `actions/setup-node` -- Node.js 环境配置

### npm 包

- `@google/gemini-cli` -- CLI 工具包
- `@google/gemini-cli-core` -- 核心功能包
- `@google/gemini-cli-a2a-server` -- A2A 服务器包

### 外部服务

- **Wombat Dressing Room** -- Google 的 npm 发布代理服务，为每个包提供独立的认证令牌
