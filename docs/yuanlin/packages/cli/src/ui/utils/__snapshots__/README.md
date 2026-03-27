# __snapshots__ (cli/src/ui/utils)

## 概述

此目录存放 `packages/cli/src/ui/utils/` 下 UI 工具模块的测试快照文件。快照由测试框架自动生成，用于回归验证 UI 组件的渲染输出。涵盖代码高亮（CodeColorizer）、Markdown 展示（MarkdownDisplay）、表格渲染（TableRenderer）、边框样式（borderStyles）、终端设置（terminalSetup）以及文本输出（textOutput）等模块的快照。包含 `.snap`（文本快照）和 `.snap.svg`（SVG 视觉快照）两种格式。

## 目录结构

```
__snapshots__/
├── CodeColorizer-colorizeCode-does-not-let-colors-from-ansi-escape-codes-leak-into-colorized-code.snap.svg
├── CodeColorizer.test.tsx.snap
├── MarkdownDisplay.test.tsx.snap
├── TableRenderer-TableRenderer-calculates-column-widths-based-on-ren-.snap.svg
├── TableRenderer-TableRenderer-calculates-width-correctly-for-conten-.snap.svg
├── TableRenderer-TableRenderer-does-not-parse-markdown-inside-code-s-.snap.svg
├── TableRenderer-TableRenderer-handles-nested-markdown-styles-recurs-.snap.svg
├── TableRenderer-TableRenderer-handles-non-ASCII-characters-emojis-.snap.svg
├── TableRenderer-TableRenderer-handles-wrapped-bold-headers-without-showing-markers.snap.svg
├── TableRenderer-TableRenderer-renders-a-3x3-table-correctly.snap.svg
├── TableRenderer-TableRenderer-renders-a-complex-table-with-mixed-content-lengths-correctly.snap.svg
├── TableRenderer-TableRenderer-renders-a-table-with-long-headers-and-4-columns-correctly.snap.svg
├── TableRenderer-TableRenderer-renders-a-table-with-mixed-emojis-As-.snap.svg
├── TableRenderer-TableRenderer-renders-a-table-with-only-Asian-chara-.snap.svg
├── TableRenderer-TableRenderer-renders-a-table-with-only-emojis-and-.snap.svg
├── TableRenderer-TableRenderer-renders-complex-markdown-in-rows-and-.snap.svg
├── TableRenderer-TableRenderer-renders-correctly-when-headers-are-em-.snap.svg
├── TableRenderer-TableRenderer-renders-correctly-when-there-are-more-.snap.svg
├── TableRenderer-TableRenderer-strips-bold-markers-from-headers-and-renders-them-correctly.snap.svg
├── TableRenderer-TableRenderer-wraps-all-long-columns-correctly.snap.svg
├── TableRenderer-TableRenderer-wraps-columns-with-punctuation-correctly.snap.svg
├── TableRenderer-TableRenderer-wraps-long-cell-content-correctly.snap.svg
├── TableRenderer-TableRenderer-wraps-mixed-long-and-short-columns-correctly.snap.svg
├── TableRenderer.test.tsx.snap
├── borderStyles-MainContent-tool-group-border-SVG-snapshots-should-render-SVG-snapshot-for-a-pending-search-dialog-google_web_search-.snap.svg
├── borderStyles-MainContent-tool-group-border-SVG-snapshots-should-render-SVG-snapshot-for-a-shell-tool.snap.svg
├── borderStyles-MainContent-tool-group-border-SVG-snapshots-should-render-SVG-snapshot-for-an-empty-slice-following-a-search-tool.snap.svg
├── borderStyles.test.tsx.snap
├── terminalSetup.test.ts.snap
└── textOutput.test.ts.snap
```
