---
name: aider-pair-programming
description: Delegate code edits to Aider with atomic git auto-commits.
version: 1.0.0
author: Mahyuddin bin Ab Rasid (idinUPSI), Hermes Agent
license: MIT
prerequisites:
  commands: [git, aider]
metadata:
  hermes:
    tags: [aider, git, pair-programming, coding-agent, automation, ci]
    category: software-development
    related_skills: [subagent-driven-development, code-wiki]
    requires_toolsets: [terminal]
---

# Aider Pair Programming Skill

[Aider](https://aider.chat) is a terminal-native AI pair programmer that edits real files in a git repo and writes one commit per successful change, so every edit is inspectable and revertible on its own. This skill drives Aider **non-interactively** through the `terminal` tool for a scoped coding subtask — it does not replace Hermes' own `patch`/`write_file` tools for the main conversation's edits.

## When to Use

- The user explicitly asks to use Aider, or wants "one git commit per AI edit" so a bad change is a single `git revert` away.
- A self-contained coding subtask should run in its own subprocess/context, independent of the main conversation's edits (e.g. dispatched via `delegate_task`).
- A batch of mechanical edits (rename, add tests, repo-wide lint fix) is cheaper on a different model than the current session's.

Skip it for edits that are already part of the current conversation — use Hermes' own `patch` / `write_file` tools instead; spinning up Aider adds a subprocess and a second commit history to reconcile.

## Prerequisites

- **`git`** — the target directory must already be a git repo (`git init` first if not; Aider refuses to run without one unless `--skip-sanity-check-repo` is set).
- **`aider`** — install once per environment:
  ```
  python -m pip install aider-install && aider-install
  ```
  (bootstraps an isolated `uv`-managed install; `pipx install aider-chat` also works.)
- **Model access** — Aider is a separate subprocess, not routed through Hermes' own LLM call path, so it needs its own credentials:
  - Reuse Hermes' existing `OPENROUTER_API_KEY` (already in `.env`) with `--model openrouter/<model>`, e.g. `openrouter/anthropic/claude-sonnet-4.6`. No extra key needed.
  - Free/local alternative: `ollama pull qwen2.5-coder`, then `export OLLAMA_API_BASE=http://127.0.0.1:11434` and `--model ollama_chat/qwen2.5-coder`.
  - Otherwise export a provider-specific key directly (`ANTHROPIC_API_KEY`, `DEEPSEEK_API_KEY`, …).
  - See `references/aider.conf.yml.example` for a starter `.aider.conf.yml`.

## How to Run

Invoke through the `terminal` tool. Aider's default mode is an interactive REPL — **always** pass `--message` and `--yes-always` so it applies one instruction and exits instead of waiting on stdin (a call missing either flag will hang):

```
terminal('cd /path/to/repo && aider --yes-always --no-show-release-notes \
  --model openrouter/anthropic/claude-sonnet-4.6 \
  --message "Add input validation to parse_config()" config.py')
```

## Quick Reference

| Flag | Purpose |
|---|---|
| `--message "..."` | One-shot instruction; Aider applies it and exits (required for scripted use) |
| `--yes-always` | Auto-confirm every prompt — required, there is no REPL to answer them |
| `--model <name>` | e.g. `openrouter/anthropic/claude-sonnet-4.6`, `ollama_chat/qwen2.5-coder` |
| `--test-cmd "pytest -q" --auto-test` | Re-run tests after each edit; Aider retries until the command exits 0 |
| `--lint-cmd "python: ruff check --fix"` | Per-language lint command, run automatically (`--auto-lint` is on by default) |
| `--no-auto-commits` | Disable Aider's own commit-per-edit when Hermes should stage/commit instead |
| `--no-check-update --no-show-release-notes` | Silence version-check prompts in scripted runs |
| `git log --oneline -n <N>` | Review Aider's commit trail |
| `git revert <sha>` | Undo one of Aider's commits (there's no REPL `/undo` outside its own session) |

## Procedure

1. Confirm the target is a git repo: `terminal('git -C /path/to/repo rev-parse --is-inside-work-tree')`. `git init` first if not.
2. Optionally copy `references/aider.conf.yml.example` to `.aider.conf.yml` at the repo root to fix the model and defaults instead of repeating flags every call.
3. Run Aider with one specific instruction and the exact files it should touch:
   ```
   terminal('cd <repo> && aider --yes-always --model <model> --message "<instruction>" <file1> <file2>')
   ```
   Naming files keeps context small and the diff predictable; omit them only for a genuinely repo-wide instruction.
4. For a task that should self-correct against a test suite, add `--test-cmd "<test command>" --auto-test` — Aider re-edits and re-commits until the command exits 0 or it gives up.
5. Read Aider's stdout (it reports each file changed and the resulting commit hash), then `git log --oneline -3` to confirm what actually landed.
6. If the result is wrong, `git revert <sha>` (adds a new commit, safe on a shared branch) rather than `git reset --hard` unless the branch is disposable.

## Pitfalls

- Never omit `--yes-always` or `--message` — without them Aider opens its interactive prompt and the `terminal` tool call hangs waiting for input that never arrives.
- Aider only commits after a *successful* edit; a failed or partial edit is not committed, so `git status`/`git log` reflect reality — check them before assuming the change landed.
- Local Ollama models (Qwen2.5-Coder, DeepSeek) are free but noticeably weaker on multi-file edits than Claude/GPT-tier models — reserve them for mechanical, single-file changes, not architecture-level work.
- Aider auto-adds `.aider.chat.history.md` / `.aider.input.history` to `.gitignore` on first run in a repo — expected, not a bug; add project files that should stay out of its context to `.aiderignore` (same syntax as `.gitignore`) instead.
- `AIDER_MODEL_API_KEY`-style env vars only affect Aider's own subprocess; setting them does not change which model the current Hermes session uses.

## Verification

`terminal('cd <repo> && git log --oneline -1')` — the top commit's message should describe the instruction just given (Aider auto-generates the commit message from the change it made).
