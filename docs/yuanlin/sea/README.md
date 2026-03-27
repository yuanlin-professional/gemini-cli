# sea/

## 概述

`sea/` 目录包含 Node.js **Single Executable Application (SEA)** 的启动器代码和对应测试。SEA 是 Node.js 的实验性特性，允许将 Node.js 应用打包为单个可执行二进制文件。`sea-launch.cjs` 是该二进制文件的入口点，负责从嵌入的资产中安全地提取运行时文件，验证完整性后动态加载主程序。

## 目录结构

```
sea/
├── sea-launch.cjs        # SEA 启动器（CommonJS 格式，二进制入口点）
└── sea-launch.test.js    # 启动器的单元测试
```

## 架构图

```mermaid
graph TB
    subgraph SEA二进制文件
        Binary["gemini 二进制文件<br/>（Node.js + 嵌入资产）"]
        SeaLaunch["sea-launch.cjs<br/>启动器入口"]
        Manifest["manifest.json<br/>资产清单"]
        MainBundle["gemini.mjs<br/>主程序 Bundle"]
        Chunks["*.js Chunks<br/>代码分割产物"]
        NativeModules["native_modules/<br/>原生模块（.node 文件）"]
        SandboxFiles["*.sb 文件<br/>沙箱配置"]
        PolicyFiles["*.toml 文件<br/>策略定义"]
    end

    subgraph 运行时目录
        RuntimeDir["临时运行时目录<br/>gemini-runtime-{版本}-{用户名}"]
        ExtractedMain["gemini.mjs"]
        ExtractedChunks["*.js Chunks"]
        ExtractedNative["native_modules/"]
    end

    Binary --> SeaLaunch
    SeaLaunch -->|"1. 读取清单"| Manifest
    SeaLaunch -->|"2. 检查/创建运行时"| RuntimeDir
    SeaLaunch -->|"3. 提取资产"| MainBundle
    SeaLaunch -->|"3. 提取资产"| Chunks
    SeaLaunch -->|"3. 提取资产"| NativeModules
    MainBundle --> ExtractedMain
    Chunks --> ExtractedChunks
    NativeModules --> ExtractedNative
    SeaLaunch -->|"4. 动态 import"| ExtractedMain
```

## 核心组件

### sea-launch.cjs

SEA 二进制的入口文件，使用 **CommonJS** 格式（Node SEA 要求入口必须为 CJS）。该文件实现了以下核心功能：

#### 导出函数

| 函数 | 说明 |
|------|------|
| `sanitizeArgv(argv, execPath, resolveFn)` | 清理 Node SEA 有时注入的"幽灵"参数（argv[2] 等于 argv[0] 的情况） |
| `getSafeName(name)` | 将字符串净化为安全的文件路径名（仅保留字母数字和 `.-`） |
| `verifyIntegrity(dir, manifest, fsMod, cryptoMod)` | 使用 SHA-256 验证运行时目录中所有文件的完整性 |
| `prepareRuntime(manifest, getAssetFn, deps)` | 准备运行时目录，必要时从嵌入资产中提取文件 |
| `main(getAssetFn)` | 主函数，串联整个启动流程 |

#### 启动流程详解

```mermaid
flowchart TD
    A["main() 启动"] --> B["设置 IS_BINARY=true"]
    B --> C["启用编译缓存<br/>enableCompileCache()"]
    C --> D["清理幽灵参数<br/>sanitizeArgv()"]
    D --> E["读取嵌入的 manifest.json"]
    E --> F{"manifest 存在?"}
    F -->|否| G["错误退出: 二进制损坏"]
    F -->|是| H["prepareRuntime()"]
    H --> I{"运行时目录存在?"}
    I -->|是| J{"目录安全?<br/>（所有权 + 权限 0700）"}
    J -->|是| K{"完整性验证通过?<br/>（SHA-256 哈希）"}
    K -->|是| L["复用已有运行时"]
    K -->|否| M["删除并重建"]
    J -->|否| M
    I -->|否| M
    M --> N["创建临时 setup 目录"]
    N --> O["从 SEA 资产提取所有文件"]
    O --> P["原子 rename 到最终目录"]
    P --> Q{"rename 成功?"}
    Q -->|是| L
    Q -->|否| R{"最终目录已被其他进程创建?"}
    R -->|是且验证通过| L
    R -->|否| S["错误退出: 运行时设置失败"]
    L --> T["import(gemini.mjs)"]
    T --> U["CLI 正常运行"]
```

#### 安全特性

1. **目录所有权检查**: 验证运行时目录的 UID 与当前进程 UID 一致
2. **权限检查**: 确保目录权限为 0700（仅所有者可访问，Windows 跳过）
3. **完整性校验**: 对每个文件进行 SHA-256 哈希验证，与 manifest 中的记录比对
4. **原子性操作**: 先在临时目录提取，然后 `rename` 到最终位置，避免部分提取的损坏状态
5. **并发安全**: 如果 `rename` 失败（其他进程抢先创建），会检查已有目录的完整性

#### 运行时目录命名

```
{tempBase}/gemini-runtime-{version}-{username}
```

- Windows 上优先使用 `%LOCALAPPDATA%/Google/GeminiCLI/`
- 其他平台使用系统 tmpdir

### sea-launch.test.js

使用 Vitest 编写的全面测试套件，覆盖以下场景：
- `sanitizeArgv`: 幽灵参数的清理和正常参数的保留
- `getSafeName`: 特殊字符的净化
- `verifyIntegrity`: 正确哈希和篡改检测
- `prepareRuntime`: 新建、复用、并发、权限异常等各种场景
- `main`: 完整的启动流程测试

## 依赖关系

### 内部依赖

- **构建端**: `scripts/build_binary.js` 负责调用 `node --experimental-sea-config` 生成 SEA blob，其中 `sea-launch.cjs` 被指定为主入口
- **资产来源**: `bundle/` 目录下的所有打包产物通过 `node:sea` API 的 `getAsset()` 嵌入

### 外部依赖（Node.js 内置模块）

| 模块 | 用途 |
|------|------|
| `node:sea` | SEA 资产读取 API |
| `node:fs` | 文件系统操作 |
| `node:os` | 系统信息（tmpdir、userInfo） |
| `node:crypto` | SHA-256 完整性校验 |
| `node:path` | 路径处理 |
| `node:url` | pathToFileURL 转换 |
| `node:module` | enableCompileCache 编译缓存 |

## 数据流

### SEA 构建流程（在 build_binary.js 中）

```mermaid
flowchart LR
    A["esbuild 打包<br/>bundle/gemini.js + chunks"] --> B["生成 manifest.json<br/>文件列表 + SHA-256 哈希"]
    B --> C["生成 sea-config.json<br/>入口 + 资产映射"]
    C --> D["node --experimental-sea-config<br/>生成 sea-prep.blob"]
    D --> E["复制 Node 二进制"]
    E --> F["npx postject 注入 blob"]
    F --> G["codesign/signtool 签名"]
    G --> H["最终 gemini 二进制"]
```

### 运行时资产结构

```
manifest.json
├── main: "gemini.mjs"
├── mainHash: "sha256..."
├── version: "0.36.0"
└── files[]
    ├── { key: "chunk-XXX.js", path: "chunk-XXX.js", hash: "sha256..." }
    ├── { key: "files:node_modules/@lydell/...", path: "node_modules/...", hash: "sha256..." }
    ├── { key: "sandbox-macos-*.sb", path: "sandbox-macos-*.sb", hash: "sha256..." }
    └── { key: "policies:*.toml", path: "policies/*.toml", hash: "sha256..." }
```
