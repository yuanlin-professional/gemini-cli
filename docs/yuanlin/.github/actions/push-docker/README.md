# push-docker

## 概述

`push-docker` 是一个 GitHub 复合动作（Composite Action），用于构建项目包并将 Docker 镜像推送到 GitHub Container Registry（GHCR）。该动作完成从代码检出、依赖安装、项目构建、npm 打包到 Docker 镜像构建与推送的完整流程，并在失败时自动创建 GitHub Issue 进行告警。

## 目录结构

```
.github/actions/push-docker/
└── action.yml    # 复合动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 必填 | 说明 |
|--------|------|------|
| `github-actor` | 是 | GitHub 操作者用户名，用于 GHCR 登录 |
| `github-secret` | 是 | GitHub 密钥，用于 GHCR 认证和失败时创建 Issue |
| `ref-name` | 是 | GitHub 引用名称（分支名/标签名） |
| `github-sha` | 是 | GitHub 提交 SHA 哈希值，用于检出特定提交 |

### 执行步骤

1. **打印输入参数** -- 以 JSON 格式输出所有输入参数，便于调试。
2. **代码检出** -- 使用 `actions/checkout@v4` 检出指定 SHA 的代码，获取完整提交历史（`fetch-depth: 0`）。
3. **安装依赖** -- 执行 `npm install` 安装项目依赖。
4. **设置 Docker Buildx** -- 使用 `docker/setup-buildx-action@v3` 配置 Docker Buildx 构建器。
5. **项目构建** -- 执行 `npm run build` 构建项目。
6. **打包 CLI** -- 通过 `npm pack` 分别打包 `@google/gemini-cli` 和 `@google/gemini-cli-core` 到各自的 `dist` 目录。
7. **GHCR 登录** -- 使用 `docker/login-action@v3` 登录到 `ghcr.io`。
8. **获取分支名** -- 从 `ref-name` 中提取分支名（去除 `/merge` 后缀）。
9. **构建并推送 Docker 镜像** -- 使用 `docker/build-push-action@v6` 构建镜像并推送到 GHCR，生成两个标签：分支名标签和 SHA 标签。
10. **失败告警** -- 若任一步骤失败，自动创建带有 `release-failure` 标签的 GitHub Issue。

### 镜像标签策略

推送的 Docker 镜像会打上两个标签：

- `ghcr.io/<repo>/cli:<分支名>` -- 按分支名标记，便于追踪最新构建。
- `ghcr.io/<repo>/cli:<commit-sha>` -- 按提交哈希标记，确保可追溯到具体提交。

## 依赖关系

### 外部 Actions

- `actions/checkout@v4` -- 代码检出
- `docker/setup-buildx-action@v3` -- Docker Buildx 环境配置
- `docker/login-action@v3` -- Docker 注册中心登录
- `docker/build-push-action@v6` -- Docker 镜像构建与推送

### 项目依赖

- 项目根目录下的 `Dockerfile` -- 镜像构建定义文件
- `@google/gemini-cli` 包 -- CLI 工具包
- `@google/gemini-cli-core` 包 -- 核心功能包
- `gh` CLI 工具 -- 用于失败时创建 Issue
