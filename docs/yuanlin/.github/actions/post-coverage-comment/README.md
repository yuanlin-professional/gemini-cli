# post-coverage-comment

## 概述

`post-coverage-comment` 是一个 GitHub Actions 复合动作，用于将代码覆盖率摘要以评论形式发布到 Pull Request 上。它从 CLI 和 Core 两个包的覆盖率报告中提取关键指标（行覆盖率、语句覆盖率、函数覆盖率、分支覆盖率），生成格式化的 Markdown 评论，并通过第三方 Action 自动发布或更新 PR 评论。

## 目录结构

```
.github/actions/post-coverage-comment/
└── action.yml    # 动作定义文件
```

## 核心组件

### 输入参数（Inputs）

| 参数名 | 必填 | 说明 |
|--------|------|------|
| `cli_json_file` | 是 | CLI 包的 `coverage-summary.json` 文件路径 |
| `core_json_file` | 是 | Core 包的 `coverage-summary.json` 文件路径 |
| `cli_full_text_summary_file` | 是 | CLI 包的 `full-text-summary.txt` 文件路径 |
| `core_full_text_summary_file` | 是 | Core 包的 `full-text-summary.txt` 文件路径 |
| `node_version` | 是 | Node.js 版本（用于消息上下文） |
| `os` | 是 | 操作系统（用于消息上下文） |
| `github_token` | 是 | 用于发布评论的 GitHub Token |

### 执行步骤

1. **Print Inputs** -- 打印所有输入参数用于调试。
2. **Prepare Coverage Comment** -- 使用 `jq` 从 JSON 文件提取覆盖率数据，生成 `coverage-comment.md` 文件，内容包括：
   - 覆盖率汇总表格（Lines / Statements / Functions / Branches）
   - CLI 包完整文本报告（可折叠）
   - Core 包完整文本报告（可折叠）
   - 引导用户查看详细 HTML 报告的 artifact 链接
3. **Post Coverage Comment** -- 使用第三方 Action 将生成的 Markdown 文件作为评论发布到 PR，使用 `code-coverage-summary` 标签确保评论可被更新而非重复创建。

## 依赖关系

- **`jq`** -- 用于从 JSON 覆盖率报告中提取数据。
- **`thollander/actions-comment-pull-request@v3`** -- 第三方 Action，用于在 PR 上创建或更新评论（使用 commit hash `65f9e5c9a1f2cd378bd74b2e057c9736982a8e74` 锁定版本）。
- **覆盖率报告文件** -- 依赖上游测试步骤生成的 `coverage-summary.json` 和 `full-text-summary.txt` 文件。
