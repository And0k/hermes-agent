---
title: "Command Code — Delegate coding to Command Code CLI (Go plan, headless)"
sidebar_label: "Command Code"
description: "Delegate coding to Command Code CLI (Go plan, headless)"
---

{/* This page is auto-generated from the skill's SKILL.md by website/scripts/generate-skill-docs.py. Edit the source SKILL.md, not this page. */}

# Command Code

Delegate coding to Command Code CLI (Go plan, headless).

## Skill metadata

| | |
|---|---|
| Source | Bundled (installed by default) |
| Path | `skills/autonomous-ai-agents/command-code` |
| Version | `1.0.0` |
| Author | Andrey Korzh, Hermes Agent |
| License | MIT |
| Platforms | linux, macos, windows |
| Tags | `Coding-Agent`, `Command-Code`, `Autonomous`, `Delegation`, `Go-Plan` |
| Related skills | [`opencode`](../../bundled/autonomous-ai-agents/autonomous-ai-agents-opencode.md), [`claude-code`](../../bundled/autonomous-ai-agents/autonomous-ai-agents-claude-code.md), [`codex`](../../bundled/autonomous-ai-agents/autonomous-ai-agents-codex.md), [`hermes-agent`](../../bundled/autonomous-ai-agents/autonomous-ai-agents-hermes-agent.md) |

## Reference: full SKILL.md

:::info
The following is the complete skill definition that Hermes loads when this skill is triggered. This is what the agent sees as instructions when the skill is active.
:::

# Command Code CLI

Use [Command Code](https://commandcode.ai) as an autonomous coding worker orchestrated by Hermes terminal/process tools. On the **Go plan** ($1/month) there is **no Provider API** — only the interactive CLI with browser OAuth login. Headless mode (`cmd -p`) reuses that login and outputs machine-readable JSON.

## When to Use

- User explicitly asks to use Command Code / `cmd`
- Go-plan models (DeepSeek V4, Qwen 3.x, Kimi, GLM, MiniMax, Mistral, etc.) are wanted and opencode credits are exhausted
- Long-running coding sessions with progress checks
- Parallel task execution in isolated workdirs/worktrees

## Prerequisites

- Command Code installed: `npm i -g command-code` (provides `cmd` and `cmdc`)
- Auth configured once in a real terminal: `cmd login` (browser OAuth). Headless runs reuse `~/.commandcode/auth.json`.
- Verify: `cmd status` should print `Authenticated as ...`
- Git repository for code tasks (recommended)

## One-Shot Tasks (Headless)

Use `cmd -p` for bounded, non-interactive tasks. Add `--yolo` to allow file writes and shell commands (otherwise blocked in headless mode). Add `--output-format json` for machine-readable NDJSON:

```
terminal(command="cmd -p 'Add retry logic to API calls and update tests' --yolo --output-format json", workdir="~/project")
```

Force a specific model:

```
terminal(command="cmd -p 'Refactor auth module' -m deepseek/deepseek-v4-flash --yolo --output-format json", workdir="~/project")
```

## Interactive Sessions (Background)

For iterative work requiring multiple exchanges, start the TUI in background:

```
terminal(command="cmd", workdir="~/project", background=true, pty=true)
# Returns session_id

# Send a prompt
process(action="submit", session_id="<id>", data="Implement OAuth refresh flow and add tests")

# Monitor progress
process(action="poll", session_id="<id>")
process(action="log", session_id="<id>")

# Exit cleanly — Ctrl+C
process(action="write", session_id="<id>", data="\x03")
```

**Important:** `cmd login` and the TUI require a real TTY/browser; Hermes cannot drive the Ink UI. Do the one-time login in an external terminal.

## JSON Output

`--output-format json` produces NDJSON. The final line is always:

```json
{"type":"result","subtype":"success","sessionId":"...","stopReason":"end_turn","usage":{"inputTokens":...,"outputTokens":...,"cacheReadTokens":...,"cacheWriteTokens":0},"durationMs":...,"finalText":"..."}
```

On failure: `subtype` is `error`, `max_turns`, etc. `sessionId` and `stopReason` may be omitted on early failures.

Exit codes:

| Code | Meaning |
|------|---------|
| 0 | Success |
| 3 | Not authenticated |
| 4 | Permission denied |
| 5 | Rate limit exceeded |
| 8 | Max turns reached |
| 10 | Insufficient credits |

## Sessions & Resuming

Each headless run persists its transcript. Continue the most recent run in a directory:

```
terminal(command="cmd -p --continue 'now suggest a fix' --yolo --output-format json", workdir="~/project")
```

Resume a specific session by id (printed to stderr with `--verbose`):

```
terminal(command="cmd -p --resume <session-id> 'continue the review' --yolo --output-format json", workdir="~/project")
```

## Permissions

Headless mode **blocks** file writes, edits, and shell commands by default. Pass `--yolo` (alias `--dangerously-skip-permissions`) to enable them. `--permission-mode yolo` does **not** lift the block in headless mode.

## Model Selection

Use `-m <model>` to force a model. Go-plan models include DeepSeek V4, Qwen 3.x, Kimi, GLM, MiniMax, Mistral, StepFun, Tencent, Xiaomi, NVIDIA, etc. Default model is `deepseek/deepseek-v4-flash`.

## Sandbox & Permissions

- A shell command touching a path **outside the working directory** is auto-rejected by the sandbox and can end the run with exit 0 and no work done. Scope the brief to the repository; put all outside facts (ports, endpoints, versions) inside the prompt.
- `/tmp` is an outside path too — use repo-local directories.
- Never design a fixture that reads a system binary.

## Procedure

1. Verify tool readiness:
   - `terminal(command="cmd status")` — should show `Authenticated as ...`
   - `terminal(command="cmd --version")`
2. For bounded tasks, use `cmd -p '...' --yolo --output-format json`.
3. For iterative tasks, start `cmd` with `background=true, pty=true`.
4. Monitor long tasks with `process(action="poll"|"log")`.
5. Exit interactive sessions with Ctrl+C (`\x03`) or kill, never `/exit`.
6. Summarize file changes, test results, and next steps back to user.

## Verification

Smoke test:

```
terminal(command="cmd -p 'Respond with exactly: CMD_SMOKE_OK' --output-format json")
```

Success criteria:

- Output includes `"finalText":"CMD_SMOKE_OK"`
- Command exits 0 without provider/model errors
- For code tasks: expected files changed and tests pass

## Rules

1. Prefer `cmd -p` for one-shot automation — simpler, no pty needed.
2. Use interactive background mode only when iteration is needed.
3. Always scope Command Code sessions to a single repo/workdir.
4. For long tasks, provide progress updates from process logs.
5. Report concrete outcomes (files changed, tests, remaining risks).
6. Exit interactive sessions with Ctrl+C or kill, never `/exit`.
7. **Fallback:** when Go-plan credits are exhausted (exit code 10 / `EXIT_INSUFFICIENT_CREDITS`), switch back to `opencode` (skill `opencode`) or `pi` until the next billing cycle.
