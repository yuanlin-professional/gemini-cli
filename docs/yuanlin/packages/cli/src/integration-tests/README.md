# integration-tests

## 概述

CLI 包的集成测试目录，使用 Vitest 框架编写端到端集成测试，验证 Gemini CLI 各模块协同工作的正确性。测试依赖 `AppRig` 测试工具类来模拟完整的应用运行环境，并通过预录的假响应文件驱动模型行为。

## 目录结构

```
integration-tests/
└── modelSteering.test.tsx   # 模型引导（Model Steering）功能的集成测试
```

## 核心组件

- **modelSteering.test.tsx**：测试模型引导功能的集成场景。该测试通过 `AppRig` 初始化一个完整的应用实例，加载 `steering.responses` 假响应文件，模拟用户发起长任务后在工具执行过程中注入引导提示（hint）。测试验证：当模型调用 `list_directory` 工具时，用户注入 "focus on .txt" 提示后，模型能根据提示调整后续行为，仅读取 `.txt` 文件并正确完成任务。测试还涉及工具策略设置（`PolicyDecision.ASK_USER`）和工具确认解析等交互流程。
