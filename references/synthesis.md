<!-- review-council · v1.0.0 -->

# Verification and synthesis: one ranked review

The personas return independent reports. Two stages turn them into a review: kill the false
positives, then reconcile what survives. Skipping the first stage is what makes multi-reviewer
setups notorious for volume over signal.

## Stage 1: adversarially verify every Major

A Major finding is a claim that something is wrong. Before it reaches the user, it has to
survive an attempt to refute it. Minors skip this, they are not worth the spend.

Dispatch one skeptic per Major, in parallel, fresh context:

```
READ-ONLY. Your job is to REFUTE this review finding, not to confirm it.

Finding: <the one-liner, with its cited repo/path:line>
Reviewer's reasoning: <the finding's paragraph from the report>
Diff file: <path>        Repo you may read: <path>
Briefing: <path>

Open the cited code and the code around it. Try to show the finding is wrong: the guard exists
elsewhere, the path is unreachable, the caller already validates, the framework handles it, the
type makes it impossible, the value cannot be null here, the test does cover it, the repo's
convention makes it correct.

Default to REFUTED when you cannot show the problem is real. "Might" and "could in principle"
are refutations, not confirmations: a finding that needs a hypothetical to matter is not Major.
<For a Security finding only, replace the paragraph above with:>
A latent exposure is the finding itself: a secret or token that could reach a log, a response,
or storage is CONFIRMED if a plausible path exists in the code, even if nothing has triggered
it yet. Refute it only by showing the path cannot happen (the value is redacted, the sink is
never reached, the field is never serialized).

RETURN: CONFIRMED or REFUTED, one sentence of reasoning, and the repo/path:line you read.
```

For a "no test covers this" Major, a skeptic that runs code beats one that reads it. Add this
to that skeptic's prompt:

```
Check the claim by running it. Create a throwaway worktree OUTSIDE the repo
(git -C <repo> worktree add <scratchpad>/mutant <head-sha>; for a dirty tree, then apply the
pinned diff there). Break the line the finding says is untested (invert the condition, return
early, drop the call), run only the relevant tests, and record the result. If a test fails,
the gap is not real: REFUTED. If every test passes, CONFIRMED. Then remove the worktree
(git -C <repo> worktree remove --force <scratchpad>/mutant). Never touch the user's checkout.
If the tests cannot run there (dependencies missing, no test command), fall back to reading
the tests and say that the check was not executed.
```

Then:

- **REFUTED** drops out of the review entirely, or falls to Minor if there is a residual point.
- **CONFIRMED** goes to the adjudicator with the skeptic's note attached.
- For a high-stakes change, run three skeptics per Major and keep it only on a majority
  CONFIRMED. For an ordinary change one skeptic is enough.

Record what was refuted and why. It belongs in the report as a short list, because "these five
things were checked and are fine" is useful to the author, and the recorded reason lets a later
round see whether it still holds instead of re-litigating the same point.

## Stage 2: adjudicate

Dispatch one adjudicator, fresh context, most capable model, given every report path, the
confirmed Majors with their skeptic notes, the diff file, and the briefing. It does not collate,
it decides:

1. **Merge duplicates.** Several lenses hitting one issue is ONE finding with the lenses noted,
   not three findings. Note the convergence, it is real signal.
2. **Resolve splits.** Where one persona flags and another clears the same code, re-read the
   cited lines and issue a cited verdict. Decide, do not average.
3. **Rank.** Majors first, ordered by blast radius, each with a one-line confidence note.
4. **Aggregate the nits.** All Minors of the same kind collapse to one line with a count and
   file list ("naming inconsistency in 4 files: a.ts:12, b.ts:44, ..."). A review that
   enumerates 40 style nits will not be read, and burying two real Majors among them is worse
   than not reviewing at all.
5. **Flag intent conflicts, do not resolve them.** A finding that contradicts what the stated
   intent mandates is the user's call. Surface the finding and the mandating text side by side.

## The report

Return it inline. For a large council or a long session also write a copy to the scratchpad so
it survives compaction.

1. **Verdict.** One of CLEAN / MINOR ONLY / MAJOR FINDINGS, plus two sentences: what the change
   does and whether it is safe to merge as written.
2. **What is under review.** Base SHA, head or dirty-tree note, file and line counts, and the
   council that ran. If the working tree was dirty, state that the findings refer to the pinned
   diff.
3. **Major findings.** Each with `repo/path:line`, what is wrong, what it breaks, which lenses
   raised it, and whether a skeptic confirmed it. Fold every lens's view of the issue into the
   one entry: the path that breaks (Correctness), the other sites that need the same fix
   (Sweeper), what the skeptic checked and ruled out, and what the Steelman says to keep. End
   with one suggested direction for the fix, not a patch. The author should not need to open
   five reports to understand the issue or how to fix it.
4. **Missed sites.** The Sweeper's confirmed gaps, kept as their own section because they are
   outside the diff and a reader will otherwise assume they are noise.
5. **Your suspicions.** Only when the Suspicion Investigator ran: each suspicion with its
   verdict (CONFIRMED / REFUTED / UNCLEAR) and the cited evidence. Confirmed ones that are
   Major also appear, ranked, under Major findings. Note when another lens found the same issue
   on its own without seeing the suspicion; that is independent confirmation.
6. **The case for the change.** The Steelman's summary, in its own words. It goes near the top
   of the detail, not in a footnote, so the author reads a review and not a verdict.
7. **Genuine disagreements.** Where two personas reached opposite conclusions on the same code,
   both positions and how the adjudicator resolved it, with the cited lines. A council that
   never disagrees was not diverse.
8. **Checked and fine.** The refuted findings, one line each with the reason it was dismissed.
   Fairness, and a later round can check whether the reason still holds.
9. **Minors.** Aggregated, capped at roughly ten lines.
10. **Not reviewed.** What the council did not cover: files excluded from the diff, a sibling
   repo not checked out, a lens dropped and why, untracked files that may still be missing.
   Silence here reads as coverage that did not happen.

## Rounds

Each round after fixes finds less than the one before, so decide deliberately whether to run
another.

- **While a round still confirms Majors**, run another round on the fixes: pin the new diff
  against the same base, and reuse the council unless the fixes changed the risk profile.
- **Pass the previous "checked and fine" list** into the new briefing. A point that was
  refuted comes back only if the code it relied on changed or the reason no longer holds.
- **Once a round turns up only Minors**, stop the council. Give the remaining fixes a
  targeted single-reviewer pass over just the changed lines, and say in the report that the
  council has converged.

## Rules

- **Evidence over assertion.** Every Major traces to a cited `file:line`, or it is marked
  JUDGMENT and ranked below the verified ones.
- **Do not inherit a persona's error.** If a reviewer over-claimed and a skeptic refuted it, the
  report follows the evidence, not the loudest reviewer.
- **Do not soften.** If the change should not merge, say so. If it is fine and the council found
  only taste, say that too, plainly, and do not pad the report to justify the run.
- **No comments as remedies.** A finding is never resolved by suggesting a code comment. The
  remedy is a rename or a restructure.

## Then offer next actions, do not take them

Offer to hand the confirmed Majors to an implementation flow, or to draft inline PR comments
with the exact anchor for each. Do not post a comment, approve, request changes, push, or merge
without an explicit go-ahead. Posting to a PR is publishing.
