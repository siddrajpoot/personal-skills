---
name: review-copilot-comments
description: >-
  Reviews GitHub Copilot PR review comments one at a time. Classifies each as
  extremely cautious vs a real issue, restates the suggestion, offers 1–2
  alternatives only when needed, and shows what a fix should look like.
  Use when the user invokes /review-copilot-comments, asks to triage Copilot
  PR comments, or wants help deciding which Copilot review suggestions to take.
disable-model-invocation: true
---

# Review Copilot comments

Read and apply `~/.agents/skills/i-have-adhd/SKILL.md` for the rest of this session (same as `/i-have-adhd`). Do not dump that skill's rules at the user.

Triage GitHub Copilot review comments on a PR. One comment per turn. Do not batch.

## Resolve the PR

If the user gave a PR number, URL, or branch name, use that.

If they gave none, use the current checkout:

```bash
gh pr view --json number,url,title,headRefName,baseRefName
```

If that fails, stop. First line: the exact command to run, or "this branch has no PR."

## Fetch Copilot comments only

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate
```

Also pull review-level comments if the thread lives on the review, not the diff:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews --paginate
```

Keep a comment if the author looks like Copilot (`Copilot`, `copilot-pull-request-reviewer`, login/name containing `copilot`, case-insensitive). Drop human comments unless the user asked for those too.

Sort by file path, then line. Skip resolved threads if the API marks them resolved.

If there are zero Copilot comments, say that and stop.

## First turn only

State, in this order:

1. PR number, title, branch
2. How many Copilot comments remain
3. Start comment 1 of N (do not preview the others)

Time estimate: about 2–4 minutes per comment.

## Every comment (one per turn)

Read the file and surrounding diff before judging. Do not argue from the comment text alone.

### Classify

Pick **one**:

| Label | Use when |
| --- | --- |
| **Real issue** | A concrete failure is plausible: wrong behavior, race, leak, broken type/contract, security, a11y break, test that would fail, data loss. |
| **Extremely cautious** | Speculative, style-only, defensive for an impossible path, restates existing guards, or Copilot cannot see repo context. |

If mixed, pick the stronger claim and say why the other half is noise.

### Output shape (ADHD)

First line: the action for *this* comment (`Apply Copilot's patch`, `Skip`, or `Use alternative 1`).

Then:

1. **Comment** — `path:line` plus Copilot's point in one or two sentences. No quote dump.
2. **Verdict** — `Real issue` or `Extremely cautious`, plus one sentence of why.
3. **Copilot's suggestion** — their patch if present; otherwise "no patch, only prose."
4. **Alternatives** — zero, one, or two. Include only if Copilot's patch is wrong, incomplete, or fights repo patterns. Skip the section when Copilot's patch is fine or the comment should be ignored.
5. **What a fix looks like** — smallest change: file, function, before/after or a 5–15 line snippet. If verdict is skip, write "No code change."
6. **Your call** — one question only: apply Copilot / apply alternative / skip. Wait.

Do not start the next comment until the user answers that question (or says skip/next).

### After they answer

One line: what you will do (edit files, or record skip). Then do it if they asked to apply. Then:

`Comment K of N done. Next: comment K+1 — say "next".`

When K = N: `All Copilot comments done.` List skips vs applies in at most five bullets.

## Do not

- Dump every comment in one message
- Implement all fixes up front
- Treat "consider" / "you might want" as a bug by default
- Invent Copilot text; fetch it
- Repeat the ADHD rules to the user
