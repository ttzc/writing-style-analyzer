# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A **Claude Code Agent Skill** (not a typical application). It has no build system, no
dependencies, and no runnable code. It is a bundle of Markdown instructions that Claude itself
loads and executes when the skill triggers. "Running" the skill means Claude reads `SKILL.md`,
decides which workflow applies, and dynamically loads the relevant `references/*.md`.

The skill's purpose: analyze a user's writing habits by diffing AIGC-generated articles against
the user's edited versions, accumulate a style profile, and rewrite new AI text in the user's
voice. All skill content is in **Chinese**; match that when editing instructions or user-facing
strings.

## No Scripts — the LLM Is the Analyzer

There is no `scripts/` directory and no external tooling. All diff analysis is done by the LLM
directly: it reads both the AIGC original and the user-modified version side by side, identifies
pattern-level changes, and applies the five-dimension framework from `analysis_dimensions.md`.
Optionally, `diff -u` can be used for a quick macro overview, but it's not required.

This was a deliberate design simplification — an LLM reading two texts and comparing them is
more accurate and context-aware than a Python script doing mechanical string diff, especially
for Chinese text where word segmentation is non-trivial.

## Architecture: Progressive Disclosure

The skill is deliberately split so Claude loads only what the current task needs — never read
every reference up front. `SKILL.md` is the always-loaded entry point holding a **decision
tree** that routes to one of three workflows:

| User intent | Load these references | Update semantics |
|---|---|---|
| Two docs given (AIGC + edited) | `analyze_single.md` + `analysis_dimensions.md` | **Merge/append** into profile |
| "重新整理画像" / "全量分析" / "compile profile" | `compile_profile.md` + `analysis_dimensions.md` | **Overwrite** profile |
| One AI article + "改成我的风格" | `rewrite.md` | Read-only (consumes profile) |

`analysis_dimensions.md` is the shared analysis framework (five dimensions: 结构性 / 词汇 /
句式 / 语气与风格 / 内容操作) used by **both** analysis workflows — edit it once to change how
both analyze.

`assets/profile_template.md` is the skeleton for a brand-new profile (first analysis or full
rebuild).

## Data Model (runtime state, created on use)

Everything the skill "remembers" lives under `data/` (only `data/samples/` exists initially):

| Artifact | Path | Written by |
|---|---|---|
| Style profile | `data/profile.md` | Incremental append (analyze_single) OR full overwrite (compile_profile) |
| Archived samples | `data/samples/{YYYY-MM-DD}-{NN}/` → `aigc_original.md`, `user_modified.md`, `diff_summary.md` | The archive step (analyze_single Step 5), after user confirms the sample is representative. `NN` is auto-incremented per day. |
| Sample index | `data/samples/INDEX.md` | Appended one row per archived sample; columns: `日期 \| 序号 \| 标题 \| 关键词` |

`data/` is a standalone git repo (`data/.git/`). Every profile write and sample archive triggers an auto-commit, providing full version history, diff, and rollback. Git is checked at skill startup (SKILL.md Prerequisites); if unavailable, the skill runs without tracking but warns once.

The **confidence-tier system** is the core of the profile logic — every habit is tagged
初步 (1 sample) → 多次观察 (2–3, consistent) → 稳定 (4+, no conflict) → 待裁决 (samples
conflict). Incremental analysis raises/lowers tiers by merging; full recompile re-adjudicates
all tiers from scratch via cross-sample validation. `rewrite.md` only applies 稳定/多次观察
habits at full weight and down-weights 初步 ones. Preserve this vocabulary exactly when editing.

## Editing Conventions

- When adding a new workflow, wire it into the `SKILL.md` decision tree AND the Resources table —
  the tree is how Claude discovers it. Keep the tree and table in sync.
- Profile writes cap at ~3000 chars: `analyze_single.md` Step 3 compresses old 稳定 patterns
  into terse statements when exceeding that. Respect the cap when changing the profile format.
- The `SKILL.md` `description` frontmatter holds the trigger phrases that activate the skill —
  editing it changes when/whether the skill fires. Treat it as load-bearing, not documentation.
