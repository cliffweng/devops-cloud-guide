---
title: "09. Networking & HTTPS"
layout: default
nav_order: 10
---

# Networking & HTTPS (DNS, TLS, ALB)
{: .no_toc }

*~9 min read*

**🎯 Interview frequent**

## Why it matters

"Trace what happens when a user hits your site" is one of the most reliable ways interviewers check whether you actually understand the systems you've deployed, versus having just clicked "deploy" and moved on. DNS, TLS, and load balancers are the connective tissue between "a user typed a URL" and "your app's code actually ran" — and debugging any of the three is a real, common on-call skill.

## Core concepts

- **DNS turns a name into an IP address, through a chain of lookups.** Your browser asks a resolver, which walks from root servers → TLD servers (`.com`) → your domain's authoritative nameservers, which hold the actual records. Results are cached at every layer according to each record's **TTL** (time-to-live) — this is exactly why DNS changes don't take effect instantly everywhere: caches at resolvers and browsers hold the old answer until their TTL expires.
- **Record types answer different questions.** `A`/`AAAA` map a name to an IPv4/IPv6 address; `CNAME` aliases one name to another name (can't coexist with other records on the same name); `ALIAS`/AWS's `Alias` records solve the same aliasing problem at the zone apex where `CNAME` isn't allowed; `MX` routes email, not web traffic.
- **TLS (what makes HTTP "HTTPS") gives you encryption, integrity, and identity verification** — not just "encrypted," but also a guarantee the server is who it claims to be, via a certificate signed by a trusted Certificate Authority. The handshake negotiates a shared symmetric key (asymmetric crypto is comparatively slow, so it's only used to bootstrap trust and exchange the key that does the actual bulk encryption).
- **A certificate is bound to a specific hostname (or wildcard) and has an expiration date.** An expired cert breaks HTTPS for everyone hitting that domain — one of the most common self-inflicted outages, and why automated renewal (AWS Certificate Manager, Let's Encrypt/Certbot) is the standard, not manual renewal on a calendar reminder.
- **A load balancer is the front door that spreads traffic across multiple backend instances**, and is also where TLS is commonly terminated (decrypted) — meaning traffic from the internet to the load balancer is encrypted, while traffic from the load balancer to your backend instances is often plain HTTP inside the private network (see [AWS mental model](../06-aws-mental-model/)'s public/private subnet split). This is a deliberate simplification: your app servers don't need to manage certificates at all.
- **AWS's Application Load Balancer (ALB) operates at layer 7 (HTTP/HTTPS)** — it can route based on URL path or hostname, supports health checks that pull unhealthy instances out of rotation automatically, and is the standard front door for HTTP(S) services on AWS. A **Network Load Balancer (NLB)** operates at layer 4 (raw TCP/UDP) for cases needing extreme throughput or non-HTTP protocols — most web apps want an ALB, not an NLB.
- **Health checks are what makes a load balancer more than a dumb traffic splitter.** The LB periodically hits a defined endpoint (e.g., `/health`) on each backend instance; instances that fail enough consecutive checks get removed from rotation until they pass again — this is the actual mechanism behind "traffic automatically stops going to a crashed instance."
- **A request's full round trip** ties all of this together: DNS resolves the hostname to the load balancer's address → TLS handshake establishes an encrypted connection to the load balancer → the load balancer picks a healthy backend instance and forwards the request (often re-encrypting, or not, depending on setup) → the response travels back the same path.

## Mental model

```
 browser
   |  1. DNS lookup: example.com -> resolver -> ... -> A/ALIAS record -> LB's address
   v
 [ TLS handshake: browser <-> load balancer ]
   |  cert verified, symmetric session key established
   v
 [ ALB, layer 7 ]
   |  health checks decide which backend instances are "in rotation"
   |  routes by path/host, forwards request (often plain HTTP inside the VPC)
   v
 backend instance (EC2 / ECS task) -- private subnet, no direct internet exposure
   |
   v
 response travels back: instance -> ALB -> (still-encrypted) TLS session -> browser
```

Think of DNS as looking up an address in a phone book that multiple people cache copies of, TLS as sealing the envelope you send once you have that address, and the load balancer as the front desk that decides which of several available people actually opens the envelope.

## Interview questions

1. **You updated a DNS record to point to a new server, but some users still hit the old one an hour later. Why?**
   Answer: DNS answers are cached at multiple layers — the user's OS, browser, and any resolvers along the path — each honoring the record's TTL before re-querying. If the TTL was, say, 24 hours, caches that fetched the old answer before your change won't re-check until their cached copy expires, regardless of how quickly the authoritative record itself changed.

2. **What does TLS actually guarantee beyond "the data is encrypted"?**
   Answer: It also guarantees integrity (the data wasn't tampered with in transit — any modification is detectable) and server identity (the certificate is signed by a trusted CA, verifying you're actually talking to the domain you think you are, not an attacker intercepting the connection). Encryption alone, without identity verification, wouldn't stop a man-in-the-middle from presenting their own encrypted connection and impersonating the real server.

3. **A site's HTTPS suddenly breaks for everyone, with no recent deploy. What's a common, easy-to-overlook cause, and how do you prevent it recurring?**
   Answer: An expired TLS certificate — certs have a fixed validity window, and if renewal isn't automated, it's easy to let it lapse without anyone noticing until it breaks in production. The fix is automated renewal (AWS Certificate Manager auto-renews certs it manages; Let's Encrypt via Certbot on a cron/systemd timer) rather than relying on someone remembering to renew manually.

4. **Why would you choose an Application Load Balancer over a Network Load Balancer for a typical web API?**
   Answer: An ALB operates at layer 7 and understands HTTP/HTTPS specifically — it can route by URL path or hostname, inspect headers, and run HTTP-level health checks, which is exactly the flexibility a typical web API's routing needs. An NLB operates at layer 4 (raw TCP/UDP) with no visibility into HTTP semantics — the right choice when you need extreme throughput/low latency or you're load-balancing a non-HTTP protocol, not the default for a standard web API.

5. **Where does TLS typically get "terminated" in a load-balanced AWS setup, and why is that a deliberate design choice rather than a shortcut?**
   Answer: TLS is typically terminated at the load balancer — it decrypts incoming HTTPS traffic and forwards it to backend instances, often over plain HTTP within the private VPC network. This is deliberate: it centralizes certificate management in one place (the LB) instead of every backend instance needing its own certificate and renewal process, and it's considered acceptable because the LB-to-backend hop stays inside a private, non-internet-exposed network — the security boundary that matters (the public internet hop) is still fully encrypted.

6. **A load balancer keeps sending traffic to an instance that's actually crashed. What's likely misconfigured?**
   Answer: The health check — either it's not configured at all, is checking the wrong endpoint/port, has thresholds too lenient to catch the failure quickly, or the "unhealthy" endpoint being checked doesn't actually reflect the app's real health (e.g., it always returns 200 regardless of the app's actual state). A load balancer only removes an instance from rotation based on what its configured health check reports — it has no independent way to know an instance is unhealthy.

## Watch

- [Everything You Need to Know About DNS: Crash Course System Design #4](https://www.youtube.com/watch?v=27r4Bzuj5NQ) — ByteByteGo. Concise walkthrough of the DNS resolution chain and record types.
- [SSL, TLS, HTTPS Explained](https://www.youtube.com/watch?v=j9QmMEWmcfo) — ByteByteGo. Covers the handshake, certificates, and why HTTPS actually protects you.
- [AWS Elastic Load Balancing Introduction](https://www.youtube.com/watch?v=qpHLRc4Qt1E) — Stephane Maarek. ALB vs. NLB and health check mechanics on AWS specifically.

## Further reading

- [AWS docs: What is Elastic Load Balancing?](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) — ALB/NLB/CLB comparison, official.
- [AWS docs: AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html) — automated cert issuance/renewal for AWS-fronted domains.
- **Azure/GCP callout:** Azure Application Gateway / Load Balancer and GCP's Cloud Load Balancing play the ALB/NLB role; Azure DNS and Cloud DNS are the equivalent managed DNS services.
