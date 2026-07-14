# code-review-ensemble

A configurable multi-agent code-review procedure that runs in any coding agent — Claude Code, Codex CLI, Gemini CLI, Cursor, or others — and orchestrates GitHub PR bots, Claude, Codex, and any CLI model you wire up (Gemini, DeepSeek, Kimi, …), then cross-references their findings to separate real bugs from hallucinations.

The set of reviewers isn't hardcoded. It comes from a JSON config you can edit per-repo, so adding a new model is one entry.

## How it works

1. **Trigger any GitHub review bots** on the PR (Cursor Bugbot, CodeRabbit, …) and wait for their findings.
2. **Run the local reviewers in parallel** — Claude sub-agent + every enabled CLI reviewer — on the same review prompt.
3. **Cross-reference everything before judging.** The orchestrator only investigates the code *after* all reviewers have returned — so it judges the reviewers instead of becoming a biased fourth one.
4. **Report** with severity-ordered findings, a per-reviewer agreement table, and a merge recommendation.

## Default reviewers

| Reviewer | Type | Enabled by default | Notes |
|---|---|---|---|
| Bugbot (Cursor) | GitHub bot | ✅ | Needs the [Cursor](https://cursor.com/) GitHub App on the repo |
| CodeRabbit | GitHub bot | — | Needs the [CodeRabbit](https://www.coderabbit.ai/) GitHub App |
| Claude | in-process sub-agent | ✅ | Claude Code only — uses its Agent tool; other hosts: substitute with a `cli` Claude entry |
| Codex | CLI | ✅ | Requires the `codex` CLI |
| Gemini | CLI | — | Requires the Gemini CLI |
| DeepSeek | CLI | — | Via [`llm`](https://llm.datasette.io) + `llm-deepseek` |
| Kimi | CLI | — | Via `llm` + an OpenAI-compatible plugin |

Toggle any with `"enabled": true | false`. Add more with one entry. See [CONFIG.md](CONFIG.md).

## Use with your coding agent

The skill is one markdown procedure (`SKILL.md`) plus a JSON config (`reviewers.default.json`) — any coding agent that can read markdown and run shell commands can follow it. Pick your host:

### Claude Code

Clone into your skills directory:

```bash
# user-level (every project)
git clone https://github.com/swe-workflow/code-review-ensemble.git \
  ~/.claude/skills/code-review-ensemble

# or project-level
git clone https://github.com/swe-workflow/code-review-ensemble.git \
  .claude/skills/code-review-ensemble
```

Invoke:

```
/code-review-ensemble            # detect PR from current branch
/code-review-ensemble 1234       # explicit PR number
```

Marked `disable-model-invocation: true`, so it only runs when you explicitly invoke it.

### Codex CLI

Codex doesn't have a slash-command skill system, but it reads `AGENTS.md` for context and accepts non-interactive prompts. Pick one:

**Direct invocation** — feed `SKILL.md` as the prompt:

```bash
git clone https://github.com/swe-workflow/code-review-ensemble.git
codex exec --full-auto < code-review-ensemble/SKILL.md
```

**Or wire it through `AGENTS.md`** — append to your `~/.codex/AGENTS.md` or the repo's `AGENTS.md`:

```markdown
## Code review

When asked for a thorough code review, follow the procedure at
https://github.com/swe-workflow/code-review-ensemble/blob/main/SKILL.md
```

Then prompt Codex with "code-review-ensemble on PR 1234".

Disable the `claude-agent` reviewer in `reviewers.default.json` (Codex has no in-process Claude sub-agent), or substitute a `cli` Claude entry.

### Gemini CLI

Create `~/.gemini/commands/code-review-ensemble.toml`:

```toml
description = "Multi-agent code review (https://github.com/swe-workflow/code-review-ensemble)"
prompt = """
Follow the procedure at https://github.com/swe-workflow/code-review-ensemble/blob/main/SKILL.md
on PR {{args}}; if no PR number is given, detect it from the current branch with `gh pr view`.
"""
```

Invoke `/code-review-ensemble 1234`. Disable or substitute the `claude-agent` reviewer (Gemini CLI has no in-process Claude sub-agent).

### Cursor

Clone the repo and add a Cursor rule referencing it:

```bash
git clone https://github.com/swe-workflow/code-review-ensemble.git \
  .cursor/skills/code-review-ensemble
```

Create `.cursor/rules/code-review-ensemble.mdc`:

```markdown
---
description: code-review-ensemble trigger
alwaysApply: false
---

When the user asks for "code-review-ensemble" or "thorough code review",
follow the procedure in `.cursor/skills/code-review-ensemble/SKILL.md`.
```

Then in chat: "code-review-ensemble on PR 1234". Disable or substitute the `claude-agent` reviewer.

### Any other agent (Aider, OpenCode, Roo Code, …)

Clone the repo and ask your agent to read and follow `SKILL.md` on the desired PR:

```
Read code-review-ensemble/SKILL.md and follow it on PR 1234.
```

Works with any agent that can read markdown and run shell commands.

## Prerequisites

- `gh` CLI, authenticated (`gh auth status`).
- For each enabled reviewer, its CLI or GitHub App installed:
  - `github-bot`: the bot's GitHub App on the repo (Cursor, CodeRabbit, …).
  - `claude-agent`: **Claude Code only** — uses its in-process Agent tool. In other hosts, disable this reviewer or swap it for a `cli` Claude entry.
  - `cli`: the binary referenced in the `command` template — e.g. `codex`, `gemini`, `llm` + the relevant model plugin.

In Claude Code, `allowed-tools` in `SKILL.md`'s frontmatter is the permission allowlist — add `Bash(<binary>:*)` for any new CLI reviewer or you'll be prompted on every run. Other hosts handle command permissions on their own terms (Codex sandbox, Gemini policy, Cursor settings).

## Configure per project

Copy `reviewers.default.json` to your repo root as `.code-review-ensemble.json` and edit it. The skill resolves the config in this order, **first match wins**:

1. `<repo-root>/.code-review-ensemble.json`
2. `./.code-review-ensemble.json` (current working dir)
3. `reviewers.default.json` (this repo's shipped default)

Full schema, reviewer types, and examples — including how to add Gemini, DeepSeek, Kimi, GLM, or another GitHub bot — live in [CONFIG.md](CONFIG.md).

## Credit

Refactored from Nolan Lawson's original [`code-review-turbo`](https://gist.github.com/nolanlawson/4150b0ca9640654c256b324fac0d5253) gist, generalised from a fixed Bugbot + Claude + Codex trio into a config-driven set of reviewers.
