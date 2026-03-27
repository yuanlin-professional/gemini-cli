# setup-npmrc

## 概述

`setup-npmrc` 是一个 GitHub 复合动作（Composite Action），用于配置 `.npmrc` 文件以设置 npm 注册中心的只读访问权限。该动作为项目中使用的 `@google-gemini` 和 `@google` 作用域包配置各自的注册中心地址和认证令牌。

## 目录结构

```
.github/actions/setup-npmrc/
└── action.yml    # 复合动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 必填 | 说明 |
|--------|------|------|
| `github-token` | 是 | GitHub Token，用于 GitHub Packages 的认证 |

### 输出参数（Outputs）

| 参数名 | 说明 |
|--------|------|
| `auth-token` | 生成的 NPM 认证令牌 |

### 执行步骤

1. **配置 .npmrc** -- 向用户主目录下的 `~/.npmrc` 写入以下配置：
   - 将 `@google-gemini` 作用域指向 GitHub Packages 注册中心（`https://npm.pkg.github.com`），并设置对应的认证令牌。
   - 将 `@google` 作用域指向 Wombat Dressing Room 注册中心（`https://wombat-dressing-room.appspot.com`），这是 Google 的 npm 包发布代理服务。

### 注册中心映射

| 包作用域 | 注册中心 | 说明 |
|----------|----------|------|
| `@google-gemini` | `https://npm.pkg.github.com` | GitHub Packages，使用 GitHub Token 认证 |
| `@google` | `https://wombat-dressing-room.appspot.com` | Google 的 npm 发布代理服务 |

## 依赖关系

### 外部服务

- **GitHub Packages** -- `@google-gemini` 作用域包的托管注册中心
- **Wombat Dressing Room** -- Google 的 npm 包发布代理服务，托管 `@google` 作用域包

### 被引用关系

- 被 `.github/actions/tag-npm-release/action.yml` 作为子步骤调用，用于在 npm 标签操作前配置注册中心认证。
