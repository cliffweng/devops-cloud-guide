---
title: "10. IaC with Terraform basics"
layout: default
nav_order: 11
---

# IaC with Terraform basics
{: .no_toc }

*~8 min read*

**🎯 Interview frequent**

## Why it matters

Clicking through the AWS console to create resources doesn't scale past a weekend project, and it's exactly the habit interviewers probe for with "how do you avoid configuration drift" or "how would you reproduce this environment." Infrastructure as Code (IaC) — writing your infrastructure as version-controlled files instead of console clicks — is table stakes for any team-run cloud environment, and Terraform is the most common tool undergrads meet first because it's cloud-agnostic and has the biggest ecosystem.

## Core concepts

- **You declare the desired end state, not the steps to get there.** A Terraform config says "there should be an S3 bucket named X with these settings" — you don't write imperative steps like "create bucket, then set versioning." Terraform figures out what needs to change to make reality match your declaration.
- **The state file is Terraform's memory of what it actually created.** It maps your config's resources to real-world IDs (this exact ARN, this exact instance ID) — without it, Terraform can't tell the difference between "this resource doesn't exist yet" and "this resource exists but I don't know its ID." Losing the state file effectively means Terraform has amnesia about everything it built.
- **`terraform plan` shows you the diff before anything changes; `terraform apply` executes it.** Plan compares your config against the current state (and, via API calls, current reality) and shows exactly what will be created, changed, or destroyed — reviewing the plan output before applying is the single most important habit for not accidentally deleting a production database.
- **Remote state (not a state file on your laptop) is required the moment more than one person touches the infrastructure.** A local state file means only you have the source of truth, and two people running `apply` from their own local state simultaneously will conflict or silently diverge. Remote backends (an S3 bucket + DynamoDB table for locking is the classic AWS pattern) give everyone the same source of truth and prevent two applies from running concurrently.
- **Providers are what make Terraform cloud-agnostic.** The AWS provider knows how to talk to AWS's APIs; there are separate providers for Azure, GCP, Kubernetes, even SaaS tools like Datadog or GitHub. The core Terraform language is the same regardless of which provider(s) a given config uses.
- **Modules are reusable, parameterized bundles of resources.** Instead of copy-pasting the same "VPC + subnets + route tables" config for every environment, you write it once as a module and call it with different inputs for dev/staging/prod — this is the main way real Terraform codebases avoid duplication.
- **Drift is when reality diverges from what Terraform thinks it created** — someone manually changed a setting in the console, or a resource was modified outside Terraform entirely. `terraform plan` surfaces drift by showing a diff even when you didn't intend to change anything, which is precisely why manual console changes to Terraform-managed resources are a habit teams actively discourage.
- **Terraform doesn't have a native rollback command.** If an `apply` causes a problem, the fix is either applying a previous, known-good version of the config (fixing forward via version control) or manually reverting the specific change — this is different from, say, a deployment rollback, and worth knowing explicitly since it surprises people coming from app-deploy tooling.

## Mental model

```
   .tf files (desired state, version-controlled)
          |
          v
   terraform plan  ---- compares config vs. state vs. real cloud ----> diff shown
          |
          v
   terraform apply ---- executes the diff via provider API calls ---->  AWS/etc.
          |
          v
   state file updated (remote: S3 + DynamoDB lock, shared source of truth)

   config changes -> new plan -> new apply   (repeat; git history = infra history)
```

Think of the `.tf` files as a blueprint and the state file as the building inspector's record of what's actually been built against that blueprint — `plan` is asking the inspector "what would change if we built the current blueprint," and `apply` is actually doing the construction.

## Interview questions

1. **What problem does the Terraform state file solve, and what breaks if it's lost or corrupted?**
   Answer: It maps each resource in your config to the specific real-world object Terraform created for it (exact IDs/ARNs), which is how Terraform knows what already exists versus what needs to be created. If it's lost, Terraform no longer knows those resources are "already managed" — the next `plan`/`apply` may try to recreate resources that already exist (causing naming conflicts or duplicate infrastructure) unless you manually re-import each resource into a fresh state.

2. **Why is `terraform plan` considered a critical safety step rather than an optional nice-to-have?**
   Answer: It shows the exact create/update/destroy diff Terraform is about to execute against real infrastructure before anything actually happens, which is the main defense against an unreviewed config change silently destroying and recreating a resource (e.g., a database, if a change forces replacement rather than an in-place update). Skipping straight to `apply` means finding out about a destructive change only after it's already happened.

3. **Two engineers both run `terraform apply` from their own laptops with local state files, at roughly the same time. What goes wrong, and how does remote state with locking fix it?**
   Answer: Each has their own local view of "what's already been created," so their applies can conflict (both trying to create the same named resource) or silently diverge (each state file thinks it owns a different, incompatible version of reality) — worst case, one apply's changes get lost or overwritten from the other's perspective. A remote backend (e.g., S3 for the shared state file, DynamoDB for a lock) makes both engineers work against the same source of truth and serializes applies so only one can run against that state at a time.

4. **Someone manually changed a security group rule in the AWS console instead of editing the Terraform config. What happens the next time someone runs `terraform plan`?**
   Answer: Terraform detects drift — it compares the actual current state of the resource (queried via the provider) against what the config says it should be, finds a mismatch, and shows a plan to revert the manual change back to what the config declares. This is exactly why manual console edits to Terraform-managed resources are discouraged: the next apply will silently undo them unless the config is updated to match, or the change is deliberately imported/codified.

5. **Why would you extract a "VPC + subnets" configuration into a reusable module instead of writing it separately for dev, staging, and prod?**
   Answer: Without a module, the same resource definitions get copy-pasted three times, so a fix or improvement (e.g., adding a missing tag, fixing a subnet CIDR bug) has to be manually repeated in three places and will inevitably drift out of sync. A module defines the resources once, parameterized by inputs (environment name, CIDR ranges, instance sizes), so each environment calls the same tested code with different values — one source of truth, three consistent outputs.

6. **An `apply` just broke production. Terraform has no `rollback` command — what do you actually do?**
   Answer: Because Terraform's model is "make reality match the current config," the fix is to apply a config that represents the desired good state — typically reverting the offending commit in version control (fixing forward) and running `plan`/`apply` again with that reverted config, rather than looking for some Terraform-native undo. This is why treating `.tf` files as version-controlled and reviewable (PRs, not direct edits) matters as much for infrastructure as it does for application code.

## Watch

- [Terraform explained in 15 mins | Terraform Tutorial for Beginners](https://www.youtube.com/watch?v=l5k1ai_GBDE) — TechWorld with Nana. Covers providers, state, plan/apply, and modules concisely.
- [Terraform in 100 Seconds](https://www.youtube.com/watch?v=tomUWcQ0P3k) — Fireship. Quick-hit visual framing of why IaC and Terraform exist.

## Further reading

- [Terraform docs: State](https://developer.hashicorp.com/terraform/language/state) — official deep dive on what state is and why it exists.
- [Terraform docs: Modules](https://developer.hashicorp.com/terraform/language/modules) — official guide to writing and calling reusable modules.
