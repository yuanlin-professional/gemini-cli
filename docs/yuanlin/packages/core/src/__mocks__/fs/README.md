# fs

## 概述

Core 包中 Node.js `fs` 模块的 mock 实现目录，供 Vitest 测试框架使用。通过有选择地模拟 `fs/promises` 模块中的特定方法（如 `readFile`），使测试能够控制文件系统读取行为，同时保持其他文件系统操作使用真实实现。

## 目录结构

```
__mocks__/fs/
└── promises.ts   # fs/promises 模块的部分 mock 实现
```

## 核心组件

- **promises.ts**：对 `node:fs/promises` 模块的部分 mock。导出一个 `mockControl` 控制对象，其中包含 `mockReadFile`（由 `vi.fn()` 创建的 mock 函数），测试中可通过该对象配置 `readFile` 的返回值。除 `readFile` 外，`access`、`writeFile`、`mkdir`、`stat`、`readdir` 等所有其他方法均从真实的 `node:fs/promises` 模块中原样导出，保证不影响非目标功能的正常执行。
