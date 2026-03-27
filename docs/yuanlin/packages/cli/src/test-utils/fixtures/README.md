# fixtures

## 概述

测试工具库的数据夹具（fixture）目录，存放预录的假模型响应文件，用于集成测试中替代真实 Gemini API 调用。每个 `.responses` 文件包含一系列 JSON 行，模拟 `generateContentStream` 方法的响应序列。

## 目录结构

```
fixtures/
├── simple.responses      # 简单场景的假响应数据
└── steering.responses    # 模型引导（Model Steering）场景的假响应数据
```

## 核心组件

- **steering.responses**：模型引导集成测试使用的假响应文件。包含三条 JSON 行，模拟一个完整的多轮工具调用场景：(1) 模型响应文本"开始长任务"并调用 `list_directory` 工具列出文件；(2) 模型响应"由于你要我关注 .txt 文件，我将读取 file1.txt"并调用 `read_file` 工具（体现了用户注入的引导提示被模型采纳）；(3) 模型响应"任务完成"。每行的格式为 `{"method":"generateContentStream","response":[...]}`，包含 `candidates` 数组，其中 `parts` 可同时包含文本和函数调用。
