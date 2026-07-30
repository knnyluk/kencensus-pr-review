# Debate prompt template

Fill in and pass as the single argument to `node "$COMPANION" task "..."`.
One call carries ALL disputed findings (letter them A, B, C, …). Verified
shape — Codex answers with per-finding evidence and a final verdict line
each.

```text
You are cross-examining candidate code review findings on the diff of branch
<branch> against <base> in this repo. Do NOT edit any files. For each finding
below, give a verdict of CONFIRM or REFUTE, with concrete evidence from the
code (run it if helpful). Be adversarial: refute anything that is not a real
defect.

Finding A: <file> lines <start>-<end>, <claim in one or two sentences,
including the concrete failure it would cause>. Severity proposed: <severity>.

Finding B: <file> lines <start>-<end>, <claim>. Severity proposed: <severity>.

End with a line per finding: 'A: CONFIRM|REFUTE' and 'B: CONFIRM|REFUTE'.
```
