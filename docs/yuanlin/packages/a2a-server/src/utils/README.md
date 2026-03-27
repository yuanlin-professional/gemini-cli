# packages/a2a-server/src/utils

## 概述

工具函数目录，提供日志记录、执行器辅助函数和测试工具。

## 目录结构

```
utils/
├── logger.ts           # Winston 日志器配置
├── executor_utils.ts   # 执行器辅助函数
└── testing_utils.ts    # 测试辅助工具（Mock Config / 请求构建 / 断言）
```

## 核心组件

### logger (`logger.ts`)

基于 Winston 的日志器，配置了自定义格式：
- 格式：`[LEVEL] YYYY-MM-DD HH:mm:ss.SSS AM/PM -- message`
- 输出：控制台
- 级别：info

### pushTaskStateFailed (`executor_utils.ts`)

辅助函数，用于在执行器中将任务状态推送为 "failed"，发布带有错误信息的状态更新事件。

### testing_utils (`testing_utils.ts`)

测试辅助工具：
- `createMockConfig()` - 创建模拟 Config 对象
- `createStreamMessageRequest()` - 构建流式消息请求
- `assertUniqueFinalEventIsLast()` - 断言最终事件在末尾
- `assertTaskCreationAndWorkingStatus()` - 断言任务创建和工作状态

## 依赖关系

### 外部依赖
- `winston` (^3.17.0) - 日志框架
- `@a2a-js/sdk` - A2A 协议类型
- `vitest` - 测试框架（testing_utils 使用）
