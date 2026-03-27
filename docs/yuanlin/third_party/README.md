# third_party/

## 概述

`third_party/` 目录存放 Gemini CLI 项目使用的第三方依赖代码，这些代码不通过 npm registry 直接安装，而是以源码形式包含在项目中。目前仅包含一个子模块 `get-ripgrep`，用于下载和管理 ripgrep 搜索工具的预编译二进制文件。

## 目录结构

```
third_party/
└── get-ripgrep/               # ripgrep 二进制下载与管理模块
    ├── package.json           # 模块配置（@lvce-editor/ripgrep）
    ├── LICENSE                # MIT 许可证
    └── src/
        ├── index.js           # 模块入口，导出 rgPath
        └── downloadRipGrep.js # ripgrep 二进制下载逻辑
```

## 架构图

```mermaid
graph TB
    subgraph GeminiCLI["Gemini CLI"]
        Core["packages/core<br/>核心模块"]
        FileSearch["文件搜索功能"]
    end

    subgraph ThirdParty["third_party/get-ripgrep"]
        Index["src/index.js<br/>导出 rgPath"]
        Download["src/downloadRipGrep.js<br/>下载逻辑"]
    end

    subgraph 外部资源
        GitHub["github.com/microsoft/<br/>ripgrep-prebuilt<br/>预编译二进制发布"]
        Cache["XDG 缓存目录<br/>vscode-ripgrep/"]
    end

    subgraph 本地产物
        Bin["third_party/get-ripgrep/bin/<br/>rg 二进制文件"]
    end

    Core --> FileSearch
    FileSearch --> Index
    Index --> Bin
    Download --> GitHub
    Download --> Cache
    Cache --> Bin
```

## 核心组件

### get-ripgrep 模块

该模块源自 [lvce-editor/ripgrep](https://github.com/lvce-editor/ripgrep) 项目，经过适配后集成到 Gemini CLI 中。它提供了跨平台的 ripgrep 二进制文件下载和路径获取能力。

**ripgrep** 是一个极速的正则表达式搜索工具，Gemini CLI 在代码库搜索功能中使用它来实现高性能文件内容检索。

#### 模块信息

| 字段 | 值 |
|------|-----|
| 包名 | `@lvce-editor/ripgrep` |
| 许可证 | MIT |
| 模块格式 | ESM |
| 原始仓库 | https://github.com/lvce-editor/ripgrep |
| ripgrep 版本 | v13.0.0-10（来自 microsoft/ripgrep-prebuilt） |

详细信息请参阅:
- [get-ripgrep/README.md](./get-ripgrep/README.md)
- [get-ripgrep/src/README.md](./get-ripgrep/src/README.md)

## 依赖关系

### 内部依赖

- `packages/core` 通过引用 `get-ripgrep` 模块获取 ripgrep 二进制路径，用于代码搜索功能

### 外部依赖

`get-ripgrep` 模块自身的依赖：

| 依赖包 | 用途 |
|--------|------|
| `got` | HTTP 下载客户端 |
| `extract-zip` | ZIP 解压（Windows 平台） |
| `execa` | 命令行执行（tar 解压） |
| `fs-extra` | 增强文件操作 |
| `tempy` | 临时文件管理 |
| `path-exists` | 路径存在检查 |
| `xdg-basedir` | XDG 缓存目录获取 |
| `@lvce-editor/verror` | 错误链封装 |

## 数据流

### ripgrep 二进制获取流程

```mermaid
flowchart TD
    A["Gemini CLI 需要 ripgrep"] --> B["require get-ripgrep"]
    B --> C["index.js 返回 rgPath"]
    C --> D{"bin/rg 存在?"}
    D -->|是| E["直接使用"]
    D -->|否| F["postinstall 触发<br/>downloadRipGrep()"]
    F --> G["根据平台/架构<br/>确定下载目标"]
    G --> H{"XDG 缓存已有?"}
    H -->|是| I["使用缓存"]
    H -->|否| J["从 GitHub 下载<br/>microsoft/ripgrep-prebuilt"]
    J --> K["保存到 XDG 缓存"]
    I --> L{".tar.gz?"}
    K --> L
    L -->|是| M["tar 解压到 bin/"]
    L -->|否| N["zip 解压到 bin/"]
    M --> E
    N --> E
```
