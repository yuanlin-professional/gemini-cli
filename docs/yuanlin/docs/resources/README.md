# docs/resources/ - 资源页面

## 概述

`docs/resources/` 目录包含 Gemini CLI 的辅助资源文档，包括常见问题解答、配额与定价说明、服务条款、故障排除指南和卸载说明。这些文档为用户提供使用过程中遇到问题时的参考。

## 目录结构

```
resources/
├── faq.md                  # 常见问题解答
├── quota-and-pricing.md    # 配额与定价说明
├── tos-privacy.md          # 服务条款与隐私政策
├── troubleshooting.md      # 故障排除指南
└── uninstall.md            # 卸载说明
```

## 核心组件

| 文档 | 描述 |
|------|------|
| `faq.md` | 常见问题及解答 |
| `quota-and-pricing.md` | API 调用配额限制和定价信息 |
| `tos-privacy.md` | 服务条款和隐私政策链接 |
| `troubleshooting.md` | 常见问题的诊断和解决步骤 |
| `uninstall.md` | 完整卸载 Gemini CLI 的步骤 |

## 依赖关系

### 内部引用

- 被 `docs/index.md` 作为支持资源引用
- `troubleshooting.md` 被其他文档在遇到问题时引用
