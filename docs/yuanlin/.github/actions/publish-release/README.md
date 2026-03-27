# publish-release

## 概述

`publish-release` 是一个 GitHub Actions 复合动作，负责 gemini-cli 项目的完整发布流程。它编排了从版本更新、构建打包、npm 发布、版本验证到 GitHub Release 创建的全部环节，支持三个包（core、cli、a2a-server）的协调发布，并提供 dry-run 模式用于模拟发布。

## 目录结构

```
.github/actions/publish-release/
└── action.yml    # 动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 必填 | 默认值 | 说明 |
|--------|------|--------|------|
| `release-version` | 是 | - | 发布版本号（如 `0.1.11`） |
| `npm-tag` | 是 | - | npm 发布标签（如 `latest`、`preview`、`nightly`） |
| `wombat-token-core` | 是 | - | core 包的 npm 发布令牌 |
| `wombat-token-cli` | 是 | - | cli 包的 npm 发布令牌 |
| `wombat-token-a2a-server` | 是 | - | a2a-server 包的 npm 发布令牌 |
| `github-token` | 是 | - | GitHub Token |
| `github-release-token` | 否 | - | 专用于创建 GitHub Release 的 Token（可触发其他工作流） |
| `dry-run` | 是 | - | 是否为模拟运行 |
| `release-tag` | 是 | - | 发布标签（如 `v0.1.11`） |
| `previous-tag` | 是 | - | 上一版本标签（用于生成 Release Notes） |
| `skip-github-release` | 否 | `false` | 是否跳过创建 GitHub Release |
| `working-directory` | 否 | `.` | 工作目录 |
| `force-skip-tests` | 否 | `false` | 是否跳过测试和验证 |
| `skip-branch-cleanup` | 否 | `false` | 是否跳过清理发布分支 |
| `gemini_api_key` | 是 | - | 集成测试用 API Key |
| `npm-registry-publish-url` | 是 | - | npm 发布注册表 URL |
| `npm-registry-url` | 是 | - | npm 注册表 URL |
| `npm-registry-scope` | 是 | - | npm 注册表作用域 |
| `cli-package-name` | 是 | - | CLI 包名称 |
| `core-package-name` | 是 | - | Core 包名称 |
| `a2a-package-name` | 是 | - | A2A 包名称 |

### 执行步骤

1. **Print Inputs** -- 打印输入参数用于调试。
2. **Configure Git User** -- 配置 Git 用户为 `gemini-cli-robot`。
3. **Create release branch** -- 从当前 HEAD 创建 `release/<tag>` 分支。
4. **Update package versions** -- 运行 `npm run release:version` 更新所有包的版本号。
5. **Commit and Push** -- 提交版本变更并推送发布分支（dry-run 时跳过推送）。
6. **Build and Prepare** -- 执行 `npm run build:packages` 和 `npm run prepare:package`。
7. **Bundle** -- 执行 `npm run bundle` 生成打包文件。
8. **Prepare for GitHub release** -- 针对 GitHub Packages 注册表执行特殊准备脚本。
9. **Configure npm** -- 使用 `actions/setup-node` 配置 npm 注册表。
10. **Publish CORE** -- 获取 core 令牌并发布 core 包。
11. **Install latest core** -- 安装刚发布的 core 包到 cli 和 a2a 工作区。
12. **Prepare bundled CLI** -- 针对非 GitHub Packages 且非 latest 标签执行 npm 发布准备脚本。
13. **Publish CLI** -- 获取 cli 令牌并发布 cli 包。
14. **Publish a2a** -- 获取 a2a-server 令牌并发布 a2a-server 包。
15. **Verify NPM release** -- 调用 `verify-release` 动作验证发布成功。
16. **Tag release** -- 调用 `tag-npm-release` 动作为发布添加 npm dist-tag。
17. **Create GitHub Release** -- 使用 `gh release create` 创建 GitHub Release（附带 bundle 文件和自动生成的 Release Notes）。
18. **Clean up release branch** -- 删除远程发布分支。

## 依赖关系

### 内部依赖（同项目 Actions）

- **`.github/actions/npm-auth-token`** -- 为每个包获取对应的 npm 发布令牌（被调用 3 次）。
- **`.github/actions/verify-release`** -- 验证 npm 发布是否成功。
- **`.github/actions/tag-npm-release`** -- 为已发布的包添加 npm dist-tag。

### 外部依赖

- **`actions/setup-node`** -- 配置 Node.js 环境和 npm 注册表认证。
- **GitHub CLI (`gh`)** -- 用于创建 GitHub Release。

### 项目脚本依赖

- `npm run release:version` -- 版本号更新脚本
- `npm run build:packages` -- 包构建脚本
- `npm run prepare:package` -- 包准备脚本
- `npm run bundle` -- 打包脚本
- `scripts/prepare-github-release.js` -- GitHub 发布准备脚本
- `scripts/prepare-npm-release.js` -- npm 发布准备脚本
