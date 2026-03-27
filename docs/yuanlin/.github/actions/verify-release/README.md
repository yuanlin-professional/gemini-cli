# verify-release

## 概述

该目录定义了一个 GitHub Actions 复合动作（Composite Action），用于验证 NPM 发布的正确性。该动作从指定的 NPM 注册表安装包，执行版本号冒烟测试（npm install 和 npx 两种方式），并运行集成测试，确保发布到 NPM 的包功能正常。

## 目录结构

```
verify-release/
└── action.yml    # GitHub Actions 复合动作定义文件
```

## 核心组件

- **action.yml**：复合动作配置，接受以下输入参数：
  - `npm-package`：NPM 包名（默认 `@google/gemini-cli@latest`）
  - `npm-registry-url`、`npm-registry-scope`：注册表地址与作用域
  - `expected-version`：预期的版本号
  - `gemini_api_key`、`github-token`：集成测试所需凭据

  执行步骤：
  1. **环境准备**：设置 Node.js 20、配置 `.npmrc`、清除 npm 缓存
  2. **NPM 安装**：使用 `nick-fields/retry` 进行最多 10 次重试安装（超时 900 秒，间隔 30 秒）
  3. **冒烟测试 - npm install**：验证全局安装的 `gemini --version` 与预期版本一致
  4. **冒烟测试 - npx 运行**：验证 `npx` 方式运行时版本一致
  5. **集成测试**：安装项目依赖后运行 `npm run test:integration:sandbox:none`，并禁用 CI 模式以兼容交互式测试
