---
title: "05. Images, registries, compose"
layout: default
nav_order: 6
---

# Images, registries, compose
{: .no_toc }

*~7 min read*

**Interview occasional**

## Why it matters

Building a container on your laptop is half the story — you also need to get that image somewhere your deploy target can pull it from, and most real apps are more than one container (an API, a database, maybe a cache). This topic is the glue between "I have a Dockerfile" and "this runs as a multi-container app pulled from somewhere other than my machine."

## Core concepts

- **A registry is just a server for storing and distributing images**, the same relationship a package registry (npm, PyPI) has to your language's packages. Docker Hub is the public default; most companies also run a private registry (Amazon ECR, GitHub Container Registry, GitLab Registry) for images they don't want public.
- **Image tags are labels, not guarantees.** `myapp:latest` is just a mutable pointer someone can re-push at any time — it's convenient for local dev but a liability in production, because "latest" today might not be "latest" tomorrow, silently changing what gets deployed. Production pipelines tag images with something immutable and traceable: a git SHA, a semantic version, or a build number.
- **`docker push`/`docker pull` move images to/from a registry**, and require authentication for private registries (`docker login`) — this is the same secrets-handling concern from [CI with GitHub Actions](../03-ci-github-actions/): registry credentials go in CI secrets, never hardcoded in a Dockerfile or script.
- **Multi-stage builds keep production images small.** A `Dockerfile` can have multiple `FROM` stages — one with the full build toolchain (compilers, dev dependencies) that produces an artifact, and a final minimal stage that only copies that artifact in. This avoids shipping your entire build toolchain into the production image just to run the compiled result.
- **Docker Compose defines a multi-container app as one file.** A `docker-compose.yml` describes each service (container), how they're networked together, what ports are published, and what environment variables/volumes each needs — `docker compose up` starts the whole stack with one command instead of several long `docker run` invocations.
- **Compose is for local dev and small deployments, not production orchestration at scale.** It's excellent for "spin up my API + Postgres + Redis together on one machine" — but it doesn't handle multi-host scheduling, rolling deploys, or auto-healing, which is where [Kubernetes overview](../11-kubernetes-overview/) or a managed service like ECS ([Compute: EC2 / ECS / Lambda](../07-compute-ec2-ecs-lambda/)) takes over.
- **Compose services can depend on each other**, but `depends_on` only controls *start order*, not "wait until the dependency is actually ready" — a database container can report "started" before it's actually accepting connections, so apps still need their own retry/backoff logic on startup, not just a Compose ordering hint.

## Mental model

```
   your laptop                    registry (Docker Hub / ECR / GHCR)
   docker build -t myapp:sha123        ^
        |                              |
        +----- docker push ------------+
                                        |
                                        v
                                  docker pull myapp:sha123
                                        |
                                        v
                              deploy target (EC2 / ECS / k8s node)

   docker-compose.yml
   services:
     api:      -----depends_on----> db
     db:       (postgres image, volume for data)
     cache:    (redis image)
   -> `docker compose up` starts all three, networked together, one command
```

Think of a registry as GitHub for built artifacts instead of source code — you push a specific, tagged snapshot, and every environment that pulls that tag gets byte-for-byte the same thing.

## Interview questions

1. **Why is tagging an image `latest` risky for a production deploy?**
   Answer: `latest` is a mutable tag — anyone can push a new image and have it become the new `latest` at any time, so two deploys referencing `latest` a day apart could pull completely different images without any code change on your end. Production deploys should reference an immutable tag (a git SHA or semantic version) so what gets deployed is exactly reproducible and traceable back to the commit that built it.

2. **What problem do multi-stage Dockerfile builds solve?**
   Answer: They let you use a full build environment (compilers, dev dependencies, build tools) in an early stage to produce a compiled artifact, then copy only that artifact into a clean, minimal final stage — so the shipped production image doesn't carry the entire build toolchain's size and attack surface, just the runtime and the artifact it needs.

3. **How is Docker Compose different from Kubernetes, and when would Compose actually be the right choice?**
   Answer: Compose defines and runs a multi-container app on a single host with one config file and one command — it has no concept of scheduling across multiple machines, rolling updates, or self-healing if a container crashes on a node that's gone. It's the right choice for local development, small single-server deployments, or CI test environments where you just need several services networked together quickly; once you need multi-host scale, auto-healing, or zero-downtime rolling deploys, that's Kubernetes' (or ECS's) job.

4. **Your Compose file has `depends_on: [db]` on your API service, but the API still fails on startup with a database connection error. Why didn't `depends_on` prevent that?**
   Answer: `depends_on` only guarantees Docker starts the `db` container before the `api` container — it does not wait for Postgres to finish its own startup and actually be ready to accept connections, which can take a few seconds after the container itself reports "running." The API needs its own retry-with-backoff logic on startup (or a Compose healthcheck + `condition: service_healthy`) rather than assuming start order equals readiness.

5. **You need your CI pipeline to push a built image to a private registry. What has to be set up for that to work securely?**
   Answer: Registry credentials (a token or username/password, scoped as narrowly as possible — e.g., push-only if that's all CI needs) stored as CI secrets, never hardcoded in the Dockerfile, compose file, or workflow YAML; the pipeline calls `docker login` using those secrets right before `docker push`, and the image is tagged with something traceable (like the commit SHA) rather than `latest`.

## Watch

- [What is Docker Compose? | Docker Concepts](https://www.youtube.com/watch?v=f3xb7kt_dH4) — Docker (official). Short official explainer on what Compose actually manages.
- [Docker Compose in 12 Minutes](https://www.youtube.com/watch?v=Qw9zlE3t8Ko) — Jake Wright. Hands-on walkthrough writing a multi-service `docker-compose.yml`.

## Further reading

- [Docker docs: Multi-stage builds](https://docs.docker.com/build/building/multi-stage/) — official guide to multi-stage Dockerfiles.
- [Amazon ECR docs: What is Amazon ECR?](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html) — AWS's private container registry, if you're pushing images for ECS/EKS deploys.
