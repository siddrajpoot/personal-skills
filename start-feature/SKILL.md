---
name: start-feature
description: Use when starting feature, epic, bug, triage, architecture, or review work and the user wants help choosing the right Matt Pocock skill or workflow.
disable-model-invocation: true
---

# Start Feature

Choose the correct Matt Pocock flow before creating files or invoking another skill.

**REQUIRED SUB-SKILL:** Invoke `/i-have-adhd:i-have-adhd` before starting the questionnaire. Its response-shaping rules remain active for the rest of the session unless the user turns them off.

**REQUIRED BACKGROUND:** Read `/mattpocock-skills:ask-matt`.

## Questionnaire

Use the structured question tool when available. Ask one question at a time.

### 1. What kind of work is this?

Offer:

- Shape or build a feature
- Diagnose a bug or regression
- Triage incoming requests
- Improve codebase architecture
- Review existing changes

Route non-feature work directly (merge and rebase conflicts have no dedicated skill in 1.3; handle them without one):

| Need                         | Recommendation                                   |
| ---------------------------- | ------------------------------------------------ |
| Bug, regression, or flake    | `/mattpocock-skills:diagnosing-bugs`             |
| Incoming raw requests        | `/mattpocock-skills:triage`                                        |
| Codebase health              | `/mattpocock-skills:improve-codebase-architecture`                 |
| Existing branch or PR review | `/mattpocock-skills:code-review`                 |

### 2. For feature work, how large or uncertain is it?

Offer:

- Small and settled; should fit one session
- Multi-session feature with a visible path
- Large or foggy epic where the path is unclear
- A design question needs a runnable answer
- Unsure

Recommend:

| Shape             | Flow                                                                                                  |
| ----------------- | ----------------------------------------------------------------------------------------------------- |
| Small and settled | `/mattpocock-skills:grilling` → `/mattpocock-skills:implement`                                                          |
| Multi-session     | `/mattpocock-skills:grilling` → `/mattpocock-skills:to-spec` → `/mattpocock-skills:to-tickets` → fresh `/mattpocock-skills:implement` session per ticket, or `/mattpocock-skills:implement-spec` for the whole spec in one run |
| Large or foggy    | `/mattpocock-skills:wayfinder` → `/mattpocock-skills:to-spec` → `/mattpocock-skills:to-tickets`                                                             |
| Runnable unknown  | `/mattpocock-skills:handoff` → `/mattpocock-skills:prototype` → `/mattpocock-skills:handoff` back                                         |
| Unsure            | Start with `/mattpocock-skills:grilling`; reassess before implementation                              |

## Confirmation gate

Present:

1. Recommended flow.
2. Why it fits the stated size and goal.
3. Whether a local feature workspace will be created.
4. The next skill to invoke.

Wait for confirmation. Do not invoke the next skill or scaffold files before approval.

## Multi-session feature workspace

After the user confirms a multi-session feature or epic, scaffold the workspace before invoking the first recommended skill:

1. Read `CLAUDE.local.md` and `.claude/personal/agents/issue-tracker.md`.
2. Ask for or infer the feature slug and parent epic/ticket. These are setup questions after route approval, not part of the routing questionnaire.
3. Check `.claude/personal/scratch/<feature-slug>/`.
4. If it exists, offer to resume it; never overwrite it.
5. Otherwise create:

```text
.claude/personal/scratch/<feature-slug>/
├── tech-spec.md
├── handoff.md
└── issues/
```

Copy:

- `.claude/personal/templates/feature-tech-spec.md` → `tech-spec.md`
- `.claude/personal/templates/feature-handoff.md` → `handoff.md`

`/mattpocock-skills:to-tickets` later writes one ticket per file under `issues/` using `.claude/personal/templates/local-ticket.md`. Jira remains the source of truth.

When writing the "Jira ticket contents" section of any local ticket, run the text through the `humanizer` skill before saving. Bullet points are fine in ticket descriptions and testing notes.

Before ending planning, update `handoff.md` from its template. Fill the `Start next conversation` block with `/continue-feature` and the exact handoff path. Do not leave placeholders in that block.

## Continue an existing feature

Use `/continue-feature <handoff-path-or-feature-name>` to resume the next local ticket or refresh the handoff after verified implementation. Do not duplicate that lifecycle in this skill.

## Guardrails

- Small tasks, bugs, and reviews do not get a feature workspace.
- After verified implementation, offer `/mattpocock-skills:retro`.
- Keep grilling, spec, and ticket generation in one context when possible.
- Start each `/mattpocock-skills:implement` ticket in fresh context.
- All approval and side-effect gates in `CLAUDE.local.md` still apply.
