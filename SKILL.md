---
name: writing-style-analyzer
description: >
  分析用户写作习惯 & 按用户风格改写文章。三大核心能力：
  1) 分析单篇：对比 AIGC 文章与用户修改版，提取写作风格（触发词："分析写作习惯"、"对比写作风格"、"帮我分析我改了哪些"）；
  2) 全量整理：用所有归档样本重新编译精准画像（触发词："重新整理画像"、"全量分析"、"compile profile"）；
  3) 改写文章：根据画像将 AI 文章改写成用户风格（触发词："用我的风格改写"、"改成我的风格"、"按我的写作习惯改"、"帮我改成我的风格"、"rewrite in my style"）。
  画像存储于 data/profile.md，代表性样本归档到 data/samples/。
agent_created: true
---

# Writing Style Analyzer

## Prerequisites: 版本追踪

`data/` 目录由 git 追踪。每次触发 skill 时：

1. 检查 git 环境：`git --version`
2. 若 `data/.git/` 不存在 → `git init data/` → 写入 `data/.gitignore`（内容：`*.tmp`）→ 初始 commit
3. 若 git 不可用 → 提示用户安装 git，本次运行跳过版本追踪（功能不受影响，只是无法回滚）
4. 每次写入 profile.md 或归档样本后，自动 git commit

**回滚画像**：当用户说"回滚画像"，执行 `git -C data log --oneline` 展示提交历史，让用户选择目标 commit，然后 `git -C data checkout <sha> -- profile.md`。

## Workflow Decision Tree

```
用户请求
├── 提供了两篇文档（AIGC + 修改版）
│   └── → 加载 references/analyze_single.md，走单篇分析流程
├── "重新整理画像" / "全量分析" / "compile profile"
│   └── → 加载 references/compile_profile.md，走全量重编译
├── 提供一篇 AI 文章 + 改写意图（"用我的风格改写" / "改成我的风格" / "rewrite in my style" 等）
│   └── → 加载 references/rewrite.md，走改写流程
└── 意图不明确
    └── → 追问：分析单篇？全量整理？改写文章？
```

## Storage

| 内容 | 位置 | 说明 |
|------|------|------|
| 写作习惯画像 | `data/profile.md` | 增量分析累积 / 全量重编译覆盖，每次写入自动 git commit |
| 代表性样本 | `data/samples/{YYYY-MM-DD}-{NN}/` | `aigc_original.md` + `user_modified.md` + `diff_summary.md`，NN 自动递增防同日覆盖 |
| 样本索引 | `data/samples/INDEX.md` | 表格，列：日期 \| 序号 \| 标题 \| 关键词 |

> `data/` 是独立 git 仓库，所有写入自动 commit，提供完整版本历史、diff 追溯和回滚能力。

## Resources

按需动态加载，每次只加载当前任务需要的 reference：

| 任务 | 加载文件 |
|------|----------|
| 分析单篇 | `references/analyze_single.md` + `references/analysis_dimensions.md` |
| 全量整理 | `references/compile_profile.md` + `references/analysis_dimensions.md` |
| 改写文章 | `references/rewrite.md` |

### assets/profile_template.md
新画像初始化模板。首次分析或全量重建时使用。

分析由 LLM 直接完成，无外部脚本依赖。
