# scripts

## 概述

VS Code IDE Companion 扩展的构建和发布辅助脚本目录，包含用于检查发布状态和生成第三方许可声明的自动化脚本。

## 目录结构

```
scripts/
├── check-vscode-release.js   # 检查是否需要发布新版本
└── generate-notices.js        # 生成第三方依赖许可声明
```

## 核心组件

- **check-vscode-release.js**：VS Code 扩展发布检查脚本。工作流程分三步：(1) 通过 `gcloud storage ls` 查询 GCS 存储桶 (`gs://gemini-cli-vscode-extension/release/1p/signed`) 中最新的已签名 `.vsix` 文件，从文件名中提取版本号和 Git 提交哈希（7位短哈希）；(2) 使用 `git log` 对比该提交与 `origin/main` 之间在 `packages/vscode-ide-companion` 路径下的新提交（排除版本 bump 提交）；(3) 使用 `git diff` 检查 `NOTICES.txt` 是否有变化以判断依赖是否变更。最终输出汇总报告，说明是否需要新发布以及是否需要额外的许可审查。
- **generate-notices.js**：生成第三方依赖的许可声明文件。
