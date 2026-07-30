# Kencensus PR Review

A consensus-based PR review skill for Claude Code. Two independent reviewers —
Claude Code and OpenAI Codex — review the same diff, then cross-examine each
other's findings before anything is reported. The result is a ranked list of
PR comments worth leaving, with the weak findings filtered out by debate
rather than by hope.

## How to use

In any git repo, with the PR branch checked out:

- `/kencensus-pr-review`, or
- ask in plain words: "run a kencensus review of this branch against main"

The skill reviews local git state (branch vs. base ref). For a GitHub PR,
check it out first (`gh pr checkout <n>`).

## What you get

1. **PR comments to leave** — ranked most → least important, each with
   repo-relative `file:line-range`, severity, consensus status, and the
   comment text to post. Comments that did **not** reach consensus stay in
   the list, marked `CONTESTED`, with each reviewer's position.
2. **Debunked during review** — findings one reviewer raised that were
   refuted with evidence during cross-examination, and who raised them.

Nothing is posted anywhere automatically; the output is the list.

## How it works

1. **Parallel independent reviews** — Codex reviews the diff via the Codex
   plugin's companion runtime while Claude Code does its own review of the
   same diff. Claude records its findings before reading Codex's output.
2. **Reconciliation** — findings are matched by file, line overlap, and root
   cause. Matches become consensus. Codex-only findings are verified by
   Claude; Claude-only findings are sent back to Codex in a single read-only
   cross-examination round (CONFIRM/REFUTE protocol — see
   `debate-prompt.md`). Codex typically *executes* the code to back its
   verdicts.
3. **Ranking and report** — consensus and contested comments ranked by
   real-world impact; refuted findings listed as debunked with the evidence
   that killed them.

## Dependencies

| Dependency | Why | Install / check |
|---|---|---|
| Claude Code | Orchestrator and first reviewer | — |
| Codex plugin (`codex@openai-codex`) | Provides `codex-companion.mjs`, the runtime that drives Codex reviews | `/plugin marketplace add openai/codex-plugin-cc`, then `/plugin install codex@openai-codex` |
| Codex CLI (`@openai/codex`) | The second reviewer | `npm install -g @openai/codex` |
| OpenAI auth | Codex must be logged in | `codex login` (ChatGPT login works) |
| Node.js + npm | Runs the companion script and Codex CLI | any recent Node |
| git | The diff under review | — |
| GitHub CLI (`gh`) | Optional — only to check out a PR by number | `brew install gh` |

Preflight check (also run automatically by the skill):

```bash
COMPANION=$(ls -d "$HOME"/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs | sort -V | tail -1)
node "$COMPANION" setup --json   # want "ready": true and "loggedIn": true
```

## Files

- `SKILL.md` — the agent-facing pipeline instructions (the skill itself)
- `debate-prompt.md` — the verified cross-examination prompt template
- `README.md` — this file

## Caveats

- Codex runs shell commands during review, including executing the code under
  review — only point it at code you would be willing to run.
- Each review/debate call is a fresh Codex thread; expect one to a few
  minutes per Codex leg.

Verified end-to-end on 2026-07-30 with codex-cli 0.146.0, Codex plugin 1.0.5,
Node v25.8.1.
