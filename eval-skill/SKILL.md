---
name: eval-skill
description: A/B test another skill against a control to see if it actually helps. Runs N treatment agents (following the skill) and 1 control agent (not following it) on the same task in isolated worktrees, then has a blind judge score them. Use when asked to "test this skill", "eval this skill", "does this skill actually help", or before trusting a new/edited skill's instructions.
---

# Eval Skill

Test whether `<skill-name>` actually improves output on `<task prompt>`, versus an agent given the same task with no skill guidance.

## Args

- `skill-name` (required) — directory name under `.claude/skills/` (e.g. `audit-docs`, `review-changes`).
- `task prompt` (required) — the concrete task both control and treatment agents will attempt.
- `--trials N` (optional, default 4) — number of treatment runs. Control is always exactly 1.
- `--criteria "..."` (optional) — skip inference entirely and use this rubric verbatim as the judge criteria.

## Process

1. **Resolve the skill path.** Check `.claude/skills/<skill-name>/SKILL.md` exists. If not, stop and report — do not spawn any agents.
2. **Derive judge criteria.**
   - If `--criteria` was passed, use it verbatim and skip straight to step 3.
   - Otherwise, read the target skill's `SKILL.md` (frontmatter `description` plus body) and draft a rubric tailored to what that specific skill is trying to achieve — e.g. for a docs-auditing skill, criteria like "correctly flags stale examples," "doesn't over-strip load-bearing phrasing," "respects one-file-at-a-time scope," rather than generic quality bullets.
   - Show the draft rubric to the user and ask them to confirm, edit, or say "use the default" instead.
   - If the user asks for the default, or inference genuinely isn't possible (e.g. the target skill's `SKILL.md` is too sparse to infer anything specific from), fall back to this generic rubric:
     - Correctness — does the output actually solve the task without introducing errors?
     - Thoroughness — does it cover the scope of the task, or cut corners?
     - Adherence to repo conventions — does it match CLAUDE.md / .cursor/rules/ patterns?
     - Actionability — could someone act on this output as-is, or does it need significant rework?
3. **Run the Workflow script below** via the `Workflow` tool, passing `args: { skillName, taskPrompt, trials, criteria }` where `criteria` is the finalized rubric from step 2.
4. **Reveal and save.** The workflow returns `{ runs, judgeVerdict }` with each run in `runs` carrying an `isControl` boolean (the judge itself never sees this field — it only sees each run's `label`, so its scoring stays blind). After the workflow returns:
   - Read each run's `isControl` field to identify the control — do not infer it from array position, since a dropped/null run can shift indices among survivors.
   - Write full results (all runs' diffs/outputs, judge scores + rationale, revealed labels) to `.claude/personal/eval-runs/<skill-name>-<task-slug>-<n>/results.md`, where `<task-slug>` is a kebab-case slug of the task prompt and `<n>` increments if the directory already exists.
   - Print a short chat summary: did treatment beat control and by how much (per judge scores), and how much the treatment runs diverged from each other (flag high divergence as a sign the skill's instructions are ambiguous).

## Workflow script

```js
export const meta = {
  name: 'eval-skill',
  description: 'A/B test a skill against a control across isolated worktrees, then blind-judge the results',
  phases: [
    { title: 'Run' },
    { title: 'Judge' },
  ],
}

const { skillName, taskPrompt, trials, criteria } = args
const treatmentCount = trials || 4
const skillPath = `.claude/skills/${skillName}/SKILL.md`

const DEFAULT_CRITERIA = [
  'Correctness — does the output actually solve the task without introducing errors?',
  'Thoroughness — does it cover the scope of the task, or cut corners?',
  'Adherence to repo conventions — does it match CLAUDE.md / .cursor/rules/ patterns?',
  'Actionability — could someone act on this output as-is, or does it need significant rework?',
].join('\n')

const rubric = criteria || DEFAULT_CRITERIA

const RUN_SCHEMA = {
  type: 'object',
  properties: {
    summary: { type: 'string' },
    diff: { type: 'string' },
    output: { type: 'string' },
  },
  required: ['summary', 'diff', 'output'],
}

phase('Run')

const controlPrompt = `${taskPrompt}\n\nDo not read or follow any SKILL.md file for this task. Use your own default judgment only.\n\nWhen done, respond with a JSON object matching this shape: { "summary": "<what you did>", "diff": "<full git diff of any files you changed, or empty string>", "output": "<any explicit textual output you produced, or empty string>" }`

const treatmentPrompt = `${taskPrompt}\n\nRead ${skillPath} and follow it exactly to complete this task.\n\nWhen done, respond with a JSON object matching this shape: { "summary": "<what you did>", "diff": "<full git diff of any files you changed, or empty string>", "output": "<any explicit textual output you produced, or empty string>" }`

const thunks = [
  () => agent(controlPrompt, { label: 'control', phase: 'Run', schema: RUN_SCHEMA, isolation: 'worktree' }),
  ...Array.from({ length: treatmentCount }, (_, i) => () =>
    agent(treatmentPrompt, { label: `treatment-${i + 1}`, phase: 'Run', schema: RUN_SCHEMA, isolation: 'worktree' })
  ),
]

const rawRuns = await parallel(thunks)

const runs = rawRuns
  .map((result, index) => ({ isControl: index === 0, result }))
  .filter((entry) => entry.result)
  .map((entry, survivorIndex) => ({
    label: `Run ${String.fromCharCode(65 + survivorIndex)}`,
    isControl: entry.isControl,
    summary: entry.result.summary,
    diff: entry.result.diff,
    output: entry.result.output,
  }))

if (runs.length < 2) {
  return { runs, judgeVerdict: null, error: 'Fewer than 2 runs survived — cannot judge.' }
}

phase('Judge')

const JUDGE_SCHEMA = {
  type: 'object',
  properties: {
    scores: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          label: { type: 'string' },
          score: { type: 'number' },
          rationale: { type: 'string' },
        },
        required: ['label', 'score', 'rationale'],
      },
    },
    ranking: { type: 'array', items: { type: 'string' } },
  },
  required: ['scores', 'ranking'],
}

const judgePrompt = `You are blind-judging ${runs.length} independent attempts at the same task: "${taskPrompt}"\n\nScore each run 1-10 against this rubric:\n${rubric}\n\nRuns:\n${runs.map((run) => `--- ${run.label} ---\nSummary: ${run.summary}\nDiff:\n${run.diff}\nOutput:\n${run.output}`).join('\n\n')}\n\nYou are not told which run (if any) used a specific skill's instructions — judge purely on merit against the rubric above. Return scores per run and an overall ranking from best to worst.`

const judgeVerdict = await agent(judgePrompt, {
  label: 'judge',
  phase: 'Judge',
  schema: JUDGE_SCHEMA,
  model: 'claude-opus-4-8',
  effort: 'high',
})

return { runs, judgeVerdict }
```

## Output format

`.claude/personal/eval-runs/<skill-name>-<task-slug>-<n>/results.md`:

```markdown
# Eval: <skill-name> — "<task prompt>"

## Verdict
<did the control or a treatment run rank highest, judge's overall take>

## Runs

### Run A (control)
Score: X/10 — <rationale>
<summary, diff, output>

### Run B (treatment)
...
```

## Error handling

- Unknown skill name: report and stop before running the workflow.
- Fewer than 2 surviving runs: report the workflow's `error` field, do not fabricate a verdict.
