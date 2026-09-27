---
title: "08. Storage & databases on AWS"
layout: default
nav_order: 9
---

# Storage & databases on AWS (S3, RDS overview)
{: .no_toc }

*~8 min read*

**🎯 Interview frequent**

## Why it matters

"Where does the data live" is a question every real app and every system-design-flavored interview eventually asks. AWS gives you a spectrum from "throw files at an object store" to "fully managed relational database" to "run your own database on a VM" — knowing which one fits which job (and why) is a recurring signal interviewers look for.

## Core concepts

- **S3 is object storage, not a filesystem.** You store and retrieve whole objects (files) by key inside a bucket — there's no in-place partial edit like a real filesystem, no directory structure underneath (the "folders" you see in the console are just key prefixes with `/` in them for display). This makes S3 a great fit for static assets, backups, logs, user uploads, and data lake storage — and a bad fit for anything needing frequent small in-place writes or file-locking semantics.
- **S3 is durable by design and regional by default.** Objects are redundantly stored across multiple AZs within a region automatically — you don't configure that. Buckets are private by default; a public bucket is something you explicitly opt into, and "someone left an S3 bucket public" is one of the most common real-world cloud data leak stories, which is why bucket policies and "Block Public Access" settings get checked in almost every security review.
- **S3 storage classes trade retrieval speed/cost for storage cost.** Standard (frequent access), Infrequent Access (cheaper storage, cost to retrieve), Glacier (cheapest storage, slow/costly retrieval, built for archival). Lifecycle rules can automatically move objects to colder tiers as they age — a common real cost-optimization lever.
- **RDS is a managed relational database** (Postgres, MySQL, and others) — AWS handles patching, automated backups, and failover, so you don't SSH into a box to run `apt upgrade postgresql`. You still design your schema, write your queries, and choose your instance size; RDS manages the operational layer underneath, not your data model.
- **Multi-AZ RDS is for availability, read replicas are for read scaling — they solve different problems.** Multi-AZ keeps a synchronously-replicated standby in a different AZ that RDS automatically fails over to if the primary goes down (protects against an AZ outage, doesn't help with read load). Read replicas are asynchronously-replicated copies you can route read-only queries to, reducing load on the primary — they don't provide automatic failover for writes on their own.
- **Connection limits are a real constraint relational databases have that you'll hit in practice.** Every open connection costs the database memory; a burst of traffic opening a connection per request (instead of pooling) can exhaust the database's max connections long before it runs out of CPU. This is why connection pooling (in-app, or via a proxy like RDS Proxy) is a standard piece of any RDS-backed app at any real scale.
- **When to reach for S3 vs. RDS vs. "just run a database on EC2":** S3 for unstructured blobs/files at any scale; RDS when you need relational structure, transactions, and joins without wanting to own the ops; self-managed on EC2 only when you need a specific database engine/version/extension RDS doesn't support, or specific OS-level control — which is a deliberately narrow use case, not a default.
- **Presigned URLs let you grant temporary, scoped access to a private S3 object** (e.g., "let this user upload directly to this exact key for the next 10 minutes") without making the bucket public or routing the file through your own server — a common pattern for user file uploads/downloads.

## Mental model

```
  unstructured files/blobs             structured, relational, transactional
  (images, backups, logs, uploads)     (users, orders, anything needing joins/ACID)
           |                                          |
           v                                          v
          S3                                         RDS
   (bucket + key, durable            (managed Postgres/MySQL,
    across AZs, private              Multi-AZ for failover,
    by default)                       read replicas for read scale)

  presigned URL -> temporary, scoped access to one S3 object, no public bucket needed
```

Think of S3 as a warehouse of labeled boxes (grab box `key`, that's it) and RDS as a librarian who actually understands relationships between the books (joins, transactions, consistency) — you don't ask the warehouse to cross-reference anything, and you don't dump loose files on the librarian's desk.

## Interview questions

1. **Why is S3 described as "object storage" rather than a filesystem, and what's a workload that fits it poorly because of that?**
   Answer: S3 stores and retrieves whole immutable objects by key — there's no in-place partial write, no real directory structure, and no file-locking; every "write" replaces the whole object. A workload needing frequent small in-place edits to shared files (e.g., a database's own data files, or a traditional multi-writer shared filesystem) fits poorly, since you'd have to rewrite the entire object for any change.

2. **A bucket meant to hold private user documents was found publicly readable. What settings would you check first, and what's the general principle for avoiding this?**
   Answer: Check the bucket policy and access control list for any statement granting public read/list access, and check whether "Block Public Access" (an account/bucket-level setting that overrides permissive policies) is enabled. The general principle: buckets are private by default, so any public exposure was an explicit configuration choice somewhere — treat "public" as something that requires a deliberate, reviewed decision, never a default or an accident you don't notice.

3. **What's the difference between Multi-AZ RDS and a read replica, and why would having one but not the other leave a gap?**
   Answer: Multi-AZ keeps a synchronous standby in another AZ that RDS automatically fails over to on a primary failure — it protects availability, not read throughput. A read replica is an asynchronous copy you route read queries to, reducing load on the primary — it protects read scalability, not automatic write-failover. Having only a read replica means an AZ outage taking down the primary still causes a write outage (replicas aren't auto-promoted by default); having only Multi-AZ means you have failover protection but no relief if read traffic, not availability, is your bottleneck.

4. **Your app is hitting "too many connections" errors on RDS even though CPU and memory look fine. What's likely happening, and how would you fix it?**
   Answer: The app is likely opening a new database connection per request (or per worker without pooling) instead of reusing a bounded pool of connections, and the database's max connection limit — a fixed resource independent of CPU/memory headroom — is being exhausted. The fix is connection pooling, either in the application/ORM layer or via a proxy like RDS Proxy, so a bounded number of long-lived connections are shared across many requests instead of one-connection-per-request.

5. **When would you choose to run a database on a plain EC2 instance instead of using RDS?**
   Answer: Only when you need something RDS structurally doesn't offer — a specific database engine or version RDS doesn't support, a specific extension requiring OS-level installation, or genuine OS-level control over the database process. Outside of that narrow case, self-managing means taking back all the patching, backup, and failover work RDS does for you, for no benefit — it's a deliberate tradeoff, not a lighter-weight default.

6. **How would you let a user upload a large file directly to S3 from their browser, without routing the file's bytes through your own backend server?**
   Answer: Generate a presigned URL server-side (scoped to a specific bucket/key, with an expiration and often a specific HTTP method), and have the browser `PUT` the file directly to that URL. The backend never touches the file's bytes — it only issues the short-lived, scoped permission — which avoids the backend becoming a bandwidth/memory bottleneck for large uploads while keeping the bucket itself private.

## Watch

- [Introduction to Amazon Simple Storage Service (S3)](https://www.youtube.com/watch?v=77lMCiiMilo) — Amazon Web Services (official). Short official primer on buckets, objects, and durability.
- [What is Amazon RDS and How It Works](https://www.youtube.com/watch?v=tLp8pPNdDXQ) — CBT Nuggets. Covers managed backups, Multi-AZ, and read replicas clearly.

## Further reading

- [AWS docs: Amazon S3 storage classes](https://aws.amazon.com/s3/storage-classes/) — the cost/retrieval tradeoffs across storage tiers.
- [AWS docs: Amazon RDS Multi-AZ deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html) — official failover mechanics.
- **Azure/GCP callout:** Azure Blob Storage and Azure SQL/Database for Postgres/MySQL map to S3/RDS respectively; GCP's are Cloud Storage and Cloud SQL — same split between object storage and managed relational databases.
