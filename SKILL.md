---
name: code-review-turbo
description: Run a configurable multi-agent code review on the current branch's PR. Triggers any configured GitHub review bots, runs Claude, Codex, and any other CLI models (Gemini, DeepSeek, Kimi, …) in parallel, then cross-references all findings to filter out hallucinations. Reviewers are defined in a config file, so you can add or remove models. Use when you want a thorough, multi-perspective code review before merging.
metadata:
  disable-model-invocation: 'true'
  argument-hint: '[pr-number]'
allowed-tools: Bash(gh:*) Bash(git:*) Bash(codex:*) Bash(gemini:*) Bash(llm:*) Bash(cat:*) Bash(tee:*) Bash(sleep:*) Bash(test:*) Agent Read Grep Glob Write(/tmp/*)
---

# Code Review Turbo

Configurable multi-agent code review: GitHub review bots + Claude sub-agent + Codex + any other CLI models you add, with cross-referencing to separate real bugs from hallucinations.

The set of reviewers is **not hardcoded** — it comes from a config file. See [CONFIG.md](CONFIG.md) for the schema and how to add models like Gemini, DeepSeek, or Kimi.

**Host-agent-agnostic.** This procedure runs in any coding agent that can read markdown and run shell commands — Claude Code, Codex CLI, Gemini CLI, Cursor, … The YAML frontmatter above is Claude Code skill metadata; other agents ignore it. See [README.md](README.md#use-with-your-coding-agent) for per-agent install. The only host-specific reviewer type is `claude-agent` (Claude Code's in-process Agent tool) — in other hosts, disable it or swap in a `cli` Claude entry such as `{ "name": "Claude", "type": "cli", "command": "claude -p < {PROMPT_FILE}" }`.

## Step 0: Load Reviewer Configuration

Resolve the config file with this cascade (**first match wins**):

1. `<repo-root>/.code-review-turbo.json` — get the repo root with `git rev-parse --show-toplevel` (if that fails — e.g. not a git repo — skip to 2)
2. `./.code-review-turbo.json` in the current working directory
3. `reviewers.default.json` shipped alongside this `SKILL.md`

Read the resolved file and parse the `reviewers` array. If a config file is found but cannot be parsed as valid JSON, STOP and tell the user which file is malformed — do not silently fall back, since they intended a specific config.

Keep only reviewers where `enabled` is not `false`. Each reviewer has a `name`, a `type` (`github-bot`, `claude-agent`, or `cli`), and type-specific fields documented in [CONFIG.md](CONFIG.md).

Tell the user which reviewers are enabled (e.g. "Running 4 reviewers: Bugbot, Claude, Codex, Gemini"). Then group them by type:

- **`github-bot`** reviewers run first (Step 1) — they are async and need polling.
- **`claude-agent`** and **`cli`** reviewers run together in parallel (Step 3).

If **no** `github-bot` reviewers are enabled, skip Step 1 entirely.

## Step 1: Trigger and Collect GitHub Review Bots

Run this step only if at least one `github-bot` reviewer is enabled. It handles **all** enabled bots together: the PR-level work happens once, and only the trigger and match steps are per-bot.

**Matching convention.** A comment belongs to a bot when its `author.login` (lowercased) exactly equals one of that bot's `authorLogins` — exact match, not substring, so a human like `coderabbit-fan` can't be mistaken for `coderabbitai[bot]`. In jq:

```
select(.author.login | ascii_downcase | IN("cursor[bot]"))   # IN(<the bot's authorLogins, lowercased>)
```

If a bot's findings never get picked up, inspect `author.login` on one of its real comments and correct its `authorLogins`.

### Step 1a — Ensure a PR exists (once)

Determine the PR number: use `$ARGUMENTS` if provided, otherwise detect it from the current branch:

```
gh pr view --json number,isDraft -q '{number: .number, isDraft: .isDraft}'
```

If no PR exists, create a **draft** PR for the current branch:

```
gh pr create --draft --fill
```

Note the PR number and whether it is a draft. Do this once, no matter how many bots are enabled.

### Step 1b — Get the head-commit timestamp (once)

```
gh pr view <number> --json commits --jq '.commits | last | .committedDate'
```

This is the staleness baseline for every bot — fetch it a single time.

### Step 1c — Trigger each bot that needs it

Fetch the existing comments once, from both sources, so you can inspect them per bot:

```
gh pr view <number> --json comments
gh api repos/{owner}/{repo}/pulls/<number>/comments
```

For each enabled `github-bot`, post its `trigger` comment when **any** of these hold:

- the PR was just created in 1a, or is a **draft** with no prior `trigger` comment for that bot;
- the bot has **never** commented (in either source); or
- the bot's most recent comment is **older** than the head-commit timestamp from 1b — its review is stale, commits landed after it last ran.

Skip the trigger when the bot already has a comment newer than the head commit (its review is current). To trigger:

```
gh pr comment <number> --body "<the bot's trigger>"
```

Track which bots you triggered — those are the ones to wait for in 1d. Tell the user what you did (e.g. "Bugbot was stale; triggered a fresh review. CodeRabbit is current.").

### Step 1d — Poll once for the triggered bots

If you triggered no bots, skip to 1e. Otherwise poll in a single shared window — run individual commands, do **not** write a bash loop:

1. Fetch the bots' comments once, across both sources:
   - `gh pr view <number> --json comments` (top-level)
   - `gh api repos/{owner}/{repo}/pulls/<number>/comments` (inline review comments)
2. A triggered bot has **responded** when it has a comment in either source newer than the head-commit timestamp from 1b.
3. If any triggered bot hasn't responded yet, `sleep 30` and repeat from 1.
4. Stop when every triggered bot has responded, or after 30 iterations (~15 minutes total) — **one** window covering all bots, not 15 minutes per bot.

If a bot never shows up, warn the user which one, and ask whether to proceed without it or keep waiting.

### Step 1e — Collect findings once

Bots post inline review comments (one per issue) and resolve them when fixed. Resolution status is only exposed by the **GraphQL API**, so fetch every review thread once and partition by bot afterwards.

Get the repo owner and name:

```
gh repo view --json owner,name --jq '.owner.login + "/" + .name'
```

Fetch all review threads with their resolution status (one query, all bots):

```
gh api graphql -f query='
  query {
    repository(owner: "<OWNER>", name: "<REPO>") {
      pullRequest(number: <NUMBER>) {
        reviewThreads(first: 100) {
          nodes {
            isResolved
            comments(first: 10) {
              nodes { author { login } body path line }
            }
          }
        }
      }
    }
  }
'
```

Keep only threads where `isResolved` is `false` — **ignore resolved threads**, which are issues a bot already confirmed fixed. Assign each remaining thread to a bot using the matching convention above (a comment whose `author.login` matches that bot's `authorLogins`).

Also fetch top-level PR comments once (rare for these bots, but possible) and assign them the same way, keeping only non-minimized ones:

```
gh pr view <number> --json comments
```

Save each bot's active findings, tagged with the reviewer's `name`, for the cross-reference in Step 4.

## Step 2: Build the Review Prompt

Gather the PR context by running these commands:

```
gh pr diff <number>
gh pr view <number> --json title,body,baseRefName,headRefName
```

Then construct the following review prompt (referred to as `REVIEW_PROMPT` below). This EXACT prompt must be used for EVERY `claude-agent` and `cli` reviewer — do not alter it between reviewers:

---

**START OF REVIEW_PROMPT**

You are reviewing a pull request. Here is the diff:

<INSERT PR DIFF HERE>

PR title: <INSERT>
PR description: <INSERT>
Base branch: <INSERT>
Head branch: <INSERT>

Review this PR thoroughly. Focus on these categories IN ORDER OF IMPORTANCE:

### 1. Functional Bugs (MOST IMPORTANT)
Look for logic errors, off-by-one errors, null/undefined issues, race conditions, incorrect conditionals, missing edge cases, wrong variable usage, broken control flow, and any code that simply won't work as intended. This is BY FAR the most important category.

### 2. KISS Violations
Overly complex solutions where simpler ones exist. Unnecessary abstractions, premature generalizations, or convoluted logic.

### 3. DRY Violations
Duplicated logic that should be extracted. Copy-pasted code with minor variations.

### 4. Missing Tests
New functionality or bug fixes lacking appropriate test coverage.

### 5. Performance Issues
- For SQL queries: DO NOT GUESS what the query planner will do. Instead, run `EXPLAIN ANALYZE` on the actual local database to verify.
- For migrations: Will they lock tables for too long? Are they safe for large tables?
- For application code: N+1 queries, unnecessary allocations, missing batching, O(n^2) loops on large datasets.

### 6. Accessibility Issues
For any TSX/JSX files: missing aria labels, improper heading hierarchy, missing alt text, keyboard navigation issues, color contrast concerns.

DO NOT report:
- Code formatting or style issues (these are linted automatically)
- Minor TypeScript type issues (also linted)
- Nitpicks that don't affect correctness or maintainability

For each issue found, report:
- **File and line number** (from the diff)
- **Severity**: critical / high / medium / low
- **Category**: which of the above categories
- **Description**: what the issue is and why it matters
- **Suggestion**: how to fix it

Return a structured list grouped by severity (critical first, then high, medium, low).

**END OF REVIEW_PROMPT**

---

## Step 3: Run All Local Reviewers in Parallel

Launch every enabled `claude-agent` and `cli` reviewer concurrently. Use whichever parallel-execution mechanism your host agent supports (Claude Code: issue all invocations in one tool-call batch; others: background subprocesses or the host's equivalent).

### claude-agent reviewers (Claude Code only)
For each, use Claude Code's Agent tool to spawn an in-process sub-agent with the full `REVIEW_PROMPT` and access to Bash, Read, Grep, and Glob so it can run EXPLAIN queries and inspect code.

**If the host agent isn't Claude Code**, these reviewers can't run as-is. Either skip them with a notice to the user (Claude won't appear in the cross-reference), or substitute a `cli` Claude entry such as `{ "name": "Claude", "type": "cli", "command": "claude -p < {PROMPT_FILE}" }`.

### cli reviewers
Write the `REVIEW_PROMPT` ONCE to a randomly-named temp file (e.g. `/tmp/review-prompt-<random-8-chars>.txt` — generate a unique random suffix to avoid collisions with concurrent runs). All `cli` reviewers share this one file.

For each `cli` reviewer, take its `command` from the config, substitute `{PROMPT_FILE}` with the temp file path, and run it. For example, a reviewer with `"command": "codex exec --full-auto - < {PROMPT_FILE}"` runs as:

```
codex exec --full-auto - < /tmp/review-prompt-<random>.txt
```

(For Codex, `--full-auto` prevents prompting for approval on shell commands, and `-` reads the prompt from stdin. Other CLIs have their own non-interactive flags — see [CONFIG.md](CONFIG.md).)

Tag each reviewer's output with its `name` for the cross-reference step.

## Step 4: Cross-Reference and Validate

**CRITICAL: DO NOT do any of your own code research, file reading, or EXPLAIN queries until ALL reviewers (every github-bot, claude-agent, and cli reviewer) have returned their results.** If you investigate the code first, you will form your own opinions and become an extra reviewer with a veto over the others — biased toward confirming your own findings and dismissing theirs. The whole point of this step is to be an OBJECTIVE judge of the independent reviewers.

### Step 4a: Compile findings FIRST (no research yet)

Collect and deduplicate all findings from every reviewer into a single list. For each unique issue, note which reviewer(s) reported it. Do NOT yet judge whether the issues are real — just organize them.

### Step 4b: NOW do your own research to validate

Only after compiling the full list, go through each finding and verify it:

- **Read the actual source code** around each reported issue (not just the diff)
- **Run EXPLAIN ANALYZE** on any flagged SQL queries against the local database
- **Check test files** to see if flagged "missing tests" actually exist
- **Trace the logic** for any reported functional bugs — actually verify the bug is real

For each unique issue, determine:
- Is it a **real issue** (confirmed by your investigation)?
- Is it a **hallucination** (the code doesn't actually have this problem)?
- Which reviewers found it and which missed it?

Be especially careful not to dismiss a finding just because only one reviewer reported it — sometimes the lone dissenter found the most critical bug.

## Step 5: Final Report

Present the validated findings in this format:

### Critical Issues
(issues you confirmed are real and need fixing before merge)

### High Issues
(real issues that should be fixed)

### Medium Issues
(real but lower-risk issues)

### Low Issues
(minor improvements, optional)

### Dismissed Findings
(issues reported by reviewers that turned out to be hallucinations or false positives — briefly explain why each was dismissed)

### Reviewer Agreement Summary
A table with **one column per enabled reviewer** (use their configured `name`s) showing which reviewer found which real issue:

| Issue | Bugbot | Claude | Codex | … | Verdict |
|-------|--------|--------|-------|---|---------|
| ...   | ...    | ...    | ...   | … | ...     |

End with a clear **merge recommendation**: ready to merge, merge after fixes, or needs significant rework.

---

## Reference

Based on Nolan Lawson's original "code-review-turbo" skill: <https://gist.github.com/nolanlawson/4150b0ca9640654c256b324fac0d5253>. This version generalizes the fixed Bugbot + Claude + Codex trio into a config-driven set of reviewers (see [CONFIG.md](CONFIG.md)).
