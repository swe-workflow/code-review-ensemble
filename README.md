# code-review-turbo

A configurable multi-agent code-review skill for [Claude Code](https://www.anthropic.com/claude-code). Runs every reviewer you've enabled — GitHub PR bots, the in-process Claude sub-agent, Codex, and any other CLI model you wire up (Gemini, DeepSeek, Kimi, …) — then cross-references their findings to separate real bugs from hallucinations before reporting.

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
| Claude | in-process sub-agent | ✅ | Uses Claude Code's Agent tool |
| Codex | CLI | ✅ | Requires the `codex` CLI |
| Gemini | CLI | — | Requires the Gemini CLI |
| DeepSeek | CLI | — | Via [`llm`](https://llm.datasette.io) + `llm-deepseek` |
| Kimi | CLI | — | Via `llm` + an OpenAI-compatible plugin |

Toggle any with `"enabled": true | false`. Add more with one entry. See [CONFIG.md](CONFIG.md).

## Install

This is a Claude Code skill — drop it into your skills directory:

```bash
# user-level (available in every project)
git clone https://github.com/swe-workflow/code-review-turbo.git \
  ~/.claude/skills/code-review-turbo

# or project-level
git clone https://github.com/swe-workflow/code-review-turbo.git \
  .claude/skills/code-review-turbo
```

Then in Claude Code:

```
/code-review-turbo            # detect PR from current branch
/code-review-turbo 1234       # explicit PR number
```

The skill is marked `disable-model-invocation: true`, so it only runs when you invoke it explicitly — it won't surprise you mid-conversation.

## Prerequisites

- `gh` CLI, authenticated (`gh auth status`).
- For each enabled reviewer, its CLI or GitHub App installed:
  - `github-bot`: the bot's GitHub App on the repo (Cursor, CodeRabbit, …).
  - `claude-agent`: nothing extra (it's Claude Code itself).
  - `cli`: the binary referenced in the `command` template — e.g. `codex`, `gemini`, `llm` + the relevant model plugin.

If a CLI you've added isn't already in `SKILL.md`'s `allowed-tools` line, add `Bash(<binary>:*)` or you'll be prompted on every run. See [CONFIG.md](CONFIG.md#allowed-tools-note).

## Configure per project

Copy `reviewers.default.json` to your repo root as `.code-review-turbo.json` and edit it. The skill resolves the config in this order, **first match wins**:

1. `<repo-root>/.code-review-turbo.json`
2. `./.code-review-turbo.json` (current working dir)
3. `reviewers.default.json` (this repo's shipped default)

Full schema, reviewer types, and examples — including how to add Gemini, DeepSeek, Kimi, GLM, or another GitHub bot — live in [CONFIG.md](CONFIG.md).

## Credit

Refactored from Nolan Lawson's original [`code-review-turbo`](https://gist.github.com/nolanlawson/4150b0ca9640654c256b324fac0d5253) gist, generalised from a fixed Bugbot + Claude + Codex trio into a config-driven set of reviewers.
