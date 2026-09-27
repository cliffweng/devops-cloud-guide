---
title: "13. On-call & incident response basics"
layout: default
nav_order: 14
---

# On-call & incident response basics
{: .no_toc }

*~7 min read*

**Interview occasional**

## Why it matters

Every SRE/platform role, and a lot of SWE roles, eventually put you on an on-call rotation — a page at 2am is not the time to be figuring out your team's process for the first time. This topic covers the basic shape of on-call and incident response so you can talk credibly about it in an interview, and actually function on day one of your first rotation.

## Core concepts

- **On-call means being the person (or one of a rotating few) who gets paged when something automated decides a human needs to look now.** The page comes from an alert (see [Observability](../12-observability/)) crossing a threshold, routed through a paging tool (PagerDuty, Opsgenie) that handles escalation if the primary on-call doesn't acknowledge in time.
- **Severity levels exist so everyone agrees on urgency without arguing about it mid-incident.** A common scheme: SEV1 (full outage / major customer impact, drop everything), SEV2 (significant but partial impact), SEV3 (minor, can wait for business hours). Assigning severity early — even if it changes later — is what determines how many people get pulled in and how fast.
- **The incident commander (IC) role separates "who's fixing it" from "who's coordinating."** On any incident big enough to need more than one person, the IC tracks status, makes the call on next steps and escalation, and communicates out — explicitly *not* the same person heads-down debugging, because context-switching between "deep technical fix" and "coordinate and communicate" badly serves both.
- **Mitigate first, root-cause later.** The immediate goal during an active incident is stopping user impact — rolling back a bad deploy, failing over, scaling up, disabling a feature flag — not fully understanding *why* it broke. Root cause analysis happens after things are stable, in the postmortem, when there's no pressure clouding the investigation.
- **Runbooks turn "someone once knew how to fix this" into a repeatable, low-stress procedure.** A good runbook for a known failure mode gives the on-call engineer concrete steps (which dashboard to check, which command rolls back, who to escalate to) instead of them reconstructing the fix from memory under pressure at 2am.
- **A blameless postmortem is a written retrospective on the incident, focused on systemic causes, not individual fault.** "Why did our systems and processes allow this to happen" produces fixes (better alerting, a missing test, a runbook gap); "who broke it" produces defensiveness and people hiding mistakes next time — teams that blame people over systems get worse incident data over time, not better.
- **A good postmortem includes a timeline, impact, root cause, and concrete action items with owners.** An action item with no owner or no deadline reliably never gets done — the postmortem's value is almost entirely in whether its action items actually get executed, not in the document itself.
- **Rotations need to be sustainable, not heroic.** Reasonable rotation length (a week is common), genuine off-hours coverage so the same person isn't perpetually on, and enough alert quality (see [Observability](../12-observability/)'s alert fatigue) that being on-call doesn't mean getting paged nightly for noise — burnout from a badly-run rotation is a top reason engineers leave infra-heavy roles.

## Mental model

```
  alert fires (metric crosses threshold)
        |
        v
  paging tool escalates: primary on-call -> (no ack) -> secondary -> manager
        |
        v
  severity assigned (SEV1/2/3) -----> IC assigned if multi-person incident
        |
        v
  MITIGATE first (rollback / failover / feature flag off) -- stop the bleeding
        |
        v
  incident resolved, impact stopped
        |
        v
  blameless postmortem: timeline + root cause + action items w/ owners
        |
        v
  action items feed back into: better alerts, new runbook, code fix, test added
```

Think of the incident itself as a fire (put it out first, ask how it started once it's out) and the postmortem as the fire-code review afterward (why did it spread that fast, what would've contained it sooner) — mixing up the two timelines is how incidents drag on and postmortems turn into blame sessions.

## Interview questions

1. **Why does the incident commander role exist separately from the person actually fixing the problem?**
   Answer: Deep technical debugging and incident coordination (tracking status, deciding on escalation, communicating to stakeholders) require different kinds of attention, and doing both at once means either the fix or the communication suffers — usually communication, since it feels less urgent moment-to-moment than the technical problem. Splitting the roles lets the responder stay heads-down on the fix while the IC keeps the bigger picture moving.

2. **During an active SEV1, should the team prioritize finding the root cause or mitigating user impact first? Why?**
   Answer: Mitigate first — rolling back, failing over, or disabling the offending feature stops the bleeding fastest and is usually achievable without fully understanding *why* something broke. Root-causing under time pressure, with users actively impacted, is slower and more error-prone than doing it calmly afterward in the postmortem, once the fire is actually out.

3. **What makes a postmortem "blameless," and why do teams insist on that framing even when a specific person's mistake was the proximate trigger?**
   Answer: Blameless means the analysis focuses on the systemic conditions that allowed the incident to happen and reach production impact (missing tests, insufficient alerting, unclear runbooks, a review process that didn't catch it) rather than assigning fault to whoever happened to trigger it. Anyone could have made that same mistake given the same gaps — blaming the individual fixes nothing structural and teaches people to hide near-misses and mistakes in the future, which makes the *next* incident's data worse, not the current one better.

4. **A postmortem lists "improve monitoring" as an action item with no owner or deadline. What's wrong with that, and how would you fix it?**
   Answer: An action item with no owner and no deadline has no one accountable for actually doing it, so it reliably gets deprioritized and forgotten — the postmortem's real value comes from action items getting executed, not from having identified them. It needs a specific owner, a concrete and scoped description (not "improve monitoring" but "add an alert on X metric exceeding Y for Z minutes"), and a deadline or tracked ticket.

5. **Your team's on-call engineers are getting paged multiple times a night for issues that self-resolve within a minute. What's the actual problem, and what would you change?**
   Answer: This is alert fatigue/noisy alerting, not a genuine on-call load problem — alerts are firing on transient blips rather than sustained, actionable conditions. The fix is tuning the alert thresholds (e.g., requiring the condition to persist for several minutes before paging, not firing on a single data point) and auditing which alerts are actually actionable versus just noisy signal that should live on a dashboard instead of paging anyone.

6. **What's the purpose of a runbook, and what happens to incident response quality when a known, recurring failure mode doesn't have one?**
   Answer: A runbook encodes the known fix for a specific, recurring failure mode into concrete steps, so the on-call engineer doesn't have to reconstruct the solution from memory or first principles under time pressure. Without one, every recurrence of that same failure takes longer to resolve (whoever's on-call has to rediscover the fix, possibly making mistakes under pressure that someone who wrote the runbook already learned to avoid) and the resolution quality depends entirely on who happens to be on-call that week.

## Watch

- [Understanding On-Call Rotation in Site Reliability Engineering](https://www.youtube.com/watch?v=VpK6UxYqRjc) — Random Thoughts Tech. Covers rotation structure, escalation, and sustainability.
- [SEV1 SEV2 SEV3 Explained: Incident Severity Levels Guide](https://www.youtube.com/watch?v=Dn4GHh2RCEg) — CodeLucky. Walks through how severity levels drive response urgency.

## Further reading

- [Google SRE Book: Managing Incidents](https://sre.google/sre-book/managing-incidents/) — the incident commander model and incident lifecycle, from the source.
- [Google SRE Book: Postmortem Culture](https://sre.google/sre-book/postmortem-culture/) — why blameless postmortems work and how to run one.
- [PagerDuty: Incident Response documentation](https://response.pagerduty.com/) — a widely-used, freely available incident response process reference.
