---
name: writing-style-analyzer
description: >
  分析用户写作习惯 & 按用户风格改写文章。三大核心能力：
  1) 分析单篇：对比 AIGC 文章与用户修改版，提取写作风格（触发词："分析写作习惯"、"对比写作风格"、"帮我分析我改了哪些"）；
  2) 全量整理：用所有归档样本重新编译精准画像（触发词："重新整理画像"、"全量分析"、"compile profile"）；
  3) 改写文章：根据画像将 AI 文章改写成用户风格（触发词："用我的风格改写"、"改成我的风格"、"按我的写作习惯改"、"帮我改成我的风格"、"rewrite in my style"）。
  画像存储于 data/profile.md，代表性样本归档到 data/samples/。
agent_created: true
license: MIT
---

# Writing Style Analyzer

## Prerequisites: 版本追踪

### 环境检查

每次 skill 触发时检查 git 环境。若本次对话中已确认 git 可用，跳过检查。

1. 检查 git 环境：`git --version`
2. 若 `data/.git/` 不存在 → `git init data/` → 写入 `data/.gitignore`（内容：`*.tmp`）→ 初始 commit
3. 若 git 不可用 → 提示用户安装 git，本次运行跳过版本追踪（功能不受影响，只是无法回滚）

### Git Commit 约定

所有对 `data/` 的写入统一按以下格式自动 commit：

| 操作 | add 范围 | commit message |
|------|----------|----------------|
| 画像增量更新 | `git -C data add profile.md` | `"profile: 增量更新 #N"` |
| 画像全量重编译 | `git -C data add profile.md` | `"profile: 全量重编译 (N篇样本)"` |
| 样本归档 | `git -C data add samples/` | `"archive: YYYY-MM-DD-NN <标题>"` |
| 画像回滚 | `git -C data add profile.md` | `"rollback: 回滚至 <sha>"` |

**规则**：
- 始终使用精确路径 add（如 `profile.md`、`samples/`），**禁止** `add -A` 以免误提交临时文件
- commit message 前缀固定：`profile:` / `archive:` / `rollback:`
- 若 git 不可用（检查失败或 commit 执行报错），跳过 commit 并提示用户"本次写入未纳入版本追踪"

### 双层 Git 说明

本仓库有两层独立的 git：

| 层级 | 仓库位置 | 追踪内容 | 命令前缀 |
|------|----------|----------|----------|
| 外层 | 项目根目录 `.git/` | Skill 指令（SKILL.md、references/、assets/） | `git ...` |
| 内层 | `data/.git/` | 个人数据（profile.md、samples/） | `git -C data ...` |

外层 `.gitignore` 忽略了 `data/`，所以 `git status`（外层）看不到个人数据变更。查看画像历史需要用内层命令（见下方"常用 git 操作"）。

---

## Workflow Decision Tree

```
用户请求
├── 提供了两篇文档（AIGC + 修改版）
│   └── → 加载 references/analyze_single.md，走单篇分析流程
├── "重新整理画像" / "全量分析" / "compile profile"
│   └── → 加载 references/compile_profile.md，走全量重编译
├── 提供一篇 AI 文章 + 改写意图（"用我的风格改写" / "改成我的风格" / "rewrite in my style" 等）
│   └── → 加载 references/rewrite.md，走改写流程
├── "回滚画像" / "撤销上次分析" / "rollback profile"
│   └── → 执行下方「画像回滚」流程
└── 意图不明确
    └── → 追问：分析单篇？全量整理？改写文章？
```

---

## Storage

| 内容 | 位置 | 说明 |
|------|------|------|
| 写作习惯画像 | `data/profile.md` | 增量分析累积 / 全量重编译覆盖，每次写入自动 git commit |
| 代表性样本 | `data/samples/{YYYY-MM-DD}-{NN}/` | `aigc_original.md` + `user_modified.md` + `diff_summary.md`，NN 自动递增防同日覆盖 |
| 样本索引 | `data/samples/INDEX.md` | 表格，列：日期 \| 序号 \| 标题 \| 关键词 |

> `data/` 是独立 git 仓库，所有写入自动 commit，提供完整版本历史、diff 追溯和回滚能力。

---

## 画像回滚

当用户说"回滚画像" / "撤销上次分析" / "rollback profile"：

1. **展示历史**：`git -C data log --oneline -- data/profile.md`（最近 10 条）
2. **确认目标**：让用户选择要回滚到的 commit（sha 或序号），同时说明当前 HEAD
3. **执行回滚**：
   ```bash
   git -C data checkout <sha> -- profile.md
   git -C data add profile.md && git -C data commit -m "rollback: 回滚至 <sha>"
   ```
4. **同步样本**（如用户要求）：`git -C data checkout <sha> -- samples/`
5. **输出**：回滚前后的 commit sha、受影响的画像条目数量（新增/删除的 section 数）

> 回滚本身也被 git 追踪（产生新的 `rollback:` commit），所以即使回滚错误也可以再次回滚。

---

## 常用 Git 操作（用户手动使用）

用户可以在终端或对话中执行以下命令来管理画像历史：

```bash
# 查看画像修改历史
git -C data log --oneline -- data/profile.md

# 查看最近两次画像的差异
git -C data diff HEAD~1 -- data/profile.md

# 查看所有提交（含样本归档）
git -C data log --oneline

# 给当前画像状态打标签
git -C data tag stable-2026-07

# 对比两个版本的画像
git -C data diff <sha1> <sha2> -- data/profile.md

# 查看某个归档样本的原始内容
git -C data show <sha>:samples/2026-07-11-01/aigc_original.md
```

> 提醒用户：如果直接操作 `data/` 下的文件，需要手动 `git -C data commit`，否则下次 skill 写入时会自动提交未追踪的变更。

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

---

## 许可证

MIT License © 2026-present ttzc — 详见 [LICENSE](../LICENSE)。
