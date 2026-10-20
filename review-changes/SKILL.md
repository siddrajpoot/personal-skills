---
name: review-changes
description: Structured code review using a graph-derived impact map and team standards. Bare invocation reviews local working-tree changes in slim mode with ponytail; if the working tree is clean, it reviews the branch against its parent. Opt into full/Codex, skip ponytail, or force a branch/PR review. Use when reviewing PRs, diffs, or local changes.
disable-model-invocation: true
---

## Review Changes

Graph supplies _context_; reviewers judge the diff. Resolve scope, mode, and ponytail from the invocation, then run the matching phases.

**Bare invocation** (`/review-changes` with nothing after it) means: **local** + **slim** + **ponytail on**. Do not ask. Do not dispatch Codex.

### Scope

| Scope               | Triggers                                                   | Diff                                           |
| ------------------- | ---------------------------------------------------------- | ---------------------------------------------- |
| **local** (default) | none, "local", "working tree", "uncommitted", "wip"        | staged + unstaged + untracked vs `HEAD`        |
| **branch**          | "branch", "PR", "against base", "committed", "full branch" | commits + that working tree vs resolved parent |

If **local** and the working tree is clean, switch to **branch** (same slim/ponytail settings) and continue. Announce the switch. Do not ask.

### Mode

Slim and full are mutually exclusive.

| Mode               | Triggers                                                                        | Reviewers                      | Skip                                         |
| ------------------ | ------------------------------------------------------------------------------- | ------------------------------ | -------------------------------------------- |
| **slim** (default) | none, "slim", "solo", "no Codex", "without Codex", "skip Codex", "quick review" | Claude only                    | Phase 3 Codex dispatch and Phase 4 reconcile |
| **full**           | "full", "full review", "two-reviewer", "with Codex"                             | Claude + Codex, then reconcile | —                                            |

### Ponytail overlay

Independent of slim/full. **On** by default. Off: `no ponytail`, `skip ponytail`, `without ponytail`, `stop ponytail-review`.

When on, load `ponytail-review` and run Phase 2.5 on the same scoped diff. Do not copy ponytail's tags or boundaries into this skill.

Ponytail owns over-engineering (`delete` / `stdlib` / `native` / `yagni` / `shrink`). Phase 2 owns correctness, contracts, call-site migrations, standards, coverage, spaghetti, and layer leaks. Do not file the same cut in both places.

Announce scope, mode, and ponytail on/off in one sentence before Phase 0. In slim mode, do **not** invoke `codex:rescue`, `codex-companion.mjs`, or any Codex/subagent second pass. Do not stall or apologize for skipping Codex.

### Phase 0 — Resolve the comparison

**Local:** comparison is `HEAD`. Do not resolve a parent branch. Graph and (if full) Codex use working-tree scope vs `HEAD`.

**Branch:** review against the actual parent the current branch was cut from. Resolve `<base-branch>` with this priority chain and use the first that succeeds:

1. **Open PR base** — `gh pr view --json baseRefName -q .baseRefName 2>/dev/null`.
   This is the source of truth when a PR exists.
2. **Fork-point parent** — the branch this one most recently diverged from. Pick the
   candidate with the closest (most recent) merge-base to HEAD:
   ```
   for ref in $(git for-each-ref --format='%(refname:short)' refs/heads refs/remotes/origin \
        | grep -vE "(^|/)$(git branch --show-current)$"); do
     base=$(git merge-base HEAD "$ref" 2>/dev/null) || continue
     echo "$(git rev-list --count "$base..HEAD") $ref"
   done | sort -n | head -1
   ```
   The ref with the fewest commits in `base..HEAD` is the nearest parent.
3. **Fallback** — `stage`, only if nothing above resolves.

Do **not** use `@{u}` / the upstream tracking ref. That is usually this feature branch on the remote, not the parent.

Confirm the resolved base with the user only if it is surprising (e.g. the PR base is a non-integration feature branch). Use that `<base-branch>` for the graph diff and, in **full** + **branch** mode, the Codex pass.

### Phase 1 — Build the structural map (graph)

Always run this phase. If graph MCP tools are not available, stop and ask the user
to enable `code-review-graph` (for example `/mcp`). Do not review from the git diff alone.

Delegate Phase 1 entirely to an **Explore subagent** to keep intermediate graph
output out of the main context. Pass **local** (`HEAD` / working tree) or the
resolved `<base-branch>` and ask the subagent to:

1. Run these queries **in parallel** (they are independent):
   - `get_affected_flows` to find impacted execution paths.
   - `query_graph` pattern="callers_of" on each changed function/symbol to find every
     call site (catches missed-migration and contract-mismatch bugs).
   - `query_graph` pattern="tests_for" on high-risk functions to check coverage.
   - `get_impact_radius` to understand blast radius.
2. Also run `detect_changes` for a risk-scored overview of what changed.

Instruct the subagent to return **only a structural map under 400 tokens**: changed
symbols, their call sites, affected flows, and coverage gaps. Verbose graph responses
must not appear in the subagent's return value. This map is reused in Phase 2 and, in
**full** mode, Phase 3.

### Phase 1.5 — Load applicable team standards

Use the structural map and changed-file classification to determine the review's
standards scope. Read the repository's `CLAUDE.md` and `AGENTS.md`, then load the
`.cursor/rules/*.mdc` files relevant to the affected layers, flows, symbols, or files.
Changed-file classification is required for changes without meaningful graph symbols,
such as styles, configuration, documentation, or test fixtures.

Build a compact standards brief that preserves each source path and rule heading so
reviewers can cite it precisely. Select rules by the affected area. For example:

- TypeScript, naming, control flow, or helpers: `core-coding-standards.mdc`,
  `functional-patterns.mdc`, and `negative-constraints.mdc`.
- React components, hooks, MUI styling, or accessibility: `react-component-patterns.mdc`,
  `accessibility.mdc`, and `technology-stack.mdc`.
- Tests: `testing-strategy.mdc` and `technology-stack.mdc`.
- Recoil state or selectors: `state-management.mdc`.
- APIs or error handling: `error-handling-patterns.mdc` and relevant ADRs in `docs/adr/`.
- File/folder organization: `project-structure.mdc`, `modularization-conventions.mdc`,
  and ADR-0012 when applicable.

Include an applicable rule only when it governs the changed behavior. The standards
brief must tell reviewers to report each standards breach with its source path,
rule heading, and affected file/line. Treat repository guidance as authoritative over
general preferences; do not report a finding that the loaded standards explicitly permit.

### Phase 2 — Claude review

Using the structural map and Phase 1.5 standards brief, produce graph-informed
findings grouped by risk (high/medium/low). Note which findings the graph surfaced
vs. which came from reading the diff. Cite every team-standards finding as
`<repository-relative standards source>: <rule heading> — ~/dev/client/<affected file>:<line>`.

**Reviewer mindset:** The bar is not "does it work." Look for structural regressions
and missed call-site / contract migrations. Prefer the implementation that makes the
code feel _inevitable_ — boring, direct, maintainable. Be suspicious of clever.

When ponytail is on, do not write Phase 2 findings that are really `delete` / `yagni` /
`shrink` cuts — those go in Phase 2.5 only. If ponytail is off, keep those cuts here.

For each meaningful change, also check:

- **Code judo**: is there a restructuring that _deletes_ whole branches or layers
  rather than just rearranging them? If you see a path to delete complexity rather
  than rearrange it, push hard for that path.
- **Spaghetti**: new ad-hoc conditionals bolted onto unrelated flows? Call out changes
  that make the surrounding code harder to reason about, even if they technically work.
- **Layer boundary**: feature logic leaking into shared/general-purpose paths? Push
  code toward the right module instead of normalizing architectural drift.
- **Canonical helpers**: bespoke one-off where an existing utility already does this?
- **Abstraction quality**: thin wrappers or identity pass-throughs that add indirection
  without buying clarity? Be skeptical of generic mechanisms that hide simple
  data-shape assumptions.
- **Orchestration**: independent async work serialized for no reason, or related updates
  left non-atomic when a cleaner structure is obvious? Flag avoidable orchestration
  complexity that makes the implementation more brittle.
- **Permanent debt**: "temporary" branching that is likely to become permanent debt?

If ponytail is on, run Phase 2.5 next. In **slim** mode, stop after Phase 2 (and 2.5 if on) and emit the slim output format. Do not start Phase 3.

### Phase 2.5 — Ponytail review (overlay only)

Run only when the ponytail overlay is on. Follow `ponytail-review` exactly on the
same scoped diff. Prefix locations with `~/dev/client/<relative-path>` so links click.
List cuts; do not apply them. End with `net: -<N> lines possible.` or `Lean already. Ship.`

### Phase 3 — Codex independent pass (full mode only)

Dispatch an independent review to Codex via the companion script. Codex runs in its
own runtime and CANNOT query the graph, so pass the Phase 1 structural map and the
Phase 1.5 standards brief as focus text. Use the `codex:rescue` skill if available,
or invoke the script directly:

```
node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" adversarial-review --wait \
  --base <base-branch-or-HEAD> --scope <working-tree|branch> \
  "<paste the structural map and standards brief: changed symbols, call sites, affected flows, coverage gaps, applicable rules with source citations>"
```

- **local** → `--scope working-tree` and `--base HEAD`
- **branch** → `--scope branch` and Phase 0 `<base-branch>` (never hardcode `stage`)
- `--wait` blocks for the result; use `--background` + `status`/`result` for large diffs.
- Default to `--effort medium`. Only escalate to `--effort high` when the diff is large
  or touches high-risk areas (auth, API contracts, state management).
- Leave `--model` unset.
- Require Codex to cite every team-standards finding as
  `<repository-relative standards source>: <rule heading> — ~/dev/client/<affected file>:<line>`.

### Phase 4 — Reconcile (full mode only)

Merge both reviews into one verdict:

- **Confirmed (both found it)** — highest confidence, lead with these.
- **Claude-only** / **Codex-only** — report each, note which reviewer found it.
- **Conflicts** — where the two disagree, surface both positions for human judgment;
  do not silently pick a side.

### Output Format

**Slim mode** (single reviewer; label the report as slim / Claude-only):

- Findings (high → low)
- Ponytail (only if overlay on)
- Team standards adherence, including cited source rule(s) for every violation
- Test coverage status
- Overall merge recommendation

**Full mode:**

- Confirmed findings (high → low)
- Single-reviewer findings, attributed
- Conflicts requiring human judgment
- Ponytail (only if overlay on)
- Team standards adherence, including cited source rule(s) for every violation
- Test coverage status
- Overall merge recommendation

Review against the Phase 1.5 standards brief. If no standards violation is found, state that the changed areas were reviewed against the applicable team guidance rather than claiming the full repository is compliant.

**File links:** Always format file references as `~/dev/client/<relative-path>:<line>` so they are clickable in the terminal. Example: `~/dev/client/src/pano360/hooks/useFoo/useFoo.ts:46`

**Approval bar:** Correctness alone is not enough. Treat these as presumptive blockers:

- Missed call-site or contract migration
- Clear structural regression — or, if ponytail is **off**, a plausible path that deletes a whole layer/branch of complexity
- Diff pushes a file past 1k lines without strong justification
- New spaghetti growth (ad-hoc conditionals in unrelated flows)
- Feature logic scattered across shared/general-purpose paths
- Unnecessary wrapper or abstraction that makes the design more indirect
- Refactor that moves complexity around without reducing it — same number of concepts,
  just rearranged

When ponytail is **on**, put delete/shrink/yagni blockers in the ponytail section instead of repeating them here.

## Token Efficiency Rules

These exist so Phase 1 graph payloads stay in the Explore subagent and only a compact map reaches the reviewer.

- Start graph work with `get_minimal_context(task="<your task>")`.
- Do **not** force `detail_level="minimal"` on impact queries (`callers_of`, `tests_for`, `get_affected_flows`, `get_impact_radius`, `detect_changes`). Use the tool default / `standard` so call sites and flows are actually in the map. Minimal is only for a retry if a single response is too large to summarize.
- Keep Phase 1 to ≤5 graph tool calls. The subagent still returns a structural map under 400 tokens (in **full** mode that map must also fit in the Codex focus-text argument).
