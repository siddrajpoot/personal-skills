---
name: continue-feature
description: Use when resuming a multi-session feature from a handoff path or feature name, selecting the next local ticket, or finishing a ticket and preparing the next continuation.
disable-model-invocation: true
---

# Continue Feature

Resume one feature ticket from verified local state, then leave a current handoff for the next session.

**REQUIRED SUB-SKILL:** Invoke `/i-have-adhd:i-have-adhd` before presenting feature state. Its response-shaping rules remain active unless the user turns them off.

**REQUIRED BACKGROUND:** Read `/mattpocock-skills:ask-matt` when the next workflow is unclear.

## Resolve the feature workspace

Accept either a handoff path or feature name after `/continue-feature`.

### Handoff path

1. Resolve the path relative to the repository root.
2. Require a readable `handoff.md`.
3. Use its parent as the feature workspace and require adjacent `tech-spec.md` and `issues/`.
4. Report missing artifacts instead of inventing or scaffolding them.

### Feature name

1. Search only immediate workspaces under `.claude/personal/scratch/`.
2. Normalize case, spaces, and hyphens; prefer an exact slug match, then a clear fuzzy match.
3. Carry a clear match into the resume summary so the user confirms the workspace and frontier together.
4. If multiple matches are plausible, ask the user to choose. If none match, report the available names.

If no argument is provided, ask for a handoff path or feature name.

Never recover a missing workspace from Jira, GitHub, remote branches, or web searches.

## Reconstruct local state

Read:

- `CLAUDE.local.md`
- `handoff.md` and `tech-spec.md`
- The current ticket and candidate tickets under `issues/`
- Local Git status, checkout, diff, and relevant local commits

Do not fetch, pull, query Jira, inspect remote PR state, or perform any external write. Treat plans and unchecked acceptance criteria as intent, not delivered behavior. Implementation and test evidence determine what is complete.

## Select the frontier

Choose the first local ticket whose declared dependencies are complete. Do not rely on filename order alone.

- Treat a dependency as complete only when its implementation is present in the working tree or a reachable local commit and its relevant verification evidence is available.
- If the handoff conflicts with local tickets or Git evidence, identify it as stale.
- If the proposed ticket is blocked, do not invoke `/mattpocock-skills:implement`; present the blocker.
- If several tickets are independently unblocked, recommend one and explain the dependency reason.
- Scope implementation to one ticket.

Select the Matt Pocock skill from the handoff when it remains appropriate. If it is missing or stale, use `/mattpocock-skills:ask-matt` to recommend the exact next skill.

## Confirmation gate

Present:

```markdown
## Resume summary

- Feature: <name and resolved handoff path>
- Checkout: <branch and clean/dirty state>
- Delivered state: <verified concise outcome>
- Next ticket: <path, or blocker>
- Proposed skill: <exact skill>
- Watchouts: <only relevant risks>
```

Wait for confirmation every time. Do not invoke the next skill, edit feature documents, or begin implementation before approval.

## Continue the ticket

After confirmation:

1. Invoke the proposed Matt Pocock skill.
2. Read the selected local ticket plus adjacent `tech-spec.md` and `handoff.md`.
3. Work only the selected ticket.
4. Follow all approval gates in `CLAUDE.local.md`; continuation never authorizes commits, pushes, PR changes, Jira mutations, or stack repairs.
5. Verify delivered behavior before treating the ticket as complete.
6. Offer `/mattpocock-skills:retro` once the last ticket of the feature is verified.

## Finish and prepare the next continuation

After the ticket is verified, compare delivered behavior with `tech-spec.md`.

Check durable changes to product behavior, domain invariants, architecture, interfaces, state ownership, error handling, permissions, analytics, rollout, or testing strategy. Ignore transient implementation details.

If no durable contract changed, state:

```text
No tech-spec changes needed.
```

If the spec differs, present the exact delta:

```markdown
## Proposed tech-spec change

**Section:** <heading>
**Current spec:** <existing statement>
**Observed evidence:** <implemented behavior, test, or approved decision>
**Proposed replacement:** <exact new text>
```

Always wait for approval before editing `tech-spec.md`. If approval is withheld, preserve the existing spec, record the discrepancy under handoff risks, and continue the handoff refresh.

After resolving the drift check, automatically refresh `handoff.md` with:

- Date, current checkout, status, and locally verified stack facts
- Completed ticket outcome and verification
- Approved tech-spec changes by reference, without duplicating the decision
- Next unblocked ticket or explicit blocker
- Current risks, open questions, and evidence
- Existing relevant references, including Jira links, without following or synchronizing them
- Suggested skills by appending newly relevant entries without duplicates; never remove one without explicit user instruction
- A copy-ready `Start next conversation` block

Use this launch block with the exact repository-relative path and no placeholders:

```text
Use /continue-feature for:
.claude/personal/scratch/<feature-slug>/handoff.md
```

Do not update local ticket contents merely to record status. Do not mutate Jira or GitHub as part of finishing the continuation.

## Writing ticket content

When creating or editing a local ticket's "Jira ticket contents" section, run the text through the `humanizer` skill before saving. Bullet points are fine in ticket descriptions and testing notes.

## Guardrails

- Use `/start-feature` for a new feature or missing workspace.
- Never overwrite or scaffold an existing workspace.
- Never bypass an unmet ticket dependency because the user is in a hurry.
- Never silently change stable design to match implementation.
- Keep the handoff current and concise; stable decisions belong in `tech-spec.md`.
