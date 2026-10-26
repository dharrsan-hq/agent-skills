---
name: SRE Architecture
description: SRE practices distilled from the Google SRE book — SLOs, error budgets, observability, incident response, on-call, and production architecture review rules
---

# SRE Architecture

Authoritative source: *Site Reliability Engineering: How Google Runs Production
Systems* (Google). Apply the rules below when designing, reviewing, or operating
production systems.

## Core mental model

- **Reliability is a feature with a cost.** Every system has a "sweet spot" of
  reliability — 100% reliability is the wrong target: it's unattainable and it
  forecloses feature velocity. Reliability should be matched to the business
  need, not maximized.
- **SRE is what happens when you ask a software engineer to design an operations
  function.** Engineering, not manual toil, is the way systems get reliable.
- **The fundamental SRE activity is not firefighting — it is engineering work
  that prevents fires.** Operational work (toil) must be measured and capped.
- Systems fail. Design for *recovery* and *graceful degradation*, not for the
  absence of failure.

## SLOs, SLIs, and error budgets

- **SLI** = the indicator (a metric): e.g., fraction of requests served faster
  than 250 ms; fraction of requests answered correctly; fraction of requests
  served at all. **SLA** = the contract with customers (includes penalties).
- **Choose SLIs from the user's perspective**, not from the server's. Latency
  measured server-side misses client-side queuing, rendering, and network
  latency. Prefer: request latency, error rate, system throughput, availability,
  durability.
- **Every SLI should be measured at the request level**, aggregated over a
  rolling window (e.g., 28 days) rather than per-minute averages. Averages hide
  the tail; use percentiles (50th, 90th, 99th, 99.9th).
- **The 99.9% trap:** at 99.9%, 43 minutes of downtime per month is your error
  budget. At 99.99%, it's 4.3 minutes. Every "nine" costs an order of magnitude
  more effort. Don't over-promise.
- **Error budget** = 100% − SLO = the tolerated unreliability. The budget is the
  shared currency between product and reliability:
  - Budget remaining → ship features freely.
  - Budget exhausted → freeze features, focus on reliability. This rule is
    non-negotiable, decided jointly by product and SRE, and enforced.
- **SLOs are set by what users need, not what the system can deliver.** The
  system's actual performance (e.g., 99.99%) should exceed the SLO (e.g., 99.9%)
  — this headroom is the error budget. If the SLO matches the actual performance
  exactly, you can never ship.
- **Never set a 100% SLO.** It forces zero-error-budget operation.
- Don't include in your SLO the failures that are the users' fault (e.g., they
  misconfigured their client) or that are outside your control (e.g., an upstream
  ISP outage). Document these exclusions explicitly.
- Track the burn rate of the error budget: `burn_rate = actual_error_rate / SLO_error_rate`.
  Burn > 1 means you're eating the budget faster than planned. **Fast burn
  (e.g., 14.4x) pages; slow burn (e.g., 1x) tickets.**

## Service architecture and failover

- **Design for graceful degradation.** A system should fail with reduced
  functionality rather than completely. Feature flags, cache serving, and
  read-only mode are degradation strategies.
- **Layered defense:** throttle at the front, shed load, and backpressure —
  rather than letting one failing dependency cascade into a whole-system outage.
- **Failover pairs / active-passive / active-active:** consider the recovery
  time objective (RTO) and recovery point objective (RPO) when choosing.
- **N + 1 capacity**: always have at least one more replica than needed to
  serve the load. Every server should be able to fail without user impact.
- **Plan for failure of the dependency itself.** Your dependency's SLA compounds
  with your own: if your dependency is 99.9% and you need to be 99.9%, you can't
  rely on that dependency in the critical path without a mitigation.
- **Isolate failure domains:** blast radius should be small. Partition by
  customer, region, shard, or feature so one failure doesn't take out everything.
- **Canary / staged rollout:** push to a small percentage of traffic first,
  watch for anomalies, then expand. Rollback should be as fast as push.
- **Feature flags decouple deploy from release.** Ship dark, enable for canary
  cohort, then fully.

## Observability

- **The three pillars — metrics, logs, traces — are complementary, not redundant.**
  Metrics tell you *something is wrong*, logs tell you *what happened*, traces
  tell you *where in the system it happened*.
- **Metrics first** — cheap, low cardinality, great for alerting. Logs second —
  detailed, expensive, good for debugging. Traces third — essential for
  distributed systems.
- **The Four Golden Signals** (from the Google SRE book) — monitor all four for
  every user-facing service:
  1. **Latency** — time to serve a request (success *and* failure latency; a
     failing request may be fast or slow).
  2. **Traffic** — demand placed on the system (requests/sec, network I/O).
  3. **Errors** — rate of failed requests (explicit 5xx, implicit 200s with
     wrong content).
  4. **Saturation** — how "full" the service is (CPU, memory, queue depth, pool
     exhaustion, connection count).
- **Percentiles, not averages.** p50 hides the users who are suffering. Use
  p50 (median), p90, p99, p99.9. Be aware that aggregating percentiles across
  machines is not mathematically exact.
- **Cardinality discipline:** high-cardinality labels (user IDs, request IDs) in
  metrics blow up storage. Put those in logs/traces, not in metrics.
- **Log levels:** use structured logging (key=value or JSON) with levels
  (DEBUG, INFO, WARN, ERROR). Emit logs at the point of decision, not at every
  layer. Log at 10% sampling for high-volume INFO in hot paths.
- **Traces** are essential for distributed systems — instrument cross-service
  calls with a consistent trace ID (e.g., OpenTelemetry, W3C `traceparent`).
- **Dashboards** should answer "is this system healthy?" in 10 seconds. Design
  top-down: one overview dashboard per service, then detail dashboards.
- **A dashboard is a communication device.** If nobody looks at it, delete it.

## Alerting and paging hygiene

- **Every page should be actionable and urgent.** If a page doesn't require
  human action *right now*, it should be a ticket or a dashboard annotation.
- **Symptoms, not causes.** Page on user-facing symptoms (latency too high,
  error rate too high), not on intermediate causes (CPU high). A high CPU is a
  data point, not a page.
- **Multi-window, multi-burn-rate alerts:** e.g., page on 14.4x burn over 1 hour
  *and* 6x burn over 6 hours; ticket on 3x burn over 24 hours. Avoid per-minute
  threshold alerts — they are noisy and flappy.
- **Alert fatigue is real:** if a page fires more than once a week and requires
  no action, remove it. Every noisy page costs the team trust.
- **Every alert has a runbook** — a linked document explaining what to do.
  An alert without a runbook is a bug.
- **Never page on a metric that has a known cause and a safe self-healing
  mechanism** (e.g., a Kubernetes pod restart that auto-recovers).
- Escalation policy: if a page is unacknowledged within N minutes, escalate to
  the next tier.

## Capacity planning

- **Capacity is a function of:** demand forecast (traffic growth), headroom
  (margin), and provisioning latency. Provision for peak demand + growth, plus
  a safety margin for failover.
- **N + 1 / N + 2**: capacity planning should account for at least one failure
  without degradation (N + 1) or two (N + 2 for large fleets).
- **Forecast:** use at least 3 months of history to forecast growth; project at
  least 6 months ahead because provisioning lead times (hardware procurement,
  cluster expansion) are long.
- **Vertical scaling (bigger boxes)** is simpler but hits hard limits and is
  risky for failover; **horizontal scaling** is the default for stateless
  services; **sharding** is required for stateful services.
- **Utilization targets:** running at 80%+ CPU utilization means you cannot
  absorb a spike or a failure; 40–60% is the typical healthy target for
  latency-critical services.

## Incident response

- **Declare an incident early.** It's better to declare and stand down than to
  under-react. The incident commander (IC) is a role, not a job title.
- **Incident roles:**
  - **Incident Commander (IC)** — coordinates the response, delegates, doesn't
    do hands-on debugging themselves.
  - **Operations Lead (Ops)** — does hands-on work (restarts, rollbacks).
  - **Communications Lead (Comms)** — updates stakeholders, status page,
    customers.
- **Roles should be assumed by whoever is free, not whoever is senior.**
  These roles are trained, not improvised.
- **Timeline discipline:** log every action with timestamps — what was changed,
  when, by whom, and the observed effect. This is essential for the postmortem.
- **During the incident:** stabilize first, diagnose second. Rollback is often
  faster than root-cause. Reverting the last change is the first hypothesis.
- **Avoid heroics:** a single exhausted engineer debugging for 12 hours makes
  worse decisions. Hand off, rotate, sleep.
- **Status page / customer communication:** honest, periodic (e.g., every 30
  min), and action-oriented ("we are rolling back a change"), not speculative.
- **Communication templates:** pre-write status-page updates and customer
  emails for common incident classes. Improvising under stress produces bad copy.

## Postmortems

- **Blameless postmortem culture.** The question is not "who caused this?" but
  "what conditions allowed this to happen, and how do we make that impossible?"
- **Every user-facing incident gets a postmortem**, written within 48 hours,
  reviewed at a weekly meeting.
- **Postmortem structure:** incident summary (what happened, when, duration,
  impact), timeline of events, root cause (the technical cause, not the person),
  what went well, what went wrong, action items.
- **Action items are the point.** Every action item has an owner and a due
  date. Action items that are "make sure people are careful" are not action
  items — make the failure impossible via automation, guardrails, or design.
- **Track action items to completion** in a backlog with regular review.
  Unfinished postmortem actions are the leading indicator of repeat incidents.
- **Blameless doesn't mean consequence-free.** It means the postmortem is
  written without assuming bad intent, focusing on systemic causes.

## On-call and toil

- **Toil** = operational work that is manual, repetitive, automatable, tactical,
  has no enduring value, and scales with service growth. Track it and keep it
  below ~50% of time; below that, engineers can do engineering work.
- **On-call rotations:** at most one week of on-call at a time; escalate after
  30 min unacknowledged; have a documented handoff; give the on-call engineer a
  clear "operational load" budget (e.g., max 2 pages per shift — if exceeded,
  the on-call is not sustainable).
- **On-call is a first-class engineering activity.** If on-call is painful,
  the fix is engineering, not endurance.
- **Every on-call shift should have a pre-shift checklist:** read open
  incidents, check dashboards for anomalies, check upcoming deployments and
  canary status, read the current runbook index.
- **Playbooks > tribal knowledge.** Every recurring manual operation should be
  scripted or documented in a runbook; if it's done twice manually, it's
  automatable.

## Data integrity and backups

- **Backups are not a recovery plan.** A backup is only useful if you can
  restore it, within the RTO, and you've *tested* the restore.
- **Plan for the worst-case data loss:** if a database is corrupted, how far
  back can you recover? RPO is the answer; RTO is how long it takes.
- **Data integrity is more critical than availability** — a corrupted database
  served at 100% uptime is worse than a database that is down but recoverable.
- **Plan for data loss scenarios:** accidental deletion, software bugs, data
  corruption, operator error. Mitigations: soft deletes, time-travel/snapshot,
  checksums, write-ahead logging, and restores.
- **24 recovery scenarios** (from the Google SRE book): plan for each of the
  key data-loss combinations (disk failure, corruption, operator error, software
  bugs, region loss, etc.). Each scenario needs a documented recovery path.

## Production readiness checklist

- [ ] SLOs defined and agreed with product; error budget policy in place.
- [ ] SLIs measured at the request level, with dashboards for all four golden
      signals (latency, traffic, errors, saturation).
- [ ] Alerts are symptom-based, multi-window, multi-burn-rate, each with a
      runbook link.
- [ ] Load balancing and failover configured; tested with a real node failure.
- [ ] Capacity plan: N + 1 minimum; forecast documented; utilization targets set.
- [ ] Canary / staged rollout configured; rollback is a single command and
      tested.
- [ ] Feature flags for risky features; dark launch capability.
- [ ] Backups: automated, offsite, and — critically — restore-tested.
- [ ] On-call rotation exists with a runbook index and escalation policy.
- [ ] Incident response roles defined; incident commander trained.
- [ ] Postmortem process exists and blameless; action items tracked.
- [ ] Dependencies: each has a documented SLA and a fallback/degradation plan.
- [ ] Rate limiting / load shedding configured at the front door.
- [ ] Logs are structured, sampled, and retained per policy; traces wired
      across services with consistent trace IDs.
- [ ] Toil measured and below 50% of the team's time.
