# .husky

## 概述

Husky 是一个 Git hooks 管理工具，用于在 Git 操作（如 commit、push）时自动执行预定义的脚本。此目录存放项目配置的 Git hooks 脚本，确保代码在提交前通过必要的检查。

## 目录结构

```
.husky/
└── pre-commit   # 预提交钩子脚本，在 git commit 执行前自动运行代码检查（如 lint、格式化等）
```
