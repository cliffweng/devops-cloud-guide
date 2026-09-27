---
title: "14. Interview patterns"
layout: default
nav_order: 15
---

# Interview patterns (design a deploy pipeline / scale a service)
{: .no_toc }

*~9 min read*

**🎯 Interview frequent**

## Why it matters

This is the capstone: SWE/SRE/platform interviews love two open-ended prompts — "design a CI/CD pipeline for X" and "how would you scale this service" — precisely because they force you to combine everything from this guide (Git workflows, CI, containers, compute choice, storage, networking, IaC, observability, on-call) into one coherent answer under time pressure. Practicing the *flow* of these answers matters more than memorizing any single fact.

## Core concepts

- **Every design answer should start by clarifying requirements, not architecture.** For a pipeline: what's the app (monolith? microservices?), how often does it deploy, what's the team size, is downtime during deploys acceptable? For scaling: what's the current bottleneck (CPU? DB? network?), what's the actual traffic pattern (steady vs. spiky), what's the acceptable latency/availability target? Jumping straight to "we'll use Kubernetes" without asking any of this is the single most common way candidates lose points.
- **A deploy pipeline's shape, end to end**: developer pushes → CI runs (lint, unit tests, build image) → image pushed to a registry, tagged immutably → CD stage deploys to staging → automated/manual gate → deploy to production, ideally with a rollout strategy that doesn't require a full-outage window. Every one of those arrows maps to a topic in this guide — [Git workflows](../02-git-workflows-teams/), [CI](../03-ci-github-actions/), [containers/registries](../05-images-registries-compose/), and the deploy target itself ([Compute](../07-compute-ec2-ecs-lambda/) or [Kubernetes](../11-kubernetes-overview/)).
- **Rollout strategies are a concrete thing to name, not hand-wave.** Rolling deploy (gradually replace old instances with new — what a Kubernetes Deployment or an ECS rolling update does by default), blue/green (stand up an entirely new environment, cut traffic over at once, keep the old one as instant rollback), canary (send a small % of traffic to the new version, watch metrics, ramp up if healthy). Naming one and explaining *why* it fits the stated requirements is a strong signal.
- **"How would you scale this" almost always decomposes into: where's the actual bottleneck, and does scaling need to be vertical or horizontal.** Vertical (bigger instance) is simpler but has a ceiling and usually requires downtime to resize; horizontal (more instances behind a load balancer, see [Networking & HTTPS](../09-networking-https/)) scales further but requires the app to be stateless (or for state to live in a shared store, not on the instance) — naming this constraint explicitly is often the crux of the answer.
- **The database is usually the real bottleneck, not the app servers.** App servers scale horizontally easily; a relational database doesn't, without real work — read replicas for read-heavy load, caching (Redis/Memcached) in front of hot reads, and eventually partitioning/sharding if writes themselves are the bottleneck. Naming "the database is the hard part, here's specifically why" beats a generic "we'd add more servers."
- **Observability and rollback aren't afterthoughts — they're part of the design, not a bonus point mentioned at the end.** A deploy pipeline without a rollback plan (or a scale-up plan without alerting on the resource being scaled) is an incomplete design, and interviewers will often ask "and if this new version is broken, then what?" specifically to check whether you designed for failure or just for the happy path.
- **State your tradeoffs out loud, even unprompted.** "I'd use a canary deploy here because it's read-heavy public traffic and we can detect a bad version from error-rate metrics before it affects everyone — the tradeoff is it's slower to fully roll out than an all-at-once deploy" is a much stronger answer than just "canary deploy" — interviewers are scoring your reasoning, not just your vocabulary.
- **Know the difference between what you'd actually ship for a side project vs. what a large org would build.** A side project might reasonably skip canary deploys and multi-region failover; naming that explicitly ("for a side project at this scale, I'd keep this simple — rolling deploy on ECS, no canary — but if this were handling payment traffic at scale, here's what I'd add") shows judgment, not just knowledge of the fancy version.

## Mental model

```
"Design a deploy pipeline for X"          "How would you scale service Y"
        |                                          |
        v                                          v
  clarify: app shape, deploy freq,          clarify: bottleneck? traffic
  team size, downtime tolerance             pattern? latency/availability target?
        |                                          |
        v                                          v
  push -> CI (test/build/image) ->          identify bottleneck: app tier
  registry -> staging -> gate ->            (horizontal, stateless) vs.
  prod, with a named rollout                database (replicas/cache/
  strategy (rolling/blue-green/canary)      partitioning) vs. network (CDN/LB)
        |                                          |
        v                                          v
  + observability (know it broke)           + observability (know you're
  + rollback plan (undo it fast)              approaching the new limit)
```

Think of both prompts as the same skill wearing different clothes: state the requirements, name the bottleneck or the risk, propose a concrete mechanism with a name (not just a service name), and say out loud what you're trading off to get it.

## Interview questions

1. **"Design a CI/CD pipeline for a small team shipping a web app several times a day." Walk through your answer.**
   Answer: Clarify first — monolith or microservices, current pain points, downtime tolerance. Then: PR triggers CI (lint, tests, build a container image tagged with the commit SHA, not `latest`); on merge to `main`, CD pushes that exact image to a registry and deploys to staging automatically; a lightweight gate (automated smoke test, or a manual approval click) promotes the same image to production using a rolling update so there's no full-outage window; add basic alerting on error rate/latency post-deploy and a clear rollback path (redeploy the previous image tag) if the new version misbehaves.

2. **"How would you scale a REST API that's starting to time out under load?" What's your first move, before proposing any solution?**
   Answer: Find out where the actual bottleneck is before proposing a fix — is CPU/memory maxed on the app servers, is the database showing high query latency or connection saturation, or is it a downstream dependency? "Add more servers" only helps if the app tier itself is the bottleneck; if it's the database, horizontally scaling the stateless app tier just shifts more concurrent load onto an already-struggling database and can make things worse.

3. **Compare a rolling deploy, blue/green, and canary release — when would you pick each?**
   Answer: Rolling deploy gradually replaces old instances with new ones — good default, low infra overhead, but a bug in the new version affects some fraction of users for the duration of the rollout. Blue/green stands up a full parallel environment and switches traffic over at once — enables instant rollback (just switch back) at the cost of running double the infrastructure briefly. Canary sends a small percentage of traffic to the new version first and only ramps up if metrics look healthy — best when you specifically want to catch a bad release with minimal user impact before it's fully live, at the cost of a slower full rollout and needing good enough metrics to actually judge "healthy" quickly.

4. **"The database is now the bottleneck, not the app servers." What are your options, in the order you'd reach for them?**
   Answer: First, caching hot reads (Redis/Memcached) in front of the database to cut read load without touching the database itself — usually the fastest, lowest-risk win. Next, read replicas if the load is read-heavy, routing read queries away from the primary. If writes themselves are the bottleneck (replicas don't help writes), that's when partitioning/sharding the data becomes necessary — a much bigger architectural change, so it's the last resort, not the first move.

5. **You proposed a canary release for a deploy. The interviewer asks "how do you actually decide when to ramp the canary from 5% to 100%?" What's a real answer?**
   Answer: Define concrete, pre-agreed health signals before the rollout starts (error rate, p99 latency, any business-specific metric that would indicate a problem) compared against the baseline from the stable version, over a defined observation window — if the canary's metrics stay within acceptable bounds for that window, ramp up; if they degrade, roll back automatically or pause for investigation. "It looked fine so we bumped it" isn't a real answer — the decision needs to be tied to specific, monitored numbers, which ties directly back to having good [observability](../12-observability/) in place before you ever attempt a canary.

6. **An interviewer says "assume this is a side project with light traffic, not a big company." How should that change your deploy-pipeline or scaling answer?**
   Answer: Scale the design down to match — a rolling deploy on a single ECS service (or even one or two EC2 instances behind a load balancer) is entirely appropriate; blue/green, canary releases, database sharding, or multi-region failover would be over-engineering for the stated scale and should be explicitly named as *not* needed yet, with a one-line note on what you'd add first if traffic grew (e.g., "the first thing I'd add is a read replica once read load actually becomes a problem, not before"). Showing you can right-size a design to the actual constraints is itself the thing being tested.

## Watch

- [System Design Interview: A Step-By-Step Guide](https://www.youtube.com/watch?v=i7twT3x5yv8) — ByteByteGo. General framework for clarifying requirements before architecture — the same discipline applies directly to pipeline/scaling questions.
- [CICD Pipeline | System Design](https://www.youtube.com/watch?v=su3-fAEePs0) — ByteMonk. Walks through designing a CI/CD system end to end, closely mirroring this topic's first pattern.

## Further reading

- [Martin Fowler: BlueGreenDeployment](https://martinfowler.com/bliki/BlueGreenDeployment.html) — the original write-up on blue/green deploys and their tradeoffs.
- [Google SRE Book: Release Engineering](https://sre.google/sre-book/release-engineering/) — how a large org actually thinks about rollout strategy and risk.
