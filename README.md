# Writing Style Analyzer

> Claude Code Agent Skill：分析用户写作习惯，对比 AI 生成文章与人工编辑版，累积风格画像，按用户风格改写 AI 文本。

[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)
[![Skill Type](https://img.shields.io/badge/type-Claude%20Code%20Skill-purple)](https://claude.ai/code)

---

## 这是什么？

一个运行在 **Claude Code** 中的 Agent Skill（纯 Markdown 指令集，无外部依赖）。它能：

1. **分析写作风格** — 对比一篇 AI 生成的文章和用户手动修改后的版本，提取用户的写作习惯
2. **累积风格画像** — 每次分析结果合并到画像中，随着样本增多，可信度自动升级
3. **按你的风格改写** — 拿到一篇新的 AI 文章，根据画像自动改写成你的风格

整个过程由 Claude 自己完成——没有脚本、没有后端、不需要 API Key（Skill 本身不调用任何外部服务）。

---

## 三大工作流

```
用户请求
├── 提供两篇文档（AIGC + 修改版）
│   └── 对比分析 → 提取习惯 → 更新画像
├── "重新整理画像" / "全量分析"
│   └── 读取所有归档样本 → 交叉验证 → 覆盖重写画像
├── 提供一篇 AI 文章 + "改成我的风格"
│   └── 读画像 → 逐层改写 → 输出结果 + 改写说明
├── "回滚画像"
│   └── 展示 git 历史 → 选择版本 → 回滚
```

### 工作流 1：分析单篇

**触发词**：`"分析写作习惯"`、`"对比写作风格"`、`"帮我分析我改了哪些"`

输入一对文章——AI 生成的原文 + 你手动修改后的版本。Claude 会从五大维度逐一对比，找出你的修改模式，然后累积更新到风格画像中。

详细流程见 [`references/analyze_single.md`](references/analyze_single.md)

### 工作流 2：全量重编译

**触发词**：`"重新整理画像"`、`"全量分析"`、`"compile profile"`

当你积累了多篇代表性样本后，对所有归档样本进行一次全量交叉验证，生成更精准的画像。适合画像中出现多处冲突标注、或觉得不够准时使用。

详细流程见 [`references/compile_profile.md`](references/compile_profile.md)

### 工作流 3：按风格改写

**触发词**：`"用我的风格改写"`、`"改成我的风格"`、`"rewrite in my style"`

输入一篇新的 AI 文章，Claude 读取你的风格画像，按五层顺序逐层改写：

| 层序 | 改写内容 | 示例 |
|:---:|:---|:---|
| Layer 1 | **内容操作** | 删除冗余表述、添加个人观点、补充具体例子 |
| Layer 2 | **结构性** | 调整段落/句子长度、标题层级、列表风格 |
| Layer 3 | **词汇** | 替换 AI 高频词为你的偏好用词 |
| Layer 4 | **句式** | 调整主动/被动语态、连接词密度、句复杂度 |
| Layer 5 | **语气与风格** | 调整正式程度、情感词密度、人称使用 |

每条改写操作都有可追溯的画像依据，不确定的操作标注 `[推测]`。没有画像支撑的维度宁可保留原文，不强行修改。

详细流程见 [`references/rewrite.md`](references/rewrite.md)

---

## 分析框架：五大维度

所有分析工作流共享同一套分析框架（[`references/analysis_dimensions.md`](references/analysis_dimensions.md)）：

| 维度 | 关注点 |
|:---|:---|
| **一、结构性** | 段落长度偏好、句子长度偏好、标题层级习惯、列表 vs 段落、结构重组模式 |
| **二、词汇** | 高频用词替换（AI→用户）、专业术语密度、口语/书面语倾向、修饰语使用 |
| **三、句式** | 主动/被动语态比例、连接词偏好、句式复杂度、句首多样性 |
| **四、语气与风格** | 正式程度、情感表达方式、人称使用习惯（第一/二/三人称）、标点与语气词 |
| **五、内容操作** | 倾向添加什么（例子/数据/观点）、倾向删除什么（冗余/客套/AI 模板用语） |

每个维度的分析输出包含：观察到的具体模式 + 量化差异数据 + 可信度标注。

---

## 可信度分级系统

画像中每条习惯都有可信度标注，随着样本积累自动升级：

```
初步 ──→ 多次观察 ──→ 稳定
 (1)       (2-3)       (4+)
```

| 标注 | 条件 | 改写时权重 |
|:---|:---|:---|
| **初步** | 仅基于 1 次样本观察 | 降低权重，仅供参考 |
| **多次观察** | 基于 2-3 次样本，方向一致 | 正常权重 |
| **稳定** | 基于 4+ 次样本，方向一致，无矛盾 | 完全信任 |
| **待裁决** | 不同样本间观察到矛盾模式 | 暂停使用，等待更多样本 |

全量重编译（工作流 2）会跨所有样本重新判定每条习惯的可信度，确保画像精度随样本增加而提高。

---

## 存储结构

```text
data/                          # 个人数据（独立 git 仓库，不纳入主仓库）
├── profile.md                 # 写作习惯画像（每次分析后自动更新）
├── samples/
│   ├── INDEX.md               # 归档样本索引
│   ├── 2026-07-11-01/          # 每日归档，NN 自动递增
│   │   ├── aigc_original.md   # AI 生成的原文
│   │   ├── user_modified.md   # 用户修改后的版本
│   │   └── diff_summary.md    # 差异分析摘要
│   └── 2026-07-13-01/
│       └── ...
└── .git/                      # data/ 的独立版本历史
```

- **画像** (`profile.md`)：增量分析时合并追加，全量重编译时覆盖重写，每次写入自动 git commit
- **样本** (`samples/`)：代表性对比存档，每次分析后可选择是否归档（满足 ≥3/4 条件时推荐归档）
- **索引** (`INDEX.md`)：表格形式记录所有归档样本的日期、序号、标题、关键词

---

## Git 版本追踪

`data/` 是一个**独立的 git 仓库**，所有画像和样本的写入自动 commit，提供完整的版本历史。

### 双层 Git 设计

| 层级 | 仓库 | 追踪内容 | 示例命令 |
|:---|:---|:---|:---|
| 外层 | 项目根目录 `.git/` | Skill 指令（SKILL.md、references/、assets/） | `git log` |
| 内层 | `data/.git/` | 个人数据（profile.md、samples/） | `git -C data log` |

外层 `.gitignore` 忽略了 `data/`，所以你的个人数据不会出现在主仓库中，也不会被推送到 GitHub。

### 自动 Commit 约定

| 操作 | Commit Message |
|:---|:---|
| 画像增量更新 | `profile: 增量更新 #N` |
| 画像全量重编译 | `profile: 全量重编译 (N篇样本)` |
| 样本归档 | `archive: YYYY-MM-DD-NN <标题>` |
| 画像回滚 | `rollback: 回滚至 <sha>` |

### 常用手动操作

```bash
# 查看画像修改历史
git -C data log --oneline -- data/profile.md

# 查看最近两次分析间的画像差异
git -C data diff HEAD~1 -- data/profile.md

# 给当前画像打标签（如赛前稳定版本）
git -C data tag stable-2026-07

# 查看某个归档样本
git -C data show <sha>:samples/2026-07-11-01/aigc_original.md
```

回滚功能也内置于 Skill 中——对 Claude 说 `"回滚画像"` 即可。

---

## 文件结构

```text
writing-style-analyzer/
├── SKILL.md                          # Skill 入口：触发词 + 决策树 + 约定
├── README.md                         # 本文件
├── CLAUDE.md                         # 开发者文档（给 Claude 自己看的）
├── .gitignore                        # 排除 data/、本地配置
│
├── assets/
│   └── profile_template.md           # 画像骨架模板（首次分析或全量重建用）
│
├── references/                       # 按需加载的工作流指令
│   ├── analysis_dimensions.md        # 五大维度分析框架（共享）
│   ├── analyze_single.md             # 工作流 1：单篇分析
│   ├── compile_profile.md            # 工作流 2：全量重编译
│   └── rewrite.md                    # 工作流 3：按风格改写
│
└── data/                             # 运行时生成（gitignore，有独立 git）
    ├── profile.md
    ├── samples/
    │   ├── INDEX.md
    │   └── YYYY-MM-DD-NN/
    └── .git/
```

---

## 快速开始

### 前置条件

- **Claude Code**（CLI、桌面应用、Web 版或 IDE 插件均可）
- **Git**（可选，用于版本追踪；没有也能正常使用，只是无法回滚）

### 安装

将本仓库克隆到 Claude Code 能访问的位置，然后注册为 Skill：

```bash
# 克隆仓库
git clone https://github.com/ttzc/writing-style-analyzer.git

# 在 Claude Code 中注册 Skill
claude skills add writing-style-analyzer /path/to/writing-style-analyzer
```

或者直接将整个目录放入 Claude Code 的 skills 目录中。

### 第一次使用

1. **分析你的修改习惯**——准备一篇 AI 写的文章 + 你手动改过的版本，粘贴给 Claude：

   ```
   【AIGC 原文】
   <AI 生成的原文>

   【用户修改版】
   <你修改后的版本>
   ```

   或者提供两个文件路径。Claude 会分析你的修改模式并生成第一版画像。

2. **重复 3-5 次**——每次分析后确认归档代表性样本，画像的可信度会从"初步"逐步升级到"稳定"。

3. **开始改写**——有了稳定的画像后，拿到新的 AI 文章直接说：

   ```
   用我的风格改写这篇文章：
   <AI 生成的新文章>
   ```

   Claude 会逐层应用你的习惯，输出改写结果和可追溯的改写说明。

---

## 设计理念

- **LLM 即分析器**：不依赖外部 NLP 工具或脚本。Claude 直接阅读两篇文章做对比分析，这比机械式字符串 diff 更准确——尤其是对中文文本（无需分词）
- **渐进式加载**：SKILL.md 是入口，根据用户意图只加载当前工作流需要的 reference 文件，避免上下文浪费
- **画像可追溯**：每条改写操作都能溯源到画像中的具体条目和可信度层级
- **数据主权**：你的写作习惯数据完全在本地 `data/` 目录中，由独立的 git 仓库保护，不会出现在主仓库中

---

## 许可证

本项目采用 [MIT License](LICENSE)。

Copyright © 2026-present ttzc — 允许自由使用、复制、修改、合并、发布、分发、再许可和/或销售本软件的副本，唯须在所有副本或实质性部分中包含上述版权声明和本许可声明。

---

## 相关资源

- [SKILL.md](SKILL.md) — Skill 入口与完整技术细节
- [CLAUDE.md](CLAUDE.md) — 架构设计文档
- [analysis_dimensions.md](references/analysis_dimensions.md) — 五大维度分析框架
- [analyze_single.md](references/analyze_single.md) — 单篇分析工作流
- [compile_profile.md](references/compile_profile.md) — 全量重编译工作流
- [rewrite.md](references/rewrite.md) — 按风格改写工作流
