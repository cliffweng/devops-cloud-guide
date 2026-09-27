---
title: "11. Kubernetes overview"
layout: default
nav_order: 12
---

# Kubernetes overview (pods, deploys, services)
{: .no_toc }

*~9 min read*

**🎯 Interview frequent**

## Why it matters

You're not going to be asked CKA-exam-depth Kubernetes questions in an internship interview, but not knowing the difference between a pod and a deployment — or why a service exists at all — reads as a real gap in a platform/SRE loop. This topic gives you the vocabulary and mental model to talk credibly about Kubernetes, and to know when you'd actually reach for it over ECS ([Compute: EC2 / ECS / Lambda](../07-compute-ec2-ecs-lambda/)).

## Core concepts

- **Kubernetes (k8s) is a container orchestrator**: it takes a fleet of machines (nodes) and schedules containers onto them, restarts them when they crash, and reconciles the running state toward whatever you've declared you want — the same "declare desired state" idea as [Terraform](../10-iac-terraform-basics/), but continuously enforced at runtime instead of applied once.
- **A pod is the smallest deployable unit — one or more tightly-coupled containers that share network and storage.** Almost always it's one container per pod; multiple containers in a pod (sidecars) are for things that genuinely need to share a network namespace/localhost with the main container (a logging agent, a proxy). Pods are disposable — Kubernetes kills and recreates them routinely, so nothing important should live only inside a pod's own filesystem.
- **A Deployment manages a set of identical pod replicas and handles rolling updates.** You declare "I want 3 replicas of this pod spec," and the Deployment controller continuously works to make that true — if a pod dies, it's replaced automatically. When you update the pod spec (a new image version), the Deployment does a rolling update: gradually replacing old pods with new ones rather than killing everything at once, so there's no full-outage window during a deploy.
- **A Service gives a stable network identity to a set of pods that are constantly being created and destroyed.** Pods get new IP addresses every time they're recreated, so nothing should ever hardcode a pod's IP — a Service provides a stable DNS name and IP that automatically load-balances across whichever pods currently match its label selector, regardless of how many times those pods have been replaced underneath it.
- **Labels and selectors are how Kubernetes objects find each other — not names.** A Deployment's pods get labels (e.g., `app: api`); a Service selects pods by matching those labels, not by any direct reference. This loose coupling is why you can replace every pod behind a Service without ever touching the Service itself.
- **A Namespace is a way to partition one cluster into logical groups** (e.g., `dev`, `staging`, or per-team) — it's not a security boundary by itself (that needs RBAC and network policies on top), but it does scope names and is the natural place to apply resource quotas.
- **Readiness and liveness probes are Kubernetes' version of the health checks from [Networking & HTTPS](../09-networking-https/).** A liveness probe failing gets the container restarted; a readiness probe failing removes the pod from a Service's load-balancing rotation without restarting it — conflating the two is a common real misconfiguration (e.g., using only a liveness probe means a temporarily-overloaded-but-alive pod keeps receiving traffic it can't handle instead of being pulled out).
- **`kubectl` is how you talk to the cluster's API server** — `kubectl apply -f config.yaml` submits desired state (the same plan/apply rhythm as Terraform, but continuously reconciled by controllers rather than a one-time apply), and `kubectl get pods`/`kubectl logs`/`kubectl describe` are the everyday debugging trio.
- **When Kubernetes is overkill vs. worth it:** a single side project with one or two services rarely needs k8s's operational complexity — ECS or even a couple of EC2 instances get you there faster with far less to learn and maintain. Kubernetes earns its complexity at real scale (many services, multiple teams, need for portability across clouds, or an org that's already standardized on it) — knowing this tradeoff explicitly is itself a good interview answer.

## Mental model

```
  Deployment (desired: 3 replicas of pod spec X)
        |
        v
     ReplicaSet  --- ensures exactly 3 pods matching spec X exist right now
        |
   +----+----+----+
   v         v         v
  pod       pod       pod        <- disposable, each gets its own IP,
 (app:api) (app:api) (app:api)      recreated on crash or rolling update

        ^ selected by label app:api ^
                    |
                 Service (stable DNS name + IP)
                    |
              load-balances traffic across whichever
              pods currently match the label, no matter
              how many times they've been replaced
```

Think of a Deployment as a standing order ("always have 3 of these running") that a ReplicaSet continuously fulfills, and a Service as a receptionist who always knows how to route you to *someone* on the current team, even though the individual employees change constantly.

## Interview questions

1. **What's the difference between a pod and a Deployment, and why wouldn't you create pods directly in a real setup?**
   Answer: A pod is a single instance of one or more containers — it's disposable and has no built-in mechanism to recreate itself if it dies. A Deployment declares a desired number of replicas of a pod spec and continuously ensures that many are running, automatically replacing any that crash and handling rolling updates when the spec changes — creating bare pods directly means losing all of that self-healing and update orchestration.

2. **Why can't you just hardcode a pod's IP address in another service that needs to talk to it?**
   Answer: Pods are ephemeral — every time one is recreated (crash, rolling update, node failure), it gets a brand-new IP address, so any hardcoded IP goes stale almost immediately. A Kubernetes Service provides a stable DNS name/IP that's decoupled from any individual pod's lifecycle, automatically routing to whichever pods currently match its label selector.

3. **How does a rolling update actually avoid downtime when deploying a new version of an app?**
   Answer: The Deployment controller replaces old pods with new ones gradually rather than all at once — it brings up new pods, waits for them to pass their readiness probe, then terminates a corresponding number of old pods, repeating until all pods are on the new version. At every point during the rollout, there's still a mix of ready pods (old and/or new) behind the Service, so traffic never has zero healthy backends to route to.

4. **What's the practical difference between a liveness probe and a readiness probe, and what goes wrong if you only configure one?**
   Answer: A failing liveness probe causes Kubernetes to restart the container (it assumes the process is stuck/broken); a failing readiness probe removes the pod from Service load-balancing without restarting it (it assumes the process is alive but temporarily unable to serve traffic — e.g., still warming up, or briefly overloaded). Configuring only a liveness probe means an alive-but-overwhelmed pod keeps receiving traffic it can't handle (since nothing pulls it out of rotation), potentially worsening the overload instead of letting it recover.

5. **A junior teammate asks why their app needs a Namespace if it's the only thing running in the cluster. How would you explain what a Namespace actually does and doesn't provide?**
   Answer: A Namespace logically partitions objects within one cluster (scoping names, letting you apply resource quotas, and organizing dev/staging/prod or per-team resources) — but it is not, by itself, a security boundary; without RBAC rules and network policies layered on top, workloads in different namespaces can generally still reach each other and identities aren't automatically restricted to their own namespace. For a single-app cluster, a Namespace mainly buys organizational clarity, not isolation.

6. **A team running a single API and a single worker service on one cloud is debating Kubernetes vs. ECS. What would you actually ask them before recommending one?**
   Answer: Whether they expect to grow into many more services/teams needing shared infra conventions, whether multi-cloud portability is a real near-term need (not hypothetical), and whether anyone on the team already has Kubernetes operational experience — for two services on one cloud with no multi-cloud requirement, ECS gets them running with dramatically less operational surface area to learn and maintain, and Kubernetes' extra complexity (etcd, controllers, RBAC, networking plugins) wouldn't be paying for itself yet.

## Watch

- [Kubernetes Explained in 6 Minutes | k8s Architecture](https://www.youtube.com/watch?v=TlHvYWVUZyc) — ByteByteGo. Fast architectural overview of the control plane, nodes, and core objects.
- [Kubernetes Simply Explained](https://www.youtube.com/watch?v=Xv9dnKHO8tg) — TechWorld with Nana. Clear conceptual walkthrough of pods, deployments, and services.

## Further reading

- [Kubernetes docs: Pods](https://kubernetes.io/docs/concepts/workloads/pods/) — official concept reference.
- [Kubernetes docs: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) — official rolling update mechanics.
- [Kubernetes docs: Service](https://kubernetes.io/docs/concepts/services-networking/service/) — official reference on stable networking for pod sets.
