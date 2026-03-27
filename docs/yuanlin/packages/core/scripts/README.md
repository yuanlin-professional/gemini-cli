# packages/core/scripts

## 概述

`packages/core/scripts` 目录包含 core 包的构建辅助脚本，用于在构建阶段打包浏览器 MCP 工具和编译 Windows 平台的沙箱可执行程序。这些脚本在 `npm build` 或类似的构建流程中被调用，不属于运行时代码。

## 目录结构

```
scripts/
├── bundle-browser-mcp.mjs          # 浏览器 MCP 工具打包脚本
└── compile-windows-sandbox.js       # Windows 沙箱编译脚本
```

## 核心组件

### 1. bundle-browser-mcp.mjs — 浏览器 MCP 打包

使用 **esbuild** 将 `chrome-devtools-mcp` 包打包为单文件 ESM 模块，输出到 `dist/bundled/chrome-devtools-mcp.mjs`。

主要功能：
- **选择性排除工具**：读取 `browser-tools-manifest.json` 中的 `exclude` 列表，通过 esbuild 插件将被排除的工具模块替换为空模块（`export {}`），减小打包体积。
- **第三方资源复制**：将 `chrome-devtools-mcp` 的 `third_party` 目录复制到 `dist/bundled/third_party`，同时过滤掉不必要的大文件（如 `lighthouse-devtools-mcp-bundle.js`）。
- **外部依赖处理**：将 `puppeteer-core` 标记为外部依赖，不参与打包。
- **CJS 兼容**：在打包输出顶部注入 `createRequire` 以兼容 CommonJS 的 `require` 调用。

### 2. compile-windows-sandbox.js — Windows 沙箱编译

在 Windows 平台上编译 C# 源码 `GeminiSandbox.cs` 为可执行程序 `GeminiSandbox.exe`，用于提供原生受限令牌（Restricted Token）沙箱隔离能力。

主要功能：
- **平台检测**：仅在 `win32` 平台执行，其他平台跳过。
- **编译器发现**：按优先级查找 `csc.exe`（C# 编译器），依次尝试 PATH、.NET Framework 64 位和 32 位路径。
- **双重输出**：编译结果同时写入 `src/sandbox/windows/` 和 `dist/src/sandbox/windows/` 目录。
- **优雅降级**：如果编译器未找到，打印警告而非报错，沙箱将在首次运行时尝试编译。

## 依赖关系

- **bundle-browser-mcp.mjs**：
  - 构建工具依赖：`esbuild`
  - 数据依赖：`../src/agents/browser/browser-tools-manifest.json`
  - 源码依赖：`chrome-devtools-mcp`（来自 node_modules）

- **compile-windows-sandbox.js**：
  - 系统依赖：Windows .NET Framework 的 `csc.exe` 编译器
  - 源码依赖：`../src/sandbox/windows/GeminiSandbox.cs`
