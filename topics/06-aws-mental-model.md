---
title: "06. AWS mental model"
layout: default
nav_order: 7
---

# AWS mental model (regions, IAM, VPC lite)
{: .no_toc }

*~9 min read*

**🎯 Interview frequent**

## Why it matters

AWS has hundreds of services, and it's easy to drown before you've deployed anything. But almost every AWS interview question and almost every real deploy touches the same three concepts underneath all of it: where your stuff physically/logically lives (regions/AZs), who's allowed to touch it (IAM), and what network it's on (VPC). Get this mental model solid first — everything in [Compute](../07-compute-ec2-ecs-lambda/), [Storage & databases](../08-storage-databases-aws/), and [Networking](../09-networking-https/) hangs off it.

## Core concepts

- **Regions are independent, geographically separate deployments of AWS** (e.g., `us-east-1` in Virginia, `eu-west-1` in Ireland). Almost nothing is shared between regions by default — resources, data, and most services are region-scoped, so picking a region is picking where your latency-to-users and, often, your legal data-residency story comes from.
- **Availability Zones (AZs) are isolated data centers within a region**, connected by low-latency links but physically separate (separate power, cooling, networking). Spreading resources across multiple AZs is *the* mechanism for surviving a single data center failure — "multi-AZ" in a service name or setting means exactly this.
- **The account is the billing and hard-isolation boundary.** Everything you create lives inside an AWS account; resources in one account can't touch resources in another unless you explicitly grant cross-account access. Real orgs run multiple accounts (dev/staging/prod) precisely so a mistake in one can't reach another.
- **IAM answers "who can do what."** An IAM **user** or **role** is an identity; a **policy** (JSON) attached to it lists allowed (or denied) actions on specific resources. The default is deny-everything — you grant narrow permissions explicitly, not the other way around.
- **Roles, not long-lived keys, are how services (and you, ideally) should authenticate.** An EC2 instance or Lambda function gets an IAM **role** attached to it, which hands it short-lived, auto-rotated credentials — this is why you almost never see (and should never write) a hardcoded AWS access key inside application code running on AWS.
- **Least privilege is the governing principle, not a suggestion.** Grant exactly the actions and resources a task needs (`s3:GetObject` on one specific bucket, not `s3:*` on everything) — overly broad policies are the single most common root cause of "someone's compromised laptop turned into an entire AWS account takeover."
- **A VPC is your own private, logically isolated network inside a region.** It's carved into **subnets**, each pinned to one AZ. A **public subnet** has a route to an internet gateway (resources can be reached from/reach the internet); a **private subnet** doesn't — the classic pattern is a load balancer and NAT in public subnets, with app servers and databases in private subnets, reachable only from inside the VPC.
- **Security groups are a stateful firewall attached to resources** (not subnets) — they control what traffic can reach an EC2 instance, RDS database, etc., by port/protocol/source. "Why can't my app reach the database" is very often "the database's security group doesn't allow inbound traffic from the app's security group."

## Mental model

```
AWS account
 └── Region (us-east-1)
      ├── AZ: us-east-1a          AZ: us-east-1b
      │   └── VPC (10.0.0.0/16, spans both AZs)
      │        ├── public subnet (10.0.1.0/24)  --- internet gateway
      │        │     └── load balancer, NAT gateway
      │        └── private subnet (10.0.2.0/24) --- no direct internet route
      │              └── EC2 / ECS tasks, RDS database
      └── IAM: users/roles + policies -> govern who/what can call which
                                          AWS APIs, everywhere in the account
```

Think of IAM as the lock on every single door in the account (identity-based), and the VPC as the building's floor plan (network-based) — a request has to pass both checks: is this identity allowed to do this action, *and* can this network traffic even reach the resource in the first place.

## Interview questions

1. **What's the difference between a region and an Availability Zone, and why does that distinction matter for designing a highly available system?**
   Answer: A region is a broad geographic area with multiple, physically isolated data centers (AZs) inside it, connected by fast links but independent enough that one AZ's outage (power, cooling, networking) doesn't take down another. Designing for high availability means spreading resources — EC2 instances, database replicas — across at least two AZs within a region, so a single data-center-level failure doesn't take your whole system down; relying on one AZ is a single point of failure.

2. **Why should an EC2 instance use an IAM role instead of having AWS access keys hardcoded or stored in a config file on the instance?**
   Answer: An IAM role attached to the instance provides temporary, automatically-rotated credentials fetched at runtime — there's no long-lived secret sitting on disk or in source control for someone to steal. Hardcoded keys are static, don't rotate, and if the instance or a repo is compromised, the leaked key keeps working until someone manually revokes it — a much bigger and longer-lived blast radius.

3. **Explain "least privilege" in IAM with a concrete example of a policy that violates it, and how you'd fix it.**
   Answer: Least privilege means granting only the specific actions and resources a task actually needs — a violation would be attaching a policy with `"Action": "s3:*", "Resource": "*"` to a Lambda function that only needs to read one bucket. The fix is scoping it down to `"Action": ["s3:GetObject"], "Resource": "arn:aws:s3:::my-specific-bucket/*"` — so a compromised or buggy function can't accidentally (or maliciously) touch every S3 bucket in the account.

4. **What's the practical difference between a public and a private subnet, and why would you put a database in a private one?**
   Answer: A public subnet has a route table entry pointing to an internet gateway, so resources in it can have a public IP and be reached from (or reach) the internet directly; a private subnet has no such route, so nothing in it is directly internet-reachable. Databases go in private subnets so they're only reachable from inside the VPC (e.g., from app servers) — eliminating an entire class of "someone found my database's IP and is now port-scanning it from the open internet" risk.

5. **Your app server can't connect to your RDS database even though both are running and in the same VPC. What's the most likely first thing to check?**
   Answer: The RDS instance's security group — specifically, whether it has an inbound rule allowing traffic on the database port (e.g., 5432 for Postgres) from the app server's security group (or its subnet's CIDR range). Security groups are the most common cause of "both resources are up but can't talk to each other" inside a VPC.

6. **Why do real organizations use multiple AWS accounts (e.g., separate dev/staging/prod) instead of one account with IAM permissions separating environments?**
   Answer: An AWS account is a hard isolation boundary — resources, quotas, and billing are separate, and a mistake or compromise in one account (an over-permissive policy, a leaked credential, a runaway resource) can't reach into another account at all. Relying on IAM alone within a single account means a permissions misconfiguration or a sufficiently broad compromised role could touch prod resources directly; separate accounts make that structurally impossible without an explicit cross-account trust relationship.

## Watch

- [AWS IAM Overview in 7 minutes | Beginner Overview](https://www.youtube.com/watch?v=y8cbKJAo3B4) — Be A Better Dev. Tight overview of users, roles, and policies.
- [Amazon/AWS VPC (Virtual Private Cloud) Basics](https://www.youtube.com/watch?v=7_NNlnH7sAg) — Tiny Technical Tutorials. Walks through subnets, route tables, and internet gateways hands-on in the console.

## Further reading

- [AWS docs: What is IAM?](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) — official IAM concepts (users, roles, policies).
- [AWS docs: What is Amazon VPC?](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) — official VPC/subnet/route table reference.
- **Azure/GCP callout:** the same shape exists everywhere — Azure has Resource Groups + Azure AD (Entra ID) roles + Virtual Networks; GCP has Projects + IAM + VPC Networks. The names differ, the account/identity/network split doesn't.
