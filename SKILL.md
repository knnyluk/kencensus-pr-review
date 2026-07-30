---
name: kencensus-pr-review
description: Consensus PR review by two independent reviewers — Claude Code and Codex each review the diff, cross-examine each other's findings, and output a ranked list of PR comments (consensus and contested) plus a list of debunked comments. Use when asked to run kencensus, kencensus review, consensus review, or to review a PR/branch with both Claude and Codex.
---

# Kencensus PR Review

Two independent reviewers — you (Claude Code) and Codex — review the same
diff, then reconcile through cross-examination. The deliverable is a ranked
list of PR comments to leave (most important first, with file + line for
each), contested comments included and marked, followed by the comments that
were debunked during reconciliation.

Codex is driven through the Codex plugin's companion script. Shell state does
not persist between Bash tool calls, so **prepend this resolver line to every
command below that uses `$COMPANION`** (it survives plugin version bumps):

```bash
COMPANION=$(ls -d "$HOME"/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs | sort -V | tail -1)
```

## Prerequisites

- The `codex` plugin (openai-codex marketplace) installed, with the Codex CLI
  authenticated. Verify:

```bash
node "$COMPANION" setup --json
```

  Require `"ready": true` and `"loggedIn": true`. If not logged in, tell the
  user to run `!codex login`.
- A git repo with the PR branch checked out. If given a GitHub PR number,
  check the branch out first (e.g. `gh pr checkout <n>`), then proceed —
  the pipeline only needs local git state.

## Pipeline

Run every companion command with the shell cwd **inside the target repo** —
it reviews that repo's git state, not a path argument. `<base>` is the branch
the PR merges into (usually `main` or `master`).

### Stage 1 — launch Codex, then do your own review (independently)

Launch the Codex leg first so it runs while you review. Use a Bash background
task (`run_in_background: true`) with a generous timeout — a small diff takes
one to a few minutes:

```bash
cd <repo> && node "$COMPANION" review --wait --base <base>
```

While Codex runs, do YOUR review of `git diff <base>...HEAD` plus the full
changed files. **Write your findings down before reading any Codex output** —
independence is the point. For each finding record: an ID (C1, C2, …),
severity (critical/high/medium/low), `file:line-range`, the claim, and the
evidence. Verify claims against the actual code, not just the diff hunks;
run the code when it is safe and cheap to do so.

Then collect the Codex output. Its findings appear after the `# Codex Review`
header as `[P1]`-style bullets with absolute file paths and line ranges
(everything before that header is progress noise). `[P1]` maps to
critical/high, `[P2]` medium, `[P3]` low.

### Stage 2 — reconcile

Match the two finding sets by file + overlapping lines + same root cause.

- **Both raised it** → consensus. Keep the clearer wording, higher of the two
  severities.
- **Codex-only findings** → you verify: read the code, run it if safe. Agree
  → consensus. Can you demonstrate it's wrong (with evidence) → debunked.
  Genuinely unsure after a real attempt → contested.
- **Your-only findings** → cross-examine via Codex in ONE read-only `task`
  call (never pass `--write`). Fill in the template at
  [debate-prompt.md](debate-prompt.md) with all disputed findings and run:

```bash
cd <repo> && node "$COMPANION" task "<filled-in debate prompt>"
```

  Codex ends with a `A: CONFIRM|REFUTE` line per finding and evidence
  (it typically executes the code to check). CONFIRM → consensus. REFUTE →
  re-check its evidence yourself: if you agree → debunked; if you still
  believe the defect is real → contested, one rebuttal round max — record
  both positions and move on.

### Stage 3 — output

Produce exactly this structure (paths repo-relative — strip the absolute
prefix Codex prints):

```markdown
## PR comments to leave (most important first)

1. **`cart.js:3`** [high · consensus] — <comment text to post, imperative,
   states the defect and the fix>
2. **`auth.js:7-8`** [high · consensus] — …
3. **`cart.js:10-12`** [low · consensus after debate] — …
4. **`foo.js:42`** [medium · CONTESTED — Claude: real, Codex: refutes] —
   <comment text, plus one line per side's position>

## Debunked during review

- **`cart.js:17-18`** — "sort() mutates the caller's array" — refuted:
  `.filter()` on line 16 creates a fresh array; runtime check confirmed the
  caller's array is unchanged. (Raised by: Claude)
```

Rank by real-world impact, not by who raised it. Contested comments stay in
the ranked list, marked, with both positions in one line each. Every debunked
item names who raised it and the evidence that killed it. If a section is
empty, say so explicitly ("No contested comments." / "Nothing was debunked.").

Do NOT post the comments anywhere — the deliverable is the list.

## Gotchas

- Each companion call starts a **fresh Codex thread** — the debate `task` has
  no memory of the review run, so restate each disputed finding fully
  (file, lines, claim) in the prompt.
- `review` accepts no focus text; anything custom (like the debate) goes
  through `task`. Plain `task` without `--write` is read-only — never add
  `--write` in this skill.
- Codex prints **absolute** file paths; convert to repo-relative before
  writing the final list.
- Codex runs shell commands during review (including `node -e` to execute
  your code) — that's normal; it makes its CONFIRM/REFUTE verdicts unusually
  trustworthy, and it means the review should only run on code you'd be
  willing to execute.
- Set the Bash timeout to 600000 for both `review` and `task`; the first call
  also boots the shared Codex runtime on demand.
- Keep your Stage-1 review honestly independent. If you read Codex's findings
  first, the "two reviewers" premise collapses into one reviewer with an echo.

## Troubleshooting

- `setup --json` says Codex unavailable → `npm install -g @openai/codex`,
  re-run setup.
- `loggedIn: false` → user runs `!codex login`.
- Review output says nothing to review → wrong cwd (must be inside the target
  repo) or wrong `--base` ref; check `git diff --shortstat <base>...HEAD`.
