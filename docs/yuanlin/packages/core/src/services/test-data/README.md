# test-data (core/src/services)

## 概述

此目录存放 `packages/core/src/services/` 下服务层测试所需的黄金文件（golden files）。包含别名解析（resolved-aliases）的预期输出数据，用于验证别名解析服务的正确性，其中包括常规场景和重试场景两组数据。

## 目录结构

```
test-data/
├── resolved-aliases-retry.golden.json
└── resolved-aliases.golden.json
```
