# src (get-ripgrep)

## 概述

该目录是 `get-ripgrep` 第三方模块的源码目录，负责提供 ripgrep（`rg`）二进制文件的路径导出和下载功能。ripgrep 是 Gemini CLI 用于高性能文件内容搜索的核心依赖工具。

## 目录结构

```
src/
├── downloadRipGrep.js   # ripgrep 二进制文件的下载脚本
└── index.js             # 模块入口，导出当前平台对应的 rg 二进制路径
```

## 核心组件

- **index.js**：模块入口文件，导出 `rgPath` 变量。通过 `import.meta.url` 解析当前模块目录，拼接 `../bin/rg`（Windows 下为 `rg.exe`）得到平台对应的 ripgrep 二进制文件绝对路径。该路径供 Gemini CLI 的搜索功能直接调用。原始代码基于 Lvce Editor 项目（MIT 许可证）。

- **downloadRipGrep.js**：负责下载与安装 ripgrep 二进制文件到 `bin/` 目录的脚本，确保运行环境中存在可用的 `rg` 命令。
