<!-- review-council · v1.0.0 -->

# Scoping: pin the change and build the briefing

The council is only as good as the briefing, and a code review has one extra requirement a
document review does not: the artifact under review keeps moving. Pin it first.

## 1. Resolve the target and pin the base

| Target | How to capture |
| --- | --- |
| PR number or URL | `gh pr view <n> --repo <org>/<repo> --json title,body,baseRefName,headRefOid,additions,deletions,files` then `gh pr diff <n> --repo <org>/<repo> > <scratchpad>/review.diff`. Record `headRefOid` as the pin. Also read the PR body: it states the author's intent. |
| Branch vs its base | `git -C <repo> merge-base <base> HEAD` to get the base SHA, then `git -C <repo> diff <base-sha>...HEAD > <scratchpad>/review.diff`. Use the three-dot form so unrelated base commits do not appear. |
| Uncommitted working tree | `git -C <repo> diff HEAD > <scratchpad>/review.diff` for tracked changes. Check `git status --short` for untracked files and add them explicitly, they are invisible to `diff` and are a classic blind spot. Pin is `HEAD` plus the note that the tree is dirty. |
| A named set of files | Diff just those paths, and say in the briefing that the review is deliberately narrowed so the sweeper does not report the rest of the change as missing. |

The scratchpad is a temporary directory outside the repository (for example
`$TMPDIR/review-council/<run-id>/`). The diff, the briefing, the suspicions file, and every
report go there. Nothing this skill writes lands in the repo.

Record in the briefing: the **base SHA**, the head SHA or "dirty tree", the file count, and the
line counts. Reviewers read the local branch in place. If the user wants to keep working during
the review, suggest a separate worktree for that work so the checkout under review stays still.
If they keep editing this checkout anyway, the findings refer to the pinned diff, not to
whatever the files say later. Say that out loud in the report.

If the diff is very large (say above ~2000 lines), do not hand the whole thing to every
persona. Split it by area into per-persona diff slices, or narrow the review with the user
first. A reviewer that runs out of attention silently returns a shallow review.

## 2. Establish intent

A code review answers "does this do the right thing", which needs a statement of what the
right thing was. In order of preference:

1. The ticket acceptance criteria (Atlassian MCP `getJiraIssue`), when a ticket exists.
2. The PR body.
3. One sentence from the user, asked directly.

If none is available, record `INTENT: not stated` in the briefing and drop the
spec-compliance persona. Do not invent an intent from reading the diff; that makes the lens
circular, it would only ever confirm that the code does what the code does.

## 3. Establish repo ground truth

- The real local path of every repo the change touches.
- The test and typecheck commands, taken from `package.json` scripts or the repo's own docs,
  so a persona can check whether a test actually covers the new branch.
- The repo's conventions: its `CLAUDE.md`, lint and formatter config, and the nearest sibling
  code to what changed. Personas need this to tell a genuine defect from their own taste.
- For a cross-repo change, the sibling PRs. A change that looks incomplete in one repo is
  often completed in another; give the council the links so it does not report a false gap.

## 4. Write the shared briefing file

Write it to the scratchpad. Keep it factual, and keep suspicions out of it: anything that
looked off, whether the user said it or you noticed it while scoping, goes into a separate
suspicions file (below) that only the Suspicion Investigator reads. A suspicion in the shared
briefing primes every reviewer, and they chase it instead of covering their own lens.

```markdown
# SHARED BRIEFING - Code review of "<title>"

You are one of several independent reviewers, each with a distinct lens, reviewing this
change. Read this file fully, then do YOUR lens's job. You cannot see the other reviewers.
Verify everything.

## What is under review
- Diff file (review THIS, not a summary): <scratchpad>/review.diff
- Base SHA: <sha>        Head: <sha | dirty working tree>
- Scope: <N files, +A / -B lines>
- PR: <org/repo#n and URL, if any>

## Intent (what this change was supposed to do)
<ticket AC, PR body, or the user's sentence. Or: "not stated">

## Repo ground truth (USE THESE PATHS)
- <repo> -> /absolute/local/path
- Tests: <command>        Typecheck: <command>
- Conventions: <path to CLAUDE.md / lint config>. Matching existing repo style is NOT a finding.

## Sibling PRs / cross-repo context
- <org/repo#n>: <what it covers, so you do not report it as missing here>

## Untracked files included in this review
- <path> (new file, not in git yet)

## YOUR DELIVERABLE
<paste the deliverable contract from references/personas.md>
```

## 5. Write the suspicions file (only when there are suspicions)

Collect every suspicion the user stated, plus anything you noticed while scoping. If there are
none, skip this file and drop the Suspicion Investigator. Otherwise write it to the scratchpad
and recommend the Investigator when confirming the council.

```markdown
# SUSPICIONS - leads to verify, NOT conclusions

Only the Suspicion Investigator reads this file. Settle each one on the evidence.

1. <the thing that looked off, phrased as a question>    Source: <user | scoping>
```

## Why a file, not prompts

Fan-out with fresh-context isolation: each persona gets the inputs it needs and none of your
conversation history, and one file keeps every persona on identical facts. It also keeps the
diff out of N prompts, which is the difference between one council run and a context blow-up.
