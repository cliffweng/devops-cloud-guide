---
title: "12. Observability"
layout: default
nav_order: 13
---

# Observability (logs, metrics, traces, alerts)
{: .no_toc }

*~8 min read*

**🎯 Interview frequent**

## Why it matters

"How do you know it's broken before your users tell you" is asked in nearly every SRE/platform loop, and "we'd check the logs" is an incomplete answer on its own. Observability is the discipline of being able to answer *why* a system is behaving a certain way, not just *whether* it is — and it's the direct prerequisite for [on-call & incident response](../13-oncall-incident-response/), since you can't respond to something you can't see.

## Core concepts

- **Logs, metrics, and traces are the three pillars, and they answer different questions.** Logs are discrete, timestamped events with detail ("request X failed with error Y at 14:32:01") — great for deep-diving one specific thing, expensive to store and slow to search at volume. Metrics are numeric measurements aggregated over time (request rate, error rate, CPU%) — cheap to store, great for dashboards, alerting, and trends, but they lose per-request detail. Traces follow one request across every service it touched — the tool for "which of my 6 microservices is actually adding the latency."
- **The four golden signals (from Google's SRE practice) are the default starting dashboard for any service**: latency (how long requests take), traffic (how much demand there is), errors (rate of failed requests), and saturation (how close a resource — CPU, memory, connection pool — is to its limit). If you can only instrument four things, these are the four.
- **A metric without a threshold is just a number nobody looks at.** Alerting turns metrics into action — "error rate > 5% for 5 minutes" pages someone; the same metric sitting on a dashboard with no alert just quietly documents an outage after the fact instead of catching it in progress.
- **Alert fatigue is a real, self-inflicted failure mode.** Alerts that fire too often, on thresholds too sensitive, or on symptoms that aren't actually actionable train people to ignore or snooze them — the fix is fewer, higher-signal alerts tied to things that actually require a human response, not more alerts covering more metrics.
- **Distributed tracing works by propagating a trace ID across service boundaries.** Every service a request touches logs its work (a "span") tagged with the same trace ID, so a tracing tool can reconstruct the entire request's path and show exactly where time was spent — this is the only practical way to debug latency in a system with more than one or two services talking to each other.
- **Structured logging (JSON, not free-text) is what makes logs actually queryable at scale.** `{"level":"error","user_id":123,"path":"/checkout"}` can be filtered and aggregated by any field; a free-text log line can only be grep'd, which falls apart fast once you have more than one instance producing logs.
- **Dashboards are for humans watching in real time or investigating; alerts are for machines deciding when to interrupt a human.** Confusing the two — alerting on everything visible on a dashboard, or having no dashboard and relying purely on alerts — is a common early mistake.
- **SLIs, SLOs, and error budgets connect observability to a real business decision.** An SLI (service level indicator) is a measured metric (e.g., % of requests under 300ms); an SLO (objective) is the target for it (e.g., 99.5%); the gap between 100% and your SLO is your **error budget** — a concrete, spendable allowance for risk (deploys, experiments) before you're violating your own reliability promise. This is how "how reliable should this be" becomes a number instead of a vibe.

## Mental model

```
  request flows through: LB -> service A -> service B -> database
        |
        v
  [ logs ]     per-event detail, "what exactly happened at 14:32:01"
  [ metrics ]  aggregated over time, "what's the error rate right now"
  [ traces ]   one request's full path, "which hop added the latency"
        |
        v
  metrics + thresholds --> [ alerts ] --> pages a human (see on-call, topic 13)
  dashboards --> humans watching/investigating, not paging anyone automatically

  SLI (measured) vs. SLO (target) --> gap = error budget (spendable risk allowance)
```

Think of metrics as a car's dashboard gauges (fast glance, aggregated), logs as the mechanic's detailed printout after popping the hood (slow, detailed, for one specific problem), and traces as a GPS trace of exactly which streets a specific trip took — you reach for each one for a different kind of question.

## Interview questions

1. **A service's error rate metric shows a spike, but you need to know exactly which requests failed and why. Why can't the metric alone answer that, and what would you check next?**
   Answer: A metric is an aggregate number — it tells you *that* something changed, not *which specific requests* or *why*. You'd go to logs (ideally structured, filterable by request ID, user, or endpoint) for the per-event detail, and if the failure spans multiple services, a distributed trace to see exactly where in the request's path the failure or latency originated.

2. **What are the four golden signals, and why are they a reasonable default even for a service you've never worked on before?**
   Answer: Latency, traffic, errors, and saturation. They're a reasonable default because together they answer the four questions that matter for almost any service regardless of what it does internally: is it slow, how much load is it under, is it failing, and is it close to running out of some resource — a solid baseline dashboard before you know anything domain-specific about the service.

3. **A team's on-call engineers are ignoring pages because "it's always something minor." What's actually going wrong, and how would you fix it?**
   Answer: This is alert fatigue — alerts are firing too often or on thresholds too sensitive to reliably signal something that actually needs a human, which trains people to tune them out (including, eventually, the rare real emergency). The fix is auditing existing alerts for actionability (does this alert firing always require someone to actually do something right now?) and either raising thresholds, adding duration requirements ("only alert if sustained for 5 minutes," not on a single blip), or deleting alerts that don't meet that bar.

4. **How does distributed tracing actually work under the hood — what makes it possible to reconstruct one request's path across five different services?**
   Answer: A trace ID is generated (usually at the first service the request hits) and propagated forward through every subsequent service call, typically via a request header; each service records its own span (start/end time, metadata) tagged with that same trace ID. A tracing backend collects all the spans sharing a trace ID from every service and reconstructs the full timeline, showing exactly how much time each hop took.

5. **What's the difference between an SLI and an SLO, and how does an "error budget" turn that into an actual decision-making tool?**
   Answer: An SLI is the actual measured value of some indicator (e.g., "99.2% of requests completed under 300ms last week"); an SLO is the target you've committed to for that SLI (e.g., "99.5% under 300ms"). The error budget is the allowed gap between 100% and the SLO — a concrete, spendable amount of acceptable failure that lets teams make a data-driven call like "we're within budget, ship the risky deploy" or "we've blown the budget this month, freeze non-critical changes until we're back under it," instead of debating reliability in the abstract.

6. **Why is free-text logging (`print("user login failed")`) worse than structured logging at scale, specifically?**
   Answer: Free-text logs can only be searched by string matching (grep-style), which becomes slow and imprecise once you have many instances producing high-volume logs, and you can't reliably aggregate or filter by a specific field (like `user_id` or `status_code`) without fragile string parsing. Structured logs (JSON with consistent keys) let a log aggregation tool index and query by exact field values directly — "show me every failed login for user 123 in the last hour" becomes a precise query instead of a hopeful regex.

## Watch

- [SRE Golden Signals Explained](https://www.youtube.com/watch?v=-U9E1PhrM3o) — IBM Technology. Direct explanation of latency/traffic/errors/saturation and why they anchor SRE dashboards.
- [Exploring logs, metrics, and traces with Grafana](https://www.youtube.com/watch?v=1q3YzX2DDM4) — Grafana (official). Hands-on look at how the three pillars show up in a real observability tool.

## Further reading

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) — the original source for the four golden signals.
- [OpenTelemetry docs: Observability primer](https://opentelemetry.io/docs/concepts/observability-primer/) — vendor-neutral reference on logs/metrics/traces and how they relate.
- [Google SRE Workbook: Implementing SLOs](https://sre.google/workbook/implementing-slos/) — practical guide to defining SLIs/SLOs/error budgets.
