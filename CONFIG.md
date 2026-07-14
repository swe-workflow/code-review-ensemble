# Reviewer Configuration

`code-review-ensemble` reads a JSON config that lists the reviewers to run. Add, remove, enable, or disable any reviewer here — including the built-in Bugbot, Claude, and Codex.

## Where the config lives (cascade)

The skill resolves the config with this order, **first match wins**:

1. `<repo-root>/.code-review-ensemble.json` (repo root via `git rev-parse --show-toplevel`)
2. `./.code-review-ensemble.json` (current working directory)
3. `reviewers.default.json` shipped next to `SKILL.md`

To customize per-project, copy `reviewers.default.json` to your repo root as `.code-review-ensemble.json` and edit it.

## Schema

```jsonc
{
  "reviewers": [
    {
      "name": "Bugbot",          // label shown in the report (required, must be unique)
      "type": "github-bot",       // "github-bot" | "claude-agent" | "cli" (required)
      "enabled": true,            // omit or true to run; false to skip
      // --- github-bot fields ---
      "trigger": "@cursor review",      // PR comment posted to invoke the bot
      "authorLogins": ["cursor[bot]"]   // exact GitHub login(s) the bot comments under
    }
  ]
}
```

### Reviewer types

| `type` | How it runs | Can investigate code? | Required fields |
|--------|-------------|-----------------------|-----------------|
| `github-bot` | Posts a trigger comment on the PR, then polls for the bot's review comments | Yes (cloud) | `trigger`, `authorLogins` |
| `claude-agent` | In-process Claude sub-agent via Claude Code's Agent tool (**Claude Code only**) | Yes (Bash/Read/Grep/Glob) | — |
| `cli` | Runs a shell command with the review prompt on stdin | Only if the CLI is agentic | `command` |

**Agentic vs. one-shot `cli` reviewers:** agentic CLIs (Codex, Gemini CLI, Claude) can run commands (`EXPLAIN ANALYZE`, read files) to validate findings. One-shot wrappers (e.g. `llm`) only see the prompt + diff and reason from that. Both contribute findings to the cross-reference; just expect less depth from one-shot reviewers.

**`claude-agent` and non-Claude-Code hosts.** The `claude-agent` type uses Claude Code's in-process Agent tool — it only runs when the host coding agent *is* Claude Code. In Codex CLI, Gemini CLI, Cursor, or any other host, disable these entries (`"enabled": false`) or substitute a `cli` Claude reviewer: `{ "name": "Claude", "type": "cli", "command": "claude -p < {PROMPT_FILE}" }`.

## Adding a `cli` model

Set `command` to any shell invocation that runs **non-interactively** and reads the prompt from stdin. The token `{PROMPT_FILE}` is replaced with the path to a temp file containing the review prompt.

**Pipe the prompt via stdin** (`< {PROMPT_FILE}`) rather than inlining it as an argument — the prompt contains backticks and `$`, which the shell would otherwise expand.

The shipped `reviewers.default.json` already lists **Codex** (enabled) plus **Gemini**, **DeepSeek**, and **Kimi** as disabled examples — enable one with `"enabled": true`, or copy it into your project config. To add a model that isn't there yet, follow the same shape (adjust flags/model names to your installed CLI):

```jsonc
{ "name": "GLM", "type": "cli", "command": "llm -m zhipu/glm-4.6 < {PROMPT_FILE}" }
```

[`llm`](https://llm.datasette.io) (`pip install llm`, then install the relevant plugin such as `llm-gemini`, `llm-deepseek`) is a convenient universal wrapper for one-shot reviewers across many providers.

### `allowed-tools` note

`SKILL.md` front-matter allowlists the CLIs it can invoke without a permission prompt (`codex`, `gemini`, `llm`, …). If you configure a reviewer whose command starts with a **different** binary, add `Bash(<binary>:*)` to the `allowed-tools` line in `SKILL.md`, or you'll be prompted on every run.

## Adding another GitHub review bot

`reviewers.default.json` already ships **CodeRabbit** as a disabled `github-bot` example — enable it, or duplicate the entry for another bot. Set `authorLogins` to the bot's exact GitHub App login(s); inspect `author.login` on one of its existing comments to confirm the value (GitHub App bots carry a `[bot]` suffix, e.g. `coderabbitai[bot]`).
