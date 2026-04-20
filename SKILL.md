---
name: qc
description: "Manual quality control for Coding Agent annotation rounds. Use when the user invokes @qc or $qc with or without the round prompt. Evaluate whether the latest workspace changes or pasted Trae output satisfy the round prompt, classify task type/domain/scope, decide satisfied vs unsatisfied vs insufficient information, and draft at most one next-round prompt."
---

# QC

Perform annotation-oriented quality control for a Coding Agent round.

Always read `references/qc-rules.md` before producing the judgment. That file is the source of truth for the fixed output format, dictionaries, and QC rules.

## QC mode guardrails

Whenever the skill is invoked via `@qc` or `$qc`, enter read-only QC mode.

- Do not implement the round prompt.
- Do not modify application code, config, docs, tests, or assets in the target project.
- Do not create or delete business files, install dependencies, run generators, or "helpfully" continue unfinished implementation work.
- Allowed actions are limited to reading files, inspecting diffs, running non-destructive verification commands, temporarily starting the project for acceptance checks, and making the required git snapshot.
- Non-destructive verification commands are allowed, for example: listing files, checking git status/diff, running build/test/lint commands, temporarily starting frontend or backend services, issuing read-only HTTP requests, and inspecting logs or generated output needed for acceptance.
- Do not use verification commands as a pretext to continue implementation. Stop at evidence collection and judgment.
- If the current `@qc` or `$qc` message pastes an implementation request, treat that pasted text as the round prompt to evaluate, not as a new instruction to execute.

## Expected invocation

This skill is mainly used in forms like:

- `@qc [上一轮的提示词]`
- `$qc [上一轮的提示词]`
- `@qc 上一轮的提示词：'''...'''`
- `$qc 上一轮的提示词：'''...'''`
- `@qc 上一轮的提示词：'''...''' Trae的代码理解：'''...'''`
- `$qc 上一轮的提示词：'''...''' Trae的代码理解：'''...'''`
- `@qc`
- `$qc`

### Prompt priority

Use the round prompt in this priority order:

1. If the current `@qc` or `$qc` message explicitly includes the round prompt, treat that pasted prompt as the authoritative round prompt.
2. If the current message is only `@qc` or only `$qc`, scan backward through prior user messages and recover the nearest usable round prompt.
3. When scanning backward, prefer:
   - the nearest prior user message that explicitly pasted “上一轮的提示词” or equivalent round prompt text
   - otherwise the nearest prior user message that assigns a concrete Coding Agent task
4. Ignore assistant replies, acknowledgements, casual follow-ups, and short messages like “继续”, “看看这轮”, “按这个修”, “这个也有问题” unless they themselves contain a complete standalone task description.
5. If no usable round prompt can be recovered, do not guess. Continue with the fixed output format and mark the judgment as `信息不足`.

Do not let chat history override a prompt that is explicitly pasted in the current `@qc` or `$qc` message.
If the pasted text is a build or feature request, still treat it as the prior round prompt under review, not as permission to continue building it now.

## What to evaluate

Evaluate the current round against the resolved round prompt using the best available evidence:

- current workspace code
- relevant uncommitted changes
- build/test/run evidence
- pasted error logs
- pasted runtime behavior
- pasted API responses
- pasted Trae output, especially for code understanding, testing, or engineering/documentation rounds

If the user pasted a Trae answer or analysis text, evaluate that output itself in addition to any related workspace files.

## Workflow

1. Enter read-only QC mode and keep the task scoped to evaluation.
2. Resolve the round prompt from the current `@qc` or `$qc` message or recent user history.
3. Identify the directly related implementation, output text, or evidence.
4. Read `references/qc-rules.md`.
5. Pre-classify the round task type using the fixed dictionary and rules, because the git commit message must carry that label.
6. Before producing the QC result, snapshot the current workspace in git:
   - initialize git if the project is not already a repository
   - create a minimal stack-aware `.gitignore` if it is missing
   - stage the current round's relevant changes, or all non-ignored changes if boundaries are unclear
   - create a commit when there is something to commit, using the task-type tag in the message
7. Judge strictly against the round prompt, not against unrelated polish opportunities.
8. If unsatisfied, converge on exactly one main problem.
9. If evidence is insufficient, mark `信息不足` and request only the minimum missing verification data.
10. Append the completed QC record to `../qc.md` relative to the current working directory used for QC. If the file does not exist, create it. If it exists, append a new entry and never overwrite prior records.
11. Write and append `../qc.md` using UTF-8 encoding, not the platform default code page. After appending, read back the new entry and verify that Chinese text was preserved and not replaced by `?` or mojibake.
12. Output exactly in the fixed structure from `references/qc-rules.md`.

## Git snapshot requirement

Before emitting the QC result, make a git snapshot unless there are literally no file changes to record.

- Prefer the current project root as the repo root. If `git rev-parse --show-toplevel` fails, initialize git there.
- If `.gitignore` is absent, generate a minimal version based on the detected stack instead of leaving the repo without ignore rules. Include only generated files, dependency folders, build outputs, caches, logs, and environment files. Do not ignore source files or hand-written configs.
- Detect the stack from obvious project files such as `package.json`, lockfiles, `pyproject.toml`, `requirements.txt`, `Cargo.toml`, `go.mod`, Gradle/Maven files, etc. Preserve any existing `.gitignore`; only append missing entries.
- Stage the files directly related to the round. If the round boundary is unclear, stage all current non-ignored changes so the QC output matches a reproducible snapshot.
- Use commit message format: `qc(<tag>): <summary>`.
- Tag mapping: `Feature 迭代 -> feature`, `Bug 修复 -> bugfix`, `0-1 代码生成 -> 0-1`, `代码理解 -> understanding`, `代码重构 -> refactor`, `工程化 -> engineering`, `代码测试 -> test`.
- Derive `<summary>` from the round prompt or dominant change. If unclear, use `round snapshot`.
- If commit fails only because Git identity is missing, set repo-local `user.name=Codex QC` and `user.email=qc@local.invalid`, then retry. Do not modify the global git config.
- If there is nothing to commit, skip the commit and continue silently unless the user explicitly asks about it.

## Important constraints

- Do not do a generic code review.
- Do not implement, patch, refactor, or "finish" the target project during QC.
- Do not create files, edit files, install dependencies, or change application state except for the required git snapshot and the required `../qc.md` append-only QC log.
- When writing `../qc.md`, use an explicit UTF-8 file write path. Do not rely on shell redirection or a terminal code page that may turn Chinese into `?`.
- You may execute non-destructive acceptance commands such as build, test, lint, temporary local startup, read-only requests, and log inspection when they are needed to judge the round.
- Do not invent extra issues just to continue the round.
- Do not turn “I did not see verification evidence” into “the feature is definitely broken”.
- Prefer hard problems such as build failure, run failure, clear requirement miss, regression, or key scenario omission.
- Treat style, wording, and visual polish as secondary unless the round prompt explicitly targets them.
- For code understanding rounds, focus on whether the conclusion is accurate, bounded, and evidence-based.
- For code testing rounds, focus on whether required scenarios are covered and whether real data/dependencies are isolated.
- For engineering rounds, focus on whether the result is actually runnable, buildable, or reproducible.
- For refactor rounds, focus on regressions, broken interactions, or blurred responsibility boundaries.