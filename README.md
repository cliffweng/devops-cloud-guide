# DevOps & Cloud Guide

A practical DevOps and cloud guide for builders — for shipping side projects that actually run in production, and for interviewing for SWE, SRE, and platform roles.

**Live site:** https://cliffweng.github.io/devops-cloud-guide/

## Roadmap

14 topics, one file each under [`topics/`](topics/), ordered day-1-box → deploying → cloud fundamentals → interview:

1. Linux/SSH & the day-1 box
2. Git workflows for teams
3. CI with GitHub Actions
4. Containers & Docker
5. Images, registries, compose
6. AWS mental model (regions, IAM, VPC lite)
7. Compute: EC2 / ECS / Lambda — when to use which
8. Storage & databases on AWS (S3, RDS overview)
9. Networking & HTTPS (DNS, TLS, ALB)
10. IaC with Terraform basics
11. Kubernetes overview (pods, deploys, services — not CKA depth)
12. Observability (logs, metrics, traces, alerts)
13. On-call & incident response basics
14. Interview patterns (design a deploy pipeline / scale a service)

## Interview hotspots

Every topic page carries a badge (🎯 Interview frequent / Interview occasional / Background) so you know where to spend prep time. If you're short on time, prioritize these:

- **CI with GitHub Actions** — the most common "walk me through your deploy process" starting point; interviewers use it to check you understand pipelines, not just that you clicked merge.
- **Containers & Docker** — the prerequisite for almost every other topic in this guide; expect "what is a container, really" and image-layer questions.
- **AWS mental model** — IAM and the region/account/VPC shape come up as a warm-up in nearly every cloud-adjacent interview, even non-infra roles.
- **Compute: EC2 / ECS / Lambda** — the classic "how would you run this" question; interviewers are listening for a cost/ops/scale tradeoff, not a product pitch.
- **Storage & databases on AWS** — S3 and RDS basics anchor a huge fraction of "where does the data live" follow-ups in system design rounds.
- **Networking & HTTPS** — DNS, TLS, and load balancers are the connective tissue interviewers use to see if you can trace a request end-to-end.
- **IaC with Terraform basics** — "how do you avoid clicking around the AWS console" is a very common platform/SRE screening question.
- **Kubernetes overview** — you won't be asked CKA-depth questions, but not knowing a pod from a deployment is an instant red flag in platform interviews.
- **Observability** — "how do you know it's broken before your users tell you" is asked in nearly every SRE/platform loop.
- **Interview patterns** — the capstone: designing a deploy pipeline or scaling a service end-to-end, which is where everything else in this guide gets combined under interview pressure.

**Occasional** (still worth knowing, less likely to anchor a whole interview): Linux/SSH & the day-1 box, Git workflows for teams, images/registries/compose, on-call & incident response basics.

This split is a judgment call based on what shows up in practice today for SWE/SRE/platform internship interviews, not a guarantee for any specific loop — adjust your prep if a role is unusually infra-heavy.

## How to use this guide

Each topic page is designed to be read in **~7–9 minutes** and follows the same structure: why it matters, core concepts, a mental model, interview questions with brief answer keys, and a short watch list of verified YouTube videos. Read them in order if you're starting from zero, or jump straight to what you need. No backend, no auth, no sign-up — just read the pages.

This guide is **AWS-first** (it's the cloud most internships and side projects touch first), with short Azure/GCP callouts where the equivalent service is worth knowing by name.

## How to contribute

See [CONTRIBUTING.md](CONTRIBUTING.md). In short: one topic per file, keep it under ~10 minutes to read, and only link to sources (especially YouTube videos) you've personally verified exist. Open an issue before proposing new topics or restructuring the curriculum.

## Decisions

These are the product locks this guide was built against — echoed here so future contributors don't accidentally relitigate them:

- **Audience**: Builders shipping side projects, and interviewing for SWE/SRE/platform internships. Not an enterprise ops or certification course.
- **Time-boxed**: every topic is readable in ~7–9 minutes. Depth is sacrificed for scannability; "further reading" links are where depth lives.
- **Learning + interview prep in one page**: each topic pairs core concepts with interview questions, rather than splitting them into separate tracks.
- **AWS-first**: concrete examples use AWS; Azure/GCP get a short callout where the mapping is worth knowing, not a parallel deep dive.
- **Real links only**: every YouTube link is verified to exist (via the YouTube oEmbed endpoint) before being added. No invented URLs, ever.
- **Static site, GitHub Pages, Just the Docs**: no backend, no auth, no Vercel. Cheap to host, cheap to maintain, easy to contribute to via plain Markdown + front matter.
- **Non-goals**: deep multi-cloud certification tracks, heavy FinOps, compliance frameworks, quizzes/auth/progress backend.

## Enabling GitHub Pages

Just the Docs is configured via `_config.yml` (remote theme). After this lands on `main`:

1. Repo **Settings → Pages**.
2. **Build and deployment → Source**: **Deploy from a branch**.
3. Branch: `main` / folder: `/ (root)`.
4. Save. Site publishes to https://cliffweng.github.io/devops-cloud-guide/

## Local preview

```bash
bundle install
bundle exec jekyll serve
```

Then open `http://localhost:4000`.

## License

[MIT](LICENSE)
