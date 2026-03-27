# push-sandbox

## 概述

`push-sandbox` 是一个 GitHub 复合动作（Composite Action），用于构建沙盒（Sandbox）Docker 镜像并推送到 Docker Hub 容器注册中心。该动作支持多架构构建（amd64 和 arm64），包含镜像验证步骤以确保构建产物的完整性，并支持 dry-run 模式用于测试。

## 目录结构

```
.github/actions/push-sandbox/
└── action.yml    # 复合动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 必填 | 说明 |
|--------|------|------|
| `github-actor` | 是 | GitHub 操作者用户名 |
| `github-secret` | 是 | GitHub 密钥，用于认证和失败告警 |
| `dockerhub-username` | 是 | Docker Hub 用户名 |
| `dockerhub-token` | 是 | Docker Hub 个人访问令牌（需读写权限） |
| `github-sha` | 是 | GitHub 提交 SHA 哈希值 |
| `github-ref-name` | 是 | GitHub 引用名称（分支名/标签名） |
| `dry-run` | 是 | 是否为模拟运行模式（boolean） |

### 执行步骤

1. **打印输入参数** -- 以 JSON 格式输出所有输入参数。
2. **代码检出** -- 使用 `actions/checkout@v4` 检出指定 SHA 的代码。
3. **安装依赖** -- 执行 `npm install`。
4. **项目构建** -- 执行 `npm run build`。
5. **设置 QEMU** -- 使用 `docker/setup-qemu-action@v3` 配置 QEMU，用于多架构构建支持。
6. **设置 Docker Buildx** -- 使用 `docker/setup-buildx-action@v3` 配置 Buildx。
7. **Docker Hub 登录** -- 使用 `docker/login-action@v3` 登录到 `docker.io`。
8. **确定镜像标签** -- 根据引用名称判断：若匹配语义化版本标签（如 `v1.2.3`），使用版本号作为标签；否则使用提交 SHA。
9. **构建镜像（amd64）** -- 先构建 amd64 单架构镜像用于验证，使用 `npm run build:sandbox` 命令。
10. **验证镜像** -- 运行构建的镜像，检验 `@google/gemini-cli` 和 `@google/gemini-cli-core` 的 `package.json` 是否可正常解析，以及 `gemini --version` 是否可执行。
11. **发布镜像（多架构）** -- 非 dry-run 模式下，构建并推送 amd64 和 arm64 双架构镜像到 Docker Hub。
12. **失败告警** -- 若任一步骤失败，自动创建带有 `release-failure` 标签的 GitHub Issue。

### 镜像标签策略

- **发布版本**：若 `github-ref-name` 匹配 `v{major}.{minor}.{patch}[-prerelease]` 格式，标签为去掉 `v` 前缀后的版本号（如 `v1.2.3` 变为 `1.2.3`）。
- **开发版本**：使用提交 SHA 作为标签。
- 镜像名称为 `google/gemini-cli-sandbox`。

## 依赖关系

### 外部 Actions

- `actions/checkout@v4` -- 代码检出
- `docker/setup-qemu-action@v3` -- QEMU 多架构模拟支持
- `docker/setup-buildx-action@v3` -- Docker Buildx 环境配置
- `docker/login-action@v3` -- Docker 注册中心登录

### 项目依赖

- `npm run build:sandbox` 脚本 -- 沙盒镜像构建脚本
- `@google/gemini-cli` 包 -- CLI 工具包（镜像内安装）
- `@google/gemini-cli-core` 包 -- 核心功能包（镜像内安装）
- `gh` CLI 工具 -- 用于失败时创建 Issue
