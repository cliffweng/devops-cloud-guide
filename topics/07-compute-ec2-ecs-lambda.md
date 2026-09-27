---
title: "07. Compute: EC2 / ECS / Lambda"
layout: default
nav_order: 8
---

# Compute: EC2 / ECS / Lambda — when to use which
{: .no_toc }

*~8 min read*

**🎯 Interview frequent**

## Why it matters

"How would you run this" is one of the most common questions in platform and SRE interviews, and a strong answer isn't "EC2" or "Lambda" by default — it's naming the tradeoff between control, ops burden, and cost for the specific workload described. This topic gives you the decision framework, not just the three service names.

## Core concepts

- **EC2 is a virtual machine you fully control.** You choose the OS, install anything, manage patching, and it runs continuously (and bills continuously) whether or not it's doing work. Maximum flexibility, maximum ops responsibility — you're the one who scales it, patches it, and notices when it's down.
- **ECS (Elastic Container Service) runs your containers for you, on infrastructure you can choose to not manage.** You give it a container image and a task definition (CPU/memory, environment, ports); ECS handles placement, restarts on crash, and scaling. With the **Fargate** launch type, there's no EC2 instance to manage at all — you pay per task's CPU/memory while it runs. With the **EC2 launch type**, ECS schedules your containers onto EC2 instances you still own and patch — more control, more ops work, often cheaper at sustained high utilization.
- **Lambda runs a single function in response to an event, with zero servers to think about.** You upload code (or a container image), define a trigger (HTTP request via API Gateway, an S3 upload, a queue message, a schedule), and AWS runs it, scales it to zero when idle, and scales it out automatically under load. You pay per invocation and per execution time — nothing, if it's never called.
- **Cold starts are Lambda's real tradeoff.** A function that hasn't run recently needs to initialize before handling its first request, adding latency (worse for larger runtimes/packages, mitigated by "provisioned concurrency" at extra cost). This makes Lambda a poor fit for latency-sensitive, constantly-hot traffic, and a great fit for spiky or infrequent workloads.
- **Execution time and statefulness are hard constraints, not preferences.** Lambda has a maximum execution duration (15 minutes) and no local persistent disk between invocations — long-running jobs, WebSocket servers, or anything needing in-memory state across requests don't fit the model and belong on EC2/ECS instead.
- **Cost shape flips depending on traffic pattern.** A steady, predictable, always-on workload is usually cheaper on EC2/ECS (paying for reserved/continuous capacity) than Lambda (paying per-invocation adds up fast at high sustained volume). A spiky, low-traffic, or "runs occasionally" workload is usually cheaper on Lambda, since EC2/ECS would otherwise sit idle (and billing) most of the time.
- **The real decision axis is control vs. operational burden, not "which is more modern."** EC2 gives you a full OS to shape however you want, at the cost of you patching and scaling it. ECS gives you container-level control with AWS handling placement/restarts. Lambda gives up control over the runtime environment entirely in exchange for AWS handling everything below "here's my function."
- **A single system often mixes all three.** A typical side project: the main API on ECS Fargate (steady traffic, needs to stay warm), a nightly report job on Lambda (infrequent, event-driven, fine with cold start), and maybe one EC2 box for something that needs raw OS access (a custom daemon, GPU work). Picking one compute type for an entire architecture is rarely the right instinct.

## Mental model

```
                       how much control do you need over the runtime?
                       full OS  <---------------------------->  none
                        EC2            ECS (Fargate)          Lambda

  always-on, steady traffic:      EC2 / ECS  (pay for continuous capacity)
  containerized, want less ops:   ECS Fargate
  event-driven, spiky, or rare:   Lambda      (pay per invocation, scales to zero)
  need >15 min runtime or local
  persistent state across calls:  EC2 / ECS   (Lambda structurally can't do this)
```

Think of EC2 as renting an apartment (you furnish it, you fix the plumbing, you pay rent whether you're home or not), ECS as renting a furnished apartment with a building super (less work, still paying by the month), and Lambda as a hotel room you're billed for only while you're checked in.

## Interview questions

1. **A team's workload is a REST API with steady, predictable traffic 24/7. Would you recommend Lambda? Why or why not?**
   Answer: Probably not as the primary choice — Lambda bills per invocation and per execution time, so a constantly-hot, high-volume API accumulates cost quickly compared to paying for continuous EC2/ECS capacity, and steady traffic also means you're not benefiting from Lambda's main advantage (scaling to zero when idle). ECS (especially Fargate, if they don't want to manage EC2 instances) is a stronger fit — containerized, autoscaled, but billed for reserved/running capacity rather than per-invocation.

2. **What is a cold start, why does it happen, and how would you mitigate it if you had to use Lambda for a latency-sensitive endpoint?**
   Answer: A cold start is the initialization overhead (spinning up an execution environment, loading the runtime and your code) Lambda incurs when it needs to run a function that isn't already "warm" from a recent invocation — this adds noticeable latency to that first request. Mitigations: provisioned concurrency (keeping a set number of instances pre-warmed at extra cost), reducing package/dependency size to speed initialization, or reconsidering whether Lambda is the right fit if the endpoint truly needs consistently low, predictable latency.

3. **What's the practical difference between ECS on Fargate and ECS on EC2, given that both are "ECS"?**
   Answer: Both use ECS to schedule and manage your containers, but Fargate has no underlying EC2 instances for you to see or manage — AWS runs each task on infrastructure it owns, and you pay per task's allocated CPU/memory for the duration it runs. The EC2 launch type schedules those same containers onto EC2 instances that are still yours to provision, patch, and pay for continuously — more control and often cheaper at sustained high utilization, at the cost of you owning instance-level operations again.

4. **A batch job needs to process a file for up to 45 minutes. Why is Lambda not a fit here, and what would you use instead?**
   Answer: Lambda has a hard maximum execution duration of 15 minutes per invocation, so a 45-minute job structurally cannot run as a single Lambda invocation — you'd have to awkwardly split it into chunks with state handoff between invocations, which adds complexity for no real benefit. An ECS task (Fargate or EC2) or a plain EC2 instance can run for as long as needed with no such ceiling, making it the natural fit for longer batch/background work.

5. **How would you decide between EC2 and ECS Fargate for a containerized app with no unusual OS-level requirements?**
   Answer: If there's no specific need to manage the underlying instance (custom kernel modules, specific OS-level tooling, GPU drivers not exposed by Fargate), Fargate is usually the better default — it removes instance patching, capacity planning, and placement decisions entirely, letting you focus on the container itself. EC2 (with ECS's EC2 launch type, or raw EC2) earns its extra ops burden back mainly through cost efficiency at high, sustained utilization, or when you genuinely need OS-level control Fargate doesn't expose.

6. **Design a system that ingests infrequent webhook events (a few hundred a day, unpredictable timing) and also serves a dashboard with constant low-traffic usage. Would you use the same compute type for both? Justify it.**
   Answer: No — the webhook ingestion is a strong Lambda fit (event-driven, spiky, infrequent, no benefit from staying warm, and you pay nothing between events), while the dashboard is better served by a small always-on ECS Fargate service (or even a single small EC2 instance), since a UI benefits from consistent low latency without paying the cold-start tax on every visitor. Using Lambda for the dashboard would risk visible cold-start lag on low-traffic pages; using an always-on service for the rare webhook would mean paying for idle capacity most of the day for no reason.

## Watch

- [AWS EC2 vs ECS vs Lambda | Which is right for YOU?](https://www.youtube.com/watch?v=-L6g9J9_zB8) — Be A Better Dev. Direct comparison covering control, cost, and scaling tradeoffs across all three.

## Further reading

- [AWS docs: Amazon ECS launch types](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/launch_types.html) — Fargate vs. EC2 launch type comparison.
- [AWS docs: Lambda execution environment](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html) — how cold starts and execution environments actually work.
- **Azure/GCP callout:** Azure's rough equivalents are VMs, Azure Container Apps/AKS, and Azure Functions; GCP's are Compute Engine, Cloud Run/GKE, and Cloud Functions — the control-vs-ops-burden spectrum maps almost 1:1 across all three clouds.
