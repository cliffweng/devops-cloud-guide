---
title: "03. CI with GitHub Actions"
layout: default
nav_order: 4
---

# CI with GitHub Actions
{: .no_toc }

*~8 min read*

**🎯 Interview frequent**

## Why it matters

"Walk me through what happens when you push code" is one of the most common warm-up questions in SWE/SRE/platform interviews, and the honest answer for almost every modern team is "a CI pipeline runs." GitHub Actions is the most common place undergrads first meet CI/CD, because it's free, lives right next to your repo, and doesn't require standing up a separate Jenkins box.

## Core concepts

- **A workflow is a YAML file that reacts to repo events.** It lives in `.github/workflows/*.yml` and defines triggers (`on: push`, `on: pull_request`, `on: schedule`) plus one or more **jobs** to run when triggered.
- **Jobs run on runners — fresh, ephemeral VMs.** Each job (unless you say otherwise) starts from a clean GitHub-hosted VM (`runs-on: ubuntu-latest`), so nothing persists between runs unless you explicitly cache or restore it. This is why a build that "worked yesterday" can fail today — the environment is rebuilt from scratch every time.
- **Steps are the unit of work inside a job.** A step either runs a shell command (`run: npm test`) or invokes a reusable **action** (`uses: actions/checkout@v4`) — the marketplace of prebuilt actions is why you rarely write raw shell for common tasks like checking out code or setting up a language runtime.
- **Jobs run in parallel by default; use `needs` to sequence them.** If your pipeline has "lint and test in parallel, then deploy only if both pass," you express that with `needs: [lint, test]` on the deploy job — this is a common actual-pipeline-design interview question.
- **Secrets are injected, never hardcoded.** API keys, deploy credentials, and tokens go into repo/org **Settings → Secrets**, referenced as `${{ secrets.MY_KEY }}` in the workflow — this keeps them out of the YAML file (which is version-controlled and visible to anyone with repo read access) and out of build logs (GitHub automatically masks known secret values in log output).
- **CI (continuous integration) is the automated build+test on every push/PR** — it answers "does this change break anything." **CD (continuous delivery/deployment)** extends the same pipeline to actually ship the artifact — delivery stops at "ready to deploy, needs a human click"; deployment goes all the way to production automatically. Which one you have is a deliberate choice, not a technicality.
- **Status checks gate merges.** Branch protection rules can require a workflow to pass before a PR is mergeable — this is the actual enforcement mechanism behind "main is always deployable" from [Git workflows for teams](../02-git-workflows-teams/).
- **Caching and matrix builds keep CI fast.** `actions/cache` persists dependencies (e.g., `node_modules`) between runs so you're not reinstalling from scratch every time; a `strategy: matrix` runs the same job across multiple versions/OSes in parallel (e.g., testing against Node 18 and 20) instead of writing duplicate jobs.

## Mental model

```
git push / open PR
        |
        v
  [ trigger: on: push/pull_request ]
        |
        v
  job: lint  -----\
  job: test  ------+---- needs: [lint, test] ----> job: deploy
  (parallel)                                          |
                                                       v
                                          only runs if both passed,
                                          uses secrets.DEPLOY_KEY,
                                          ships to staging/prod
```

Think of a workflow file as a recipe that GitHub re-cooks from scratch, in a brand-new kitchen, every single time — nothing survives between runs except what you explicitly cache, and nothing is trusted except what you explicitly pass in as a secret.

## Interview questions

1. **What's the actual difference between continuous integration and continuous deployment, and why would a team deliberately stop at "delivery" instead of full deployment?**
   Answer: CI is the automated build-and-test step that runs on every change to catch breakage early; continuous delivery means the pipeline produces a deployable artifact and stops short of shipping it automatically, requiring a human approval step, while continuous deployment ships it all the way to production with no human in the loop. Teams stop at delivery when the blast radius of a bad deploy is high enough that they want a deliberate human gate — e.g., regulated environments, or a product where a bad release directly costs revenue.

2. **Your test job and lint job take 8 minutes combined, run sequentially, on every PR. How would you speed that up without removing any checks?**
   Answer: Run them as separate jobs instead of sequential steps in one job — GitHub Actions runs jobs in parallel by default (unless you add `needs`), so lint and test would run concurrently, cutting wall-clock time roughly to the slower of the two rather than the sum. Adding dependency caching (`actions/cache`) for whichever job installs packages is the next lever if install time dominates.

3. **Why shouldn't you put an API key directly in a workflow YAML file, even in a private repo?**
   Answer: Anyone with read access to the repo can read the YAML (and it lives forever in git history even if you remove it later), and it would also appear unmasked in workflow run logs. GitHub Secrets store the value outside the repo, inject it only at runtime as an environment variable, and automatically mask any log line that contains the exact secret value.

4. **You want "deploy" to run only after both "build" and "test" succeed. How do you express that, and what happens if "test" fails?**
   Answer: Add `needs: [build, test]` to the deploy job's definition. If `test` fails, `deploy` is skipped entirely by default — `needs` means "wait for these and only proceed if they succeeded," not just "wait for these to finish."

5. **A workflow that passed yesterday is failing today with no code changes. What are likely causes, given that GitHub-hosted runners start from a clean VM each time?**
   Answer: A dependency's upstream version changed (an unpinned package version resolved to a new release with a breaking change), the runner image itself was updated (GitHub periodically updates `ubuntu-latest`'s installed tool versions), or an external service/API the pipeline depends on changed or had an outage — since nothing persists between runs, "it worked before" guarantees nothing about today's fresh environment.

6. **How would you design a pipeline that runs unit tests on every PR but only deploys to production on merges to `main`?**
   Answer: Use different triggers per job or workflow: `on: pull_request` for the test/lint job (runs on every proposed change, gates the merge via branch protection), and a separate `on: push` trigger scoped to `branches: [main]` for the deploy job (fires only once code actually lands on `main`) — optionally with an `environment:` requiring manual approval if production deploys need a human gate.

## Watch

- [How to use GitHub Actions | GitHub for Beginners](https://www.youtube.com/watch?v=BQrohJ3PT7I) — GitHub. Official walkthrough of workflows, jobs, and triggers from the GitHub team itself.

## Further reading

- [GitHub Actions docs: Understanding GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions) — official concepts reference (workflows, jobs, runners, actions).
- [GitHub docs: Encrypted secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions) — how secrets are stored, injected, and masked in logs.
