# run-tests

## 概述

`run-tests` 是一个 GitHub 复合动作（Composite Action），用于执行项目的预检查和集成测试。该动作依次执行项目构建、单元测试、无沙盒集成测试和 Docker 沙盒集成测试，提供完整的自动化测试流程。

## 目录结构

```
.github/actions/run-tests/
└── action.yml    # 复合动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 必填 | 默认值 | 说明 |
|--------|------|--------|------|
| `gemini_api_key` | 是 | -- | 用于运行集成测试的 Gemini API 密钥 |
| `working-directory` | 否 | `.` | 测试执行的工作目录 |

### 执行步骤

1. **打印输入参数** -- 以 JSON 格式输出所有输入参数。
2. **运行测试** -- 在指定工作目录下依次执行以下操作（使用 `::group::` 进行日志分组）：
   - **Build** -- 执行 `npm run build` 构建项目。
   - **Unit Tests** -- 执行 `npm run test:ci` 运行单元测试。
   - **Integration Tests (no sandbox)** -- 执行 `npm run test:integration:sandbox:none` 运行无沙盒模式的集成测试。
   - **Integration Tests (docker sandbox)** -- 执行 `npm run test:integration:sandbox:docker` 运行 Docker 沙盒模式的集成测试。

### 环境变量

- `GEMINI_API_KEY` -- 通过输入参数传入，供集成测试中调用 Gemini API 使用。

## 依赖关系

### npm 脚本

- `npm run build` -- 项目构建脚本
- `npm run test:ci` -- CI 环境单元测试脚本
- `npm run test:integration:sandbox:none` -- 无沙盒集成测试脚本
- `npm run test:integration:sandbox:docker` -- Docker 沙盒集成测试脚本

### 外部服务

- Gemini API -- 集成测试依赖 Gemini API 服务
- Docker -- Docker 沙盒集成测试需要 Docker 运行时环境
