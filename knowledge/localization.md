# Localization — running the interview and writing the blueprint in the user's language

> The Architect detects the user's language from their first message and stays in it. This file is
> the contract for what that actually means: what changes, what must never change, and the
> Simplified-Chinese specifics that the English-only rules silently miss.

Last verified: 2026-08-10

Read this in DISCOVERY the moment the user's first message is not English, and again before
GENERATE. `blueprint-writer` and `blueprint-validator` both work against the boundary defined here.

---

## The one-line rule

**Everything a human reads follows the user's language. Everything a machine reads stays ASCII
English.** A blueprint is both documents at once, and the boundary between them is not a style
preference — cross it and the build breaks.

## The translation boundary

| Follows the user's language | Stays English, always |
|---|---|
| Interview prose, questions, recommendations | Code, identifiers, type names, function names |
| Section heading text (the number and English name survive — see below) | File and directory names, every path |
| Every "Why" / rationale cell | Env var names and their values |
| Acceptance-criteria prose — the observable part | CLI commands, package names, scripts |
| Non-goals, risks, tradeoffs, open questions | EARS keywords `WHEN` / `THE SYSTEM SHALL` |
| Epic narrative, phase descriptions | `tasks.json` keys, `id`, `epic`, `files[]`, `verify[]` |
| Prose sections of the generated `CLAUDE.md` | `status` values — `pending` / `in_progress` / `done` |
| Handoff message, build-order narrative | Git tag names, branch names, commit subjects |
| The question *inside* a clarification marker | The `[NEEDS CLARIFICATION: …]` token itself |
| — | `NOT APPLICABLE — <reason>` (the token; the reason follows the user) |
| — | `{placeholder}` tokens, `§N` references, section numbers |

**Why the right column is absolute.** `blueprint-validator` decides findings 2, 8, 9, 11 and 27 by
matching literal strings. `/architect-next` selects the next task by comparing `status` to
`"pending"`. `git tag step-07-auth` is a ref name. Translate any of these and the failure is silent:
the validator reports PASS on a blueprint whose gates cannot be evaluated.

## Numbered headings — keep the number, keep the English name

The template defines 20 numbered sections and finding 11 requires every one of them to be present.
Match is by **number**; the English name is what `/architect-audit` and any downstream tooling read.

```
Good:  ## 1. 概述 (Overview)
Good:  ## 9. 构建顺序 (Build Order)
Good:  ### 9.1 迁移与切换 (Migration & Cutover)
Bad:   ## 概述                 ← number gone; audit reports 20 missing sections
Bad:   ## 1. 概述              ← English name gone; name-based cross-references dangle
```

Cross-references inside the document use the number — `见 §7`, not `见"数据模型"一节`.

## Slugs, paths and refs are ASCII — no exceptions

The project slug is lowercase kebab-case ASCII, and it is the name of a directory, a git tag prefix,
and often a package name. CJK characters in any of those break `git tag` on some configurations, are
rejected by npm and PyPI naming rules, and mangle under a non-UTF-8 Windows console code page.

Derive the slug in this order, and **ask once in DISCOVERY** rather than guessing:

1. An English or ASCII short name the user already uses (`智能巡检系统 / SmartPatrol` → `smartpatrol`).
2. Ask: *"这个项目有英文短名吗？没有的话我用拼音生成目录名。"*
3. Fall back to toneless Hanyu Pinyin, hyphenated at word boundaries: `智能巡检` → `zhineng-xunjian`.

The same rule governs directory names in §3, table names, column names, API routes, and every
`files[]` entry. Chinese belongs in the prose describing them, never in the identifier.

## Acceptance criteria in Simplified Chinese

EARS keywords stay uppercase English; everything between them is Chinese. The keyword is the part a
grep finds, the Chinese is the part a human checks.

| | |
|---|---|
| Good | WHEN `POST /api/invoices` 收到缺少 `amount` 字段的请求体，THE SYSTEM SHALL 返回 422 且响应体为 `{ error: "amount is required" }` |
| Good | WHEN `pnpm test:e2e` 运行，THE SYSTEM SHALL 退出码为 0，且 `playwright-report/` 下生成报告 |
| Good | WHEN 数据库迁移执行完毕，THE SYSTEM SHALL 使 `\dt` 列出 7 张表，含 `users`、`sessions` |
| Bad | WHEN 用户登录，THE SYSTEM SHALL 正常工作 |
| Bad | 登录功能实现完成即可 |

## Vague-word blacklist — Simplified Chinese

Finding 2 ("an acceptance criterion that is not observable") ships an English blacklist only. These
are its Chinese counterparts. Inside a *Done when*, an `acceptance[]` entry, or a §20.1 gate line,
each one is the **same BLOCKER** as its English twin:

> 正常工作 · 工作正常 · 正常运行 · 运行正常 · 能正常使用 · 可以用了 · 没问题 · 无异常 ·
> 看起来正常 · 显示正确 · 展示无误 · 正确处理 · 妥善处理 · 合理处理 · 正确实现 · 已实现 ·
> 实现完成 · 开发完成 · 接入完成 · 对接好 · 配置好 · 符合预期 · 达到预期 · 体验流畅 ·
> 性能良好 · 稳定运行 · 兼容性良好 · 用户满意

Hedges are the same defect wearing politeness: **应该 · 大概 · 尽量 · 尽可能 · 原则上 · 基本上**
in a criterion mean nobody can decide whether the step is done.

The test is unchanged and language-independent: *could two people disagree about whether this is
done?* If the criterion names no command, no exit code, no HTTP status, no row count and no file
path, it fails — in any language.

## CJK mechanics

- **Encoding: UTF-8, no BOM.** A BOM breaks shebangs, `.env` parsing, and several JSON parsers. On
  Windows this is the single most common way a generated `workspace/` file fails its own first gate.
- **Punctuation:** full-width （），。、；：「」 in Chinese prose; ASCII punctuation inside code
  spans, commands, paths and JSON — never `｛｝` or `：` in anything a machine parses.
- **Spacing:** one space between Chinese and adjacent Latin/numeric runs (`用 TypeScript 编写`、`部署到 Vercel 上`).
  No space inside `` `code` `` spans.
- **Line endings** follow the target repo, not this plugin. On Windows check `core.autocrlf` before
  asserting a byte-exact fixture — finding 35 is reachable through line endings alone.
- **Traditional Chinese:** if the user writes 繁體, mirror it. Never convert 繁 → 简 or the reverse,
  and never mix the two in one document.

## Terminology — pick one word and keep it

Two runs of this tool must not call the same thing two things. Chinese renderings are fixed here so
the interview, the blueprint and the epics agree.

| EN | 简体中文 | EN | 简体中文 |
|---|---|---|---|
| shape | 项目形态 | acceptance criteria | 验收标准 |
| runtime track | 运行时技术栈 | verify command | 校验命令 |
| capability | 能力域 | checkpoint | 检查点 |
| blueprint | 蓝图 | gate | 门禁 |
| bundle | 蓝图包 | non-goals | 非目标 |
| build step | 构建步骤 | scope fence | 范围边界 |
| build order | 构建顺序 | greenfield | 全新项目 |
| epic | 任务群 | brownfield | 存量代码 |
| task DAG | 任务依赖图 | cutover | 切换上线 |
| version pin | 版本锁定 | rollback | 回滚 |
| provenance | 来源出处 | smoke test | 冒烟测试 |
| `[NEEDS CLARIFICATION]` | 待澄清标记（标记本身不翻译） | handoff | 交接 |

## Simplified-Chinese entry phrases

These route the same way their English and Spanish equivalents do. They are mirrored in the skill
description, which is what actually decides activation.

| Says | Route to |
|---|---|
| 帮我设计架构 · 设计一下这个应用 · 我想做一个 App · 技术选型建议 · 用什么技术栈 · 出一份技术方案 · 写个 PRD · 生成蓝图 · 规划这个项目 · 怎么搭这个项目 | `/architect` |
| 快速出方案 · 直接开做 · 别问了直接生成 · 给个默认方案 | `/architect-quick` |
| 分析我现有的代码 · 给这个仓库加个功能 · 重构这个项目 · 迁移到 X · 给现有项目出方案 | `/architect-brownfield` |
| 继续构建 · 下一步做什么 · 接着上次的进度 | `/architect-next` |
| 版本还是最新的吗 · 刷新一下依赖版本 | `/architect-refresh` |
| 检查一下这份蓝图 · 蓝图有没有问题 | `/architect-audit` |

## Failure modes actually seen

| What happened | Why it is invisible |
|---|---|
| `git tag step-03-用户认证` | Fails only on the builder's machine, at step 3, hours in |
| `"status": "待办"` in `tasks.json` | `/architect-next` finds zero unblocked tasks and reports the build finished |
| `当 <触发> 系统应当 <结果>` | Validator's EARS grep matches nothing; every criterion passes unchecked |
| `## 概述` with the number dropped | Finding 11 fires 20 times, or — worse — the audit is run by name and fires zero |
| `[需要澄清: 用哪家支付?]` | Finding 9 never fires; the marker ships inside the delivered blueprint |
| UTF-8 BOM on an emitted `.env.example` | §10 Bootstrap's first command reads a garbled first key name |

## See also

- `knowledge/skills-registry.md` — skill names and fallbacks, unaffected by language
- `templates/blueprint-template.md` — §Language rule the writer follows
- `agents/blueprint-validator.md` — findings 2, 8, 9, 11, 27, 35 all depend on this boundary
