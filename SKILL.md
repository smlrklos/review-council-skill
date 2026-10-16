---
name: review-council
version: 1.0.0
description: >
  Use when asked to review, critique, or find problems in CODE: a GitHub pull request (by
  number, URL, or branch), a local diff, uncommitted working-tree changes, a branch against
  its base, or a named set of changed files. Triggers on "review this PR", "review my
  changes", "code review this", "look at this diff", "what is wrong with this branch",
  "find the bugs in this change", "review PR 54", "give this a thorough review". Also when a
  code review should weigh several independent reviewer perspectives rather than one pass
  (correctness, security, tests, performance, concurrency, missed call sites, API consumers,
  migrations, on-call, maintainability, YAGNI, steelman). Not for reviewing a plan, RFC,
  spec, design doc, or a ticket's description, which is a proposal review and not a code
  review. Not for writing code, applying the fixes, or authoring the PR title and body. Not
  for checking that a change actually runs. Not for taking a Jira ticket from description
  through implementation, which is a ticket-driven flow with its own review step.
---

# Review Council

## Overview

Review a code change by running a panel of perspective-diverse reviewers (personas) as
parallel subagents over one pinned diff, then confronting their findings into a single ranked
review. Two principles carry the whole thing: **every finding is verified against the code and
cited as `file:line`, never theorized**, and **disagreements between reviewers are resolved on
the evidence, not averaged away**.

This skill reviews. It does not implement, fix, or push. Its output is a ranked review.

## When to use / when not

Use when the deliverable is a judgement about a code change: is it correct, what did it miss,
what will break, is it worth the complexity. Works on a PR, a branch, or dirty working tree,
with or without a ticket.

Do NOT use for:

| Not this | Because |
| --- | --- |
| Reviewing a plan, RFC, spec, design doc, or ticket description | That judges a proposal, not code. There is no diff to pin and no `file:line` to cite. |
| Taking a Jira ticket from description through implementation | That is a ticket-driven flow with its own review step built in. |
| One quick single-pass review | A council is the expensive option. A single reviewer is cheaper and usually enough. |
| Confirming the change actually runs | This council reads code. Its one execution is a skeptic checking a "no test covers this" finding, which runs the suite in a throwaway worktree, never in the user's checkout. |
| Writing the PR title and body | Different job, different conventions. |
| Applying the findings | This skill is read-only by construction. |

Reach for a council when the change is risky, wide, cross-repo, touches a hot path or a trust
boundary, or when an earlier review missed something and you want independent lenses rather
than one more pass by the same lens.

This skill is self-contained. Every persona, prompt, and rule it needs is in this directory,
so it does not depend on any other skill, plugin, or agent being installed.

## Workflow

| Step | Read |
| --- | --- |
| Resolve the target, pin the base, build the briefing | `references/scoping.md` |
| Choose and brief the council | `references/personas.md` |
| Verify findings, then adjudicate into one review | `references/synthesis.md` |

1. **Resolve the review target and pin the base.** Detect whether it is a PR number or URL, a
   branch, the working tree, or a named file set. Record the **base SHA** and write the diff
   to a file in the scratchpad. Everything downstream reads that file, so the review cannot
   drift while the branch keeps moving. The scratchpad is a temporary directory outside the
   repository: nothing this skill writes ever lands in the repo. Reviewers read the local
   branch in place; if the user wants to keep coding while the review runs, suggest they do it
   in a separate worktree so the code under review does not change halfway through.
2. **Establish intent.** What was this change supposed to do? Take it from the ticket AC, the
   PR body, or one sentence from the user. Without a stated intent there is no spec-compliance
   lens and the council can only judge the code on its own terms; say so rather than inventing
   an intent.
3. **Establish repo ground truth.** The real local path, the package manager and test command,
   and the repo's own conventions (its `CLAUDE.md`, lint config, sibling code). A finding that
   fights the repo's existing idiom is usually the reviewer being wrong, not the code.
4. **Write ONE shared briefing file** holding the diff path, base SHA, intent, repo paths,
   and conventions. Every persona reads the same file, so they reason from identical facts and
   the diff never gets pasted into N prompts. Suspicions (the user's, or ones you noticed while
   scoping) do NOT go in it: write them to a separate suspicions file that only the Suspicion
   Investigator reads, so they cannot prime the rest of the council. Templates in
   `references/scoping.md`.
5. **Choose the council.** Pick a non-overlapping subset (always include Correctness,
   Untouched-Areas Sweeper, and Steelman). If a suspicions file exists, pre-select the
   Suspicion Investigator as recommended. Confirm the set with `AskUserQuestion`, then lock
   it. Size it to risk: a small config diff takes 3 to 4 personas, an ordinary feature 5 to 6,
   a cross-repo or security-relevant change 7 to 9. State the model per persona.
6. **Dispatch the council in parallel.** One Agent call per persona (`subagent_type:
   general-purpose`), all in a SINGLE message so they run concurrently with fresh isolated
   context. Never one combined prompt for the whole council. Track with `TaskCreate` /
   `TaskUpdate`, one task per persona, `in_progress` on spawn and `completed` on return. Each
   prompt follows the contract in `references/personas.md`.
7. **Adversarially verify every Major finding** before it reaches the user. A Major that
   survives a skeptic is worth the user's attention; one that does not is noise. See
   `references/synthesis.md`.
8. **Adjudicate into one ranked review.** Merge duplicates, resolve splits by re-reading the
   cited code, rank Major / Minor, aggregate style nits into a single line. Each Major entry
   combines what every lens said about it, so the author gets the failing path, the other
   sites that need the same fix, and a direction for the fix in one place.
9. **Decide whether another round is needed.** After the author fixes the Majors, run another
   round on the fixes while a round still confirms Majors. Once a round turns up only Minors,
   stop the council and give the remaining fixes a targeted single-reviewer pass over just the
   changed lines. See "Rounds" in `references/synthesis.md`.
10. **Offer next actions, do not take outward ones.** Offer to hand the Majors to an
   implementation skill or to draft inline PR comments. Never post a comment, push, or approve
   without an explicit go-ahead.

## Guardrails

- **The diff is a file, with a recorded base SHA.** Never paste a diff into prompts, never let
  a reviewer regenerate it, never review "the current state of the branch" without a pin.
- **Verify before raising.** Every finding cites `file:line` from code the reviewer actually
  opened. A name match is not evidence.
- **Never pre-judge a finding.** Do not tell a reviewer to treat something as minor, to skip a
  category, or to keep the count down. Severity is decided against the evidence, afterwards.
- **No comment-adding recommendations.** The fix for unclear code is a better name or a
  restructure, never a comment. A persona that wants to suggest a comment must suggest the
  rename instead. This is stated in every persona prompt.
- **Read-only by default.** This skill does not edit the repo, push, approve, merge, or post.
  The only code it runs is a test-gap skeptic's mutation check, and that happens in a
  throwaway worktree that is removed afterwards.
- **Respect the repo's conventions over personal taste.** Matching existing style is not a
  finding.

## Common mistakes

- Pasting the diff into every subagent prompt instead of handing over one file.
- Running personas whose findings overlap, so the same issue arrives five times and reads as
  consensus when it is one lens duplicated.
- Dropping the Steelman, after which the review becomes a pile-on and a sound change dies of
  accumulated nits.
- Reviewing only what changed. The Untouched-Areas Sweeper is the lens that catches the missed
  second call site, which is the most common real defect in a change that otherwise looks fine.
- Letting 8 personas emit 40 style nits into the final review. Aggregate them.
- Reporting a Major that no skeptic ever challenged.
- Putting the user's suspicions in the shared briefing. Every reviewer then chases the same
  lead and the lenses collapse into one. Give them to the Suspicion Investigator only.
- Running the full seventeen on a 40-line diff. The council is the expensive option; size it to
  the risk or it stops being worth invoking.
