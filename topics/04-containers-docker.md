---
title: "04. Containers & Docker"
layout: default
nav_order: 5
---

# Containers & Docker
{: .no_toc }

*~8 min read*

**🎯 Interview frequent**

## Why it matters

"It works on my machine" stops being an acceptable answer the moment you deploy anything, and containers are the industry's answer to that problem. Docker is the near-universal starting point — it's how you package an app with everything it needs so it runs the same way on your laptop, in CI, and on a cloud VM. Almost every later topic in this guide (compute choices, Kubernetes, CI pipelines) assumes you already get containers.

## Core concepts

- **A container is an isolated process, not a mini virtual machine.** It shares the host machine's kernel but gets its own filesystem, process namespace, and network interface via Linux kernel features (namespaces and cgroups) — this is why containers start in milliseconds and use a fraction of the resources of a VM, which has to boot an entire separate OS kernel.
- **An image is a read-only template; a container is a running instance of it.** You can run many containers from the same image simultaneously, each isolated from the others, the same way many processes can run from the same compiled binary.
- **Images are built in layers, and layers are cached.** Each instruction in a `Dockerfile` (`FROM`, `RUN`, `COPY`, ...) creates a new layer stacked on the previous one. Docker caches unchanged layers on rebuild — this is why `Dockerfile` instruction *order* matters for build speed: put things that change rarely (installing dependencies) before things that change often (copying your source code), so a code-only change doesn't invalidate the expensive dependency-install layer.
- **Containers are ephemeral by design.** Anything written inside a container's writable layer disappears when the container is removed. If data needs to survive (a database's files, uploaded content), you mount a **volume** — storage that lives outside the container's lifecycle and can be reattached to a new container.
- **Networking: containers get their own virtual network by default,** and you explicitly publish ports (`-p 8080:80`) to make a container's internal port reachable from the host. Two containers on the same Docker network can reach each other by container name without any port publishing at all — this is the mechanism [images, registries, compose](../05-images-registries-compose/) builds on for multi-container apps.
- **`docker run` vs. `docker exec`.** `run` starts a brand-new container from an image; `exec` runs an additional command inside a container that's already running (e.g., `docker exec -it mycontainer bash` to poke around). Confusing the two is a common early mistake — `run`-ing repeatedly when you meant to `exec` just spins up more containers.
- **Base image choice is a real tradeoff.** A full OS base image (`ubuntu`) is easy to debug but large and has a bigger attack surface; a minimal base (`alpine`, `distroless`) is smaller and more secure but can be missing tools you'd normally reach for when debugging — teams often use a full image for local dev and a minimal one for production.

## Mental model

```
   Dockerfile                         image (read-only layers)
   FROM node:20-alpine   -----build-->  [layer: base OS]
   COPY package.json .                 [layer: base OS + deps]  <- cached if
   RUN npm install                     [layer: + deps]             package.json
   COPY . .                            [layer: + your code]        unchanged
   CMD ["node","app.js"]

   image  --docker run-->  container 1  (isolated process, own filesystem view)
   image  --docker run-->  container 2  (independent instance, same image)

   container's writable layer: gone when container is removed
   docker volume: survives, mounted in from outside
```

Think of an image as a shipped, frozen recipe and a container as one specific pot cooking from it right now — you can start ten pots from the same recipe, and throwing a pot away never changes the recipe.

## Interview questions

1. **What's the actual difference between a container and a virtual machine?**
   Answer: A VM virtualizes hardware and runs its own full guest OS kernel on top of a hypervisor, so it's heavyweight but fully isolated even at the kernel level. A container shares the host's kernel and uses namespaces (isolated views of processes, network, filesystem) and cgroups (resource limits) to isolate processes at the OS level — much lighter and faster to start, but with a smaller isolation boundary since the kernel is shared.

2. **Why does `Dockerfile` instruction order affect build speed, and how would you reorder a Dockerfile that does `COPY . .` before `RUN npm install`?**
   Answer: Docker caches each layer and only rebuilds a layer (and everything after it) if its inputs changed; if you `COPY . .` (your whole source tree) before installing dependencies, *any* code change invalidates the cache for the dependency-install layer too, forcing a full reinstall on every build. The fix: `COPY package.json package-lock.json .` and `RUN npm install` first, then `COPY . .` last — so dependency installation is only re-run when the dependency manifest itself changes.

3. **A container writes a file while it's running, then the container is removed and a new one is started from the same image. Is that file still there? Why or why not, and how would you fix it if you needed it to persist?**
   Answer: No — writes go into the container's own writable layer, which is deleted along with the container; the underlying image is read-only and untouched. To persist data across container restarts/replacements, mount a Docker volume (or bind mount) at that path, so the data lives outside the container's lifecycle and gets reattached to whatever container mounts it next.

4. **Two containers need to talk to each other. Do you need to publish ports with `-p` for that, and why or why not?**
   Answer: No — port publishing (`-p host:container`) is only needed to expose a container's port to the *host machine* (and beyond). Containers attached to the same Docker network can already reach each other directly over that internal network, addressing each other by container name, without any port being published to the host at all.

5. **When would you choose a minimal base image like `alpine` over a full one like `ubuntu`, and what's the tradeoff?**
   Answer: Minimal images are smaller (faster pulls/deploys) and have a reduced attack surface (fewer installed packages means fewer potential vulnerabilities) — generally the right default for production. The tradeoff is debuggability: minimal images often lack common shell tools or a package manager you'd reach for when troubleshooting inside a running container, so some teams keep a fuller image for local development and switch to minimal only for the deployed artifact.

6. **You run `docker run myapp` three times in a row expecting to "get back into" your app. What actually happened, and what should you have done instead?**
   Answer: Each `docker run` started a brand-new, independent container from the image — you now have three separate containers running, not one container you re-entered. To get back into an already-running container, use `docker exec -it <container> bash` (or `docker attach` if you want the original foreground process), and use `docker ps` first to see what's actually running.

## Watch

- [The Only Docker Tutorial You Need To Get Started](https://www.youtube.com/watch?v=DQdB7wFEygo) — The Coding Sloth. Covers images, containers, volumes, and networking basics hands-on.
- [Docker in 100 Seconds](https://www.youtube.com/watch?v=Gjnup-PuquQ) — Fireship. Quick-hit visual explanation of why containers exist and how they differ from VMs.

## Further reading

- [Docker docs: What is a container?](https://www.docker.com/resources/what-container/) — official conceptual overview.
- [Docker docs: Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) — layer caching, multi-stage builds, and image size tips.
