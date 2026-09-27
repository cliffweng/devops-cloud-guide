---
title: "02. Git workflows for teams"
layout: default
nav_order: 3
---

# Git workflows for teams
{: .no_toc }

*~7 min read*

**Interview occasional**

## Why it matters

You already know `git add`/`commit`/`push` from solo projects. The moment a second person touches the same repo, the game changes: branches collide, someone force-pushes over someone else's work, and "just commit to main" stops being an option. Internships and any real side-project team expect you to know branching conventions, code review etiquette, and how to not nuke a coworker's history.

## Core concepts

- **`main` should always be deployable.** Nobody commits straight to `main` on a team repo — work happens on short-lived feature branches, merged in via pull request (PR) once reviewed and (ideally) passing CI.
- **Trunk-based development vs. GitFlow.** Trunk-based: small, frequent branches merged back into `main` within a day or two, favored by fast-moving teams and paired with feature flags for anything not ready to ship. GitFlow: longer-lived `develop`/`release`/`hotfix` branches with more ceremony, more common in software with scheduled releases (e.g., versioned mobile apps). Most startups and side projects lean trunk-based because it minimizes merge pain.
- **Rebase vs. merge is a "keep history clean" tradeoff, not a correctness one.** `git merge` preserves exact history with a merge commit; `git rebase` replays your commits on top of the latest `main` for a linear history. The team rule that matters: **never rebase a branch other people are also pulling from** — rebase rewrites commit hashes, so anyone downstream gets a broken history. Rebasing your own not-yet-pushed feature branch is fine.
- **PRs are a communication tool, not just a merge mechanism.** A good PR description says *what* changed and *why* (not just what the diff shows), links the issue/ticket, and is small enough that a reviewer can actually reason about it — a 2,000-line PR gets rubber-stamped, not reviewed.
- **Merge conflicts happen when two branches edit the same lines.** Git can't guess intent, so it stops and asks you to resolve it by hand — resolving means reading both versions and deciding (or combining) what the final code should be, then committing the result. This is normal, not a sign you did something wrong.
- **Force-pushing (`git push --force`) rewrites remote history** and can silently delete a teammate's commits if their work isn't in your local history. `--force-with-lease` is the safer version — it refuses to overwrite if the remote has commits you don't have locally, which is why most teams require it over plain `--force`.
- **`.gitignore`, not "don't commit that file, I'll remember."** Secrets, `node_modules`, build artifacts, and IDE config belong in `.gitignore` from the first commit — once a secret is committed, it's in history forever unless you rewrite the repo, so treat any committed credential as compromised and rotate it.
- **Branch protection rules** (require PR review, require CI to pass, block force-push to `main`) are how teams enforce these norms automatically instead of relying on everyone remembering the etiquette.

## Mental model

```
main -----o---------------------o-----------o---------> (always deployable)
            \                   ^           ^
             \                 /           /
   feature/x  o---o---o-------/           /
                        (PR + review + CI, then merge)
                                          /
   feature/y            o-------o-------/
                        (small, short-lived, merged fast)
```

Think of `main` as the one copy everyone trusts blindly — every branch exists to protect it, and every PR is the toll booth that keeps bad or unreviewed code from getting in.

## Interview questions

1. **Why do teams avoid committing directly to `main`?**
   Answer: `main` is what gets deployed (often automatically via CI/CD), so an unreviewed or untested commit there can break production immediately. Feature branches plus PRs create a checkpoint — code review and CI — before anything reaches the branch that matters.

2. **What's the actual difference between `git merge` and `git rebase`, and when would using rebase be dangerous?**
   Answer: Merge creates a new commit that ties two histories together, preserving exactly what happened; rebase replays your commits on top of a new base, rewriting their hashes to produce linear history. Rebasing is dangerous on a branch other people have already pulled from, because rewriting commit hashes breaks their local history's connection to the remote — they'll hit confusing conflicts or need to force-pull to recover.

3. **You need to force-push to update a branch after a rebase. Why would a team require `--force-with-lease` instead of `--force`?**
   Answer: Plain `--force` overwrites the remote branch unconditionally, silently destroying any commits a teammate pushed there since you last fetched. `--force-with-lease` checks that the remote hasn't changed since your last fetch and refuses to push if it has, preventing you from accidentally clobbering someone else's work.

4. **You accidentally committed an API key. What's the right response, and why isn't `git rm` + a new commit enough?**
   Answer: The key is still readable in the commit history even after a later commit deletes the file, so anyone who clones the repo (or already did) can find it — the correct response is to treat the key as compromised and rotate/revoke it immediately, then optionally scrub history (e.g., `git filter-repo` or BFG) if the repo is public, but rotation is the step that actually closes the security hole.

5. **What makes a pull request "good" from a reviewer's perspective, beyond "the code works"?**
   Answer: It's small and scoped enough to reason about in one sitting, has a description explaining *why* the change was made (not just restating the diff), and ideally passes CI before review starts so the reviewer's time goes to design/logic feedback instead of catching things a linter or test suite would've caught.

6. **How would you resolve a merge conflict, and what's actually happening under the hood when Git reports one?**
   Answer: Git found two branches with different changes to the same lines (or the same file deleted on one side and edited on the other) and can't automatically pick a winner, so it marks the conflicting regions in the file with `<<<<<<<`/`=======`/`>>>>>>>` markers and pauses the merge. You resolve it by editing the file to the version you want (keeping, combining, or rewriting both sides), removing the markers, then `git add` and continue the merge/rebase.

## Watch

- [3 Git Workflows Every Developer Should Know (And When to Use Each)](https://www.youtube.com/watch?v=GQQqf-C2ha4) — TechWorld with Nana. Covers trunk-based, GitFlow, and forking workflows with concrete team scenarios.

## Further reading

- [Atlassian: Comparing Git workflows](https://www.atlassian.com/git/tutorials/comparing-workflows) — a clear comparison of centralized, feature-branch, GitFlow, and forking workflows.
- [GitHub docs: About pull request merges](https://docs.github.com/en/pull-requests/collaborating-on-pull-requests-with-code-quality-features/about-merge-methods-on-github) — merge vs. squash vs. rebase merge strategies on GitHub specifically.
