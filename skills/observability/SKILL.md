---
name: observability
description: Instruments running systems with structured logging, metrics, distributed tracing, and alerting. Use when building any production service, before first deployment, or when production issues are slow to detect and diagnose.
---

# Observability

## Overview

Build visibility into your system before you need it. Observability is not a feature you add after an incident — it is the instrumentation that tells you an incident is happening, where it is happening, and why. A system without observability is a system you cannot operate confidently. You will find out something is broken when a user tells you, not when your monitor fires.

**The core discipline:** Define what "working" means for the service (SLOs) before writing the first line of instrumentation. Metrics without SLOs are numbers without meaning. Alerts without SLOs are guesses.

Observability rests on three pillars — logs, metrics, and traces. Each answers a different question. All three are required for full diagnostic coverage.

## When to Use

Activate this skill when:

- Building any service that will serve production traffic
- Preparing a service for its first production deployment
- On-call engineers cannot diagnose incidents from available data
- Mean time to detection (MTTD) or resolution (MTTR) is too high
- Alerts are firing but nobody knows what to do
- Root-causing a production issue requires SSHing into servers

**When NOT to apply full rigor:**
- Throwaway prototypes with no production traffic
- Internal scripts with a single operator and no SLA

**The observability readiness test:**

> "If the service started returning errors for 10% of users right now, how long would it take to detect it, identify the cause, and fix it?"
>
> - Under 15 minutes → observability is working
> - Over 15 minutes → this skill applies

Flying blind in production is not a speed advantage. It is a debt that compounds with every incident.

## The Three Pillars

### Pillar 1 — Logs

Logs are timestamped records of discrete events. They answer: **what happened?**

**When to log:**
- Application start and stop
- User-initiated actions (create, update, delete)
- All errors and exceptions
- Calls to external APIs and services (with duration)
- Background job start, completion, and failure
- Authentication events (login, logout, failed attempt)

**Format:** structured JSON, always. Plain text log strings are not searchable, not parseable, and not aggregatable. A log that cannot be queried in a crisis is not a log — it is noise.

**Log levels:**

| Level | When to use | Example |
|-------|------------|---------|
| **ERROR** | Something failed that requires attention | Database connection refused; payment failed |
| **WARN** | Something unexpected but recoverable | Retry attempt 2 of 3; cache miss fallback |
| **INFO** | Normal significant events | Request completed; background job finished |
| **DEBUG** | Detailed diagnostic data | Query plan; parsed request body |

Run INFO in production. Enable DEBUG only on demand for specific investigations — never leave DEBUG on permanently.

**Non-negotiable rules:**
- Never log PII — no names, emails, phone numbers, passwords, or tokens
- Use opaque internal IDs (`usr-456`) for correlation, never readable identifiers
- Every log entry must be valid JSON
- Every log entry must include a `correlation_id` linking it to the originating request

### Pillar 2 — Metrics

Metrics are numerical measurements over time. They answer: **how is the system behaving at scale?**

**Metric types:**

| Type | Definition | Examples |
|------|-----------|---------|
| **Counter** | Cumulative value that only increases | Total requests served, total errors, total logins |
| **Gauge** | Current value that rises and falls | Active connections, memory used, queue depth |
| **Histogram** | Distribution of values across buckets | Response time, payload size, job duration |

Use histograms for latency — never counters or gauges. A gauge that records "current latency" is meaningless. A histogram lets you calculate p50, p95, and p99.

**Key metrics every service must expose:**

| Signal | Metric | Why |
|--------|--------|-----|
| Request rate | Requests per second | Demand on the system |
| Error rate | Errors per second; % of requests | Health of the system |
| Latency | p50, p95, p99 response time | User experience |
| Saturation | CPU %, memory %, disk %, queue depth | Headroom remaining |

These four are not aspirational — they are the minimum. A service that cannot report these four numbers is not ready for production.

### Pillar 3 — Traces

Traces record the end-to-end journey of a single request through the system. They answer: **where did this specific request spend its time, and where did it fail?**

Each trace is composed of spans — one span per service or operation the request passed through. Spans record start time, duration, and any errors at that step.

**When tracing is essential:** any architecture where a single user action triggers calls across multiple services. Without traces, you can see that something is slow but not which service is responsible.

**For monoliths:** a correlation ID threaded through all log entries for a single request provides most of the diagnostic value of traces at near-zero cost. Add distributed tracing when the monolith is decomposed into services.

**What every trace must carry:**
- Correlation ID that is also present in all related log entries
- Timing for each service hop
- Error details at the exact step where failure occurred
- User or session context (as opaque IDs, not PII)

## The Four Golden Signals

Google's SRE book identifies four signals that, together, capture the complete health of any production service. Instrument all four before adding any others.

### 1. Latency — How long requests take

Measure p50, p95, and p99. Never use average — a mean response time of 200ms can coexist with a p99 of 30 seconds. The average hides the tail, and the tail is where users leave.

- **p50:** half of requests are faster than this
- **p95:** 95% of requests are faster than this; 5% are slower
- **p99:** the slowest 1% of requests — the worst experience real users see

Alert on p99 exceeding your SLO threshold. Investigate when p95 trends upward.

### 2. Traffic — How much demand is on the system

Measure requests per second, active sessions, or the domain-appropriate equivalent (jobs processed per minute, messages consumed per second).

Traffic monitoring catches two failure modes: unexpected spikes (DDoS, viral event, runaway client) and unexpected drops (ingress failure, upstream outage, deployment that silently broke routing).

Alert on unusual drops as well as spikes — a traffic drop often signals a worse problem than a spike.

### 3. Errors — Rate of failing requests

Measure the percentage of requests returning 5xx errors. Distinguish between client errors (4xx — usually the caller's problem) and server errors (5xx — your problem).

Also track silent failures: requests that return 200 but produce incorrect results. These require business-logic-level assertions in smoke tests — raw HTTP error rates will not catch them.

Alert when error rate exceeds a defined threshold (1% is a common starting point; calibrate to your SLO).

### 4. Saturation — How full the system is

Measure the utilisation of every constrained resource: CPU, memory, disk, database connections, thread pool, message queue depth. Saturation predicts failures before they happen.

A service at 95% memory is not failing yet, but it is one traffic spike away from an OOM kill. Alert before the limit, not after.

Alert at 80% of capacity for any sustained period — this gives time to respond before the resource is exhausted.

## SLI, SLO, and SLA

Three terms that are frequently confused and must be kept distinct.

**SLI — Service Level Indicator**

The actual measurement. A number derived from real system data.

> "99.2% of requests to `/api/visits` completed successfully over the past 30 days."

**SLO — Service Level Objective**

Your internal target for an SLI. The threshold that triggers engineering action when breached.

> "We target 99.5% successful requests on `/api/visits`."

An SLO breach means the engineering team must act — investigate the cause, fix the issue, and review whether the SLO is correctly calibrated. SLOs are owned by engineering.

**SLA — Service Level Agreement**

The contractual promise to customers. Always weaker than the SLO to provide a buffer for recovery.

> "We guarantee 99.0% uptime, measured monthly. Outages beyond this threshold receive service credits."

Breaching an SLA has business and legal consequences. The gap between SLO and SLA is deliberate — if the SLO is 99.5% and the SLA is 99.0%, the team has 0.5% of error budget to consume before customers are contractually affected.

**Define SLOs before instrumenting.** Without targets, you cannot know whether your metrics are good or bad. SLOs make "the system is healthy" a measurable statement, not an opinion.

## Structured Logging Standard

Every log entry must be valid JSON with this shape:

```json
{
  "timestamp": "2026-05-22T14:32:01.123Z",
  "level": "ERROR",
  "service": "visit-service",
  "version": "2.4.1",
  "correlation_id": "req-abc123",
  "user_id": "usr-456",
  "action": "create_visit",
  "duration_ms": 234,
  "error": {
    "code": "validation_failed",
    "message": "arrival_date is required"
  },
  "metadata": {}
}
```

**Field rules:**

| Field | Required | Rule |
|-------|----------|------|
| `timestamp` | Always | ISO 8601 with milliseconds; UTC |
| `level` | Always | One of: ERROR, WARN, INFO, DEBUG |
| `service` | Always | Machine-readable service name; consistent across all instances |
| `version` | Always | Deployed version; essential for correlating issues to releases |
| `correlation_id` | Always | Unique per request; generated at the entry point and propagated |
| `user_id` | When available | Opaque internal ID only — never email, name, or readable identifier |
| `action` | Always | What the code was doing when this log was emitted |
| `duration_ms` | On completions | Wall-clock time for any operation that touches an external system |
| `error` | On ERROR level | Structured error with code and message; never a raw exception string |
| `metadata` | Optional | Arbitrary additional context; must never contain PII |

Do not log plain text strings. `logger.error("Something went wrong")` is not a log entry — it is a mystery. `logger.error({ action: "charge_card", error: { code: "card_declined" }, duration_ms: 430 })` is a log entry.

## Alerting Principles

Alert only on things that require human action. Every alert that fires must have someone who can respond and something they can do. An alert that cannot be acted on is noise — and noise trains engineers to ignore alerts.

**Alert types:**

| Type | When to use | Response required |
|------|------------|------------------|
| **Page** | Production is down; data loss risk; SLA at risk | Immediate — wake the on-call engineer |
| **Notify** | Error rate elevated; SLO at risk; approaching capacity | Check within the hour |
| **Inform** | Unusual pattern; approaching a soft limit | Check when convenient |

**Before creating any alert, answer these four questions:**

```
□ Is it actionable?    Someone can take a specific action to resolve it
□ Is it urgent?        The situation degrades if not addressed promptly
□ Is it accurate?      Low false positive rate — fires rarely when nothing is wrong
□ Does it have a runbook?  The responder knows exactly what to do
```

If any answer is no, do not create the alert yet. Fix the gap first.

**Alert fatigue is a safety failure.** A team that receives 50 alerts per shift learns to dismiss them. When the real incident arrives, it looks identical to the noise. Fewer, higher-quality alerts save more incidents than comprehensive but noisy alerting.

## Runbook Standard

Every alert must link to a runbook. A runbook is the documented procedure for what to do when the alert fires. It is written when the alert is created — not during the incident.

```markdown
## Alert: [Alert Name]

### What is happening

Plain English description of what this alert means.
What is the system doing (or failing to do) that caused this alert to fire?

### Impact

Who is affected and how. Quantify where possible:
"All users attempting to log in will see an error."
"Background jobs are queued but not processing — no data loss, 
but jobs will be delayed."

### Immediate steps

1. Check [dashboard link] — look for [specific pattern]
2. Check [log query link] — filter by `level: ERROR, service: X`
3. If [condition A] → [specific action]
4. If [condition B] → [specific action]
5. If [condition C] → restart service X via [command or link]

### Escalation

If not resolved within 30 minutes → contact [person or team] via [channel].
If data loss is possible → escalate immediately, do not wait 30 minutes.

### Related alerts

- [Link to related runbook 1]
- [Link to related runbook 2]
```

A runbook that says "investigate and fix" is not a runbook. Every step must be specific enough that an engineer unfamiliar with the service can execute it at 3am under stress.

## Dashboard Design

Every service needs a single dashboard. Not one per team, not one per concern — one dashboard that shows the complete health of the service at a glance.

**Layout:**

```
┌─────────────────────────────────────────────────────┐
│  ROW 1: Service health status  ● OK / ◐ Degraded / ● Down  │
├──────────┬──────────┬──────────┬──────────────────────┤
│  ROW 2: Four Golden Signals                          │
│  Latency │  Traffic │  Errors  │  Saturation          │
│  p50/p95 │  req/s   │  error % │  CPU / mem / disk    │
│  p99     │          │          │                      │
├──────────┴──────────┴──────────┴──────────────────────┤
│  ROW 3: Business metrics                              │
│  Visits created/hr │ Logins/hr │ PDFs exported/hr     │
├────────────────────────────────────────────────────── ┤
│  ROW 4: Infrastructure                                │
│  CPU %  │  Memory %  │  Disk %  │  DB connections      │
└─────────────────────────────────────────────────────-┘
```

**Dashboard rules:**
- Default time range: last 1 hour
- Auto-refresh: every 30 seconds in normal operation; every 10 seconds during incidents
- Mobile-viewable — incidents happen outside office hours and away from desks
- SLO thresholds drawn as horizontal reference lines on every relevant graph
- One dashboard per service — fragmented dashboards fragment situational awareness

## Health Check Endpoints

Every service exposes two health check endpoints. These are not optional — they are the interface between the service and the infrastructure that runs it.

**Public health check — used by load balancers and uptime monitors:**

```
GET /health
Authorization: none required
Response: 200 OK
Body: { "status": "ok" | "degraded" | "down" }
```

Returns `200` even when `status` is `"degraded"` — the load balancer interprets any non-200 as a complete failure. Use `"degraded"` to signal reduced capacity without pulling the instance from rotation.

**Detailed health check — used by ops dashboards and on-call engineers:**

```
GET /health/detailed
Authorization: required (internal only — not publicly accessible)
Response: 200 OK
Body:
{
  "status": "ok",
  "version": "2.4.1",
  "uptime_seconds": 86400,
  "database": "ok",
  "cache": "ok",
  "external_services": {
    "stripe": "ok",
    "sendgrid": "degraded"
  }
}
```

The detailed endpoint must reflect real dependency state — it must actually check the database connection and external services, not just return `"ok"` unconditionally. A health check that always returns `"ok"` is worse than no health check, because it creates false confidence.

## Error Tracking

Every unhandled exception in production must be captured with full context. Log entries are not sufficient — they require manual log querying to correlate. An error tracker surfaces new exceptions immediately, groups recurring issues, and shows the rate of occurrence.

**Every captured exception must include:**
- Full stack trace
- Request context: URL, HTTP method, query parameters (no body if it may contain PII)
- User context: internal user ID only — no PII
- Environment: production / staging
- Release version: which deployment introduced the error
- Frequency: is this a new error or an existing one that suddenly spiked?

**Triage rule:** new errors in production after a deployment are caused by the deployment until proven otherwise. Check error tracking immediately after every production deployment (see Stage 7 in `ci-cd-pipeline`).

**Tooling options:** Sentry, Bugsnag, Rollbar, Datadog APM. The specific tool matters less than ensuring it is configured before going to production.

## The Observability Workflow

Complete these steps before a service serves its first production request. Do not treat observability as a post-launch task.

### Step 1: Define SLOs

Before writing any instrumentation code, define what "working" means:

```
SLO worksheet — [Service Name]

Availability:  ___% of requests succeed (target before setting: measure baseline)
Latency:       p99 response time < ___ ms
Error rate:    < ___% of requests return 5xx
Freshness:     (for data pipelines) data no older than ___ minutes
```

If you have no baseline data yet, instrument first, observe for one week, then set SLOs based on observed behaviour. An SLO set without data is a guess — but a documented guess is better than none.

### Step 2: Instrument

Add structured logging and metrics to the service:

- Configure the logging library to emit JSON
- Add correlation ID middleware to generate and propagate request IDs
- Instrument the four golden signals (request rate, error rate, latency histogram, saturation gauges)
- Add health check endpoints (`/health` and `/health/detailed`)
- Connect error tracking (Sentry or equivalent)

### Step 3: Create Dashboard

Before the service handles live traffic, build the dashboard:

- Four golden signals in row 2
- Business metrics specific to this service in row 3
- Infrastructure metrics in row 4
- Draw SLO thresholds as reference lines
- Set default time range to 1 hour and auto-refresh to 30 seconds

### Step 4: Set Alerts

Create alerts only for conditions that require human action:

- Error rate > SLO threshold → page
- p99 latency > SLO threshold → page
- CPU or memory > 80% sustained → notify
- Unusual traffic drop (> 50% below baseline) → page

Write the runbook before enabling each alert.

### Step 5: Error Tracking

Configure error tracking to capture all unhandled exceptions. Verify it is working by intentionally throwing a test exception in staging and confirming it appears in the tracker.

### Step 6: Test the Observability

Deliberately break the service and verify the alerts fire:

```
Test scenarios:
□ Return 500 errors from one endpoint → error rate alert fires
□ Add artificial latency (sleep 5s) → latency alert fires
□ Exhaust database connections → saturation alert fires, /health/detailed reflects it
□ Throw an unhandled exception → error tracker captures it within 60 seconds
□ Verify logs contain correlation_id and no PII
```

Observability that has not been tested under failure conditions has not been verified.

### Step 7: Iterate

After the first month of production:

- Remove alerts with high false-positive rates
- Tighten SLOs if the service consistently exceeds them
- Add missing coverage for error modes discovered in incidents
- Review runbooks — update any steps that turned out to be wrong during a real incident

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We'll add observability once we have users" | The first users are when you most need observability — you have no baseline, no incident history, and no tolerance for extended downtime. |
| "We have logs — that's enough" | Unstructured logs that cannot be queried are archaeological artefacts, not operational tools. Metrics and alerts are what enable detection; logs enable diagnosis. |
| "We'll know when something breaks because users will tell us" | Users don't tell you — they leave. You find out days later when churn spikes. MTTD measured in days is not a strategy. |
| "Average latency is fine for monitoring" | Averages are statistically guaranteed to hide tail latency problems. A p99 of 30 seconds with a p50 of 100ms produces an average of roughly 400ms — which looks acceptable. |
| "Alerting on everything means we won't miss anything" | Alerting on everything means every alert is ignored. The team that cries wolf trains itself to dismiss alerts. Fewer, better alerts catch more incidents. |
| "We'll write the runbooks during the incident" | Writing a runbook during an incident is debugging while reading the debugging guide. Write it when calm, execute it under pressure. |

## Red Flags

- Production incidents are discovered by users before the monitoring fires
- Log entries are plain text strings with no structure
- PII (email addresses, names, passwords) appears in any log or error tracker
- No correlation ID exists — logs for a single request cannot be grouped
- Alerts fire multiple times per day with no corresponding incidents (alert fatigue)
- An alert fires and the on-call engineer does not know what to do
- Average latency is used instead of percentiles
- `/health` always returns `"ok"` regardless of dependency state
- Error tracking is configured but nobody checks it after deployments
- Dashboards exist per team but no single dashboard shows the full service health
- SLOs are written in a document but not measured or enforced

## Verification

Before a service handles production traffic:

```
□ SLOs defined and documented (availability, latency, error rate)
□ Structured JSON logging implemented with all required fields
□ No PII present in any log statement
□ Correlation ID generated and propagated on every request
□ Four golden signals instrumented (request rate, error rate, latency, saturation)
□ /health endpoint returns correct status (no auth required)
□ /health/detailed endpoint checks real dependency state (auth required)
□ Dashboard shows four golden signals with SLO reference lines
□ Alert defined for error rate exceeding SLO threshold
□ Alert defined for p99 latency exceeding SLO threshold
□ Alert defined for saturation > 80%
□ Every alert has a linked runbook
□ Error tracking capturing unhandled exceptions
□ Observability tested under simulated failure (alerts confirmed to fire)
```

---

## Quick Start: Minimum Observability in 1 Hour

The minimum viable setup for a new service. Complete in order.

```
STEP 1 — Structured logging (15 min)
□ Configure logging library to emit JSON
□ Add required fields: timestamp, level, service, version, action
□ Add correlation ID middleware: generate UUID at entry point,
  attach to all logs for the request lifetime
□ Verify: tail logs and confirm every line is valid JSON

STEP 2 — Health checks (10 min)
□ Implement GET /health → { "status": "ok" }
□ Implement GET /health/detailed → check DB + key dependencies
□ Verify: curl both endpoints and confirm responses

STEP 3 — Error tracking (10 min)
□ Install error tracking SDK (Sentry recommended for speed)
□ Configure DSN from environment variable (never hard-coded)
□ Add global unhandled exception handler
□ Verify: throw a test exception in staging, confirm it appears
  in the error tracker within 60 seconds

STEP 4 — Four golden signals (15 min)
□ Instrument request counter (increment on every request)
□ Instrument error counter (increment on every 5xx response)
□ Instrument latency histogram (record duration of every request)
□ Instrument saturation gauges (CPU %, memory % — often free
  from platform/runtime metrics)

STEP 5 — Minimal alerting (10 min)
□ Create one alert: error rate > 5% for 5 consecutive minutes → page
□ Write the runbook for that alert (10 lines minimum)
□ Verify: force errors in staging, confirm alert fires

At the end of this hour you have: structured logs you can query,
health checks the load balancer can use, exceptions captured
automatically, four golden signals measured, and one alert that
fires before users complain. Expand from here.
```

---

## See Also

- For health checks wired into the deployment pipeline, see `skills/ci-cd-pipeline/SKILL.md` (`ci-cd-pipeline`) — Stage 6 and 7 depend on `/health` and post-deploy smoke tests
- For compliance requirements that overlap with audit logging, see `skills/compliance-and-regulatory/SKILL.md` (`compliance-and-regulatory`) — structured audit logs satisfy SOC 2 CC7.2 and ISO 27001 A.12.4
- For observability as a pre-launch requirement, see `skills/shipping-and-launch/SKILL.md` (`shipping-and-launch`)
- For using logs and traces to diagnose production failures, see `skills/debugging-and-error-recovery/SKILL.md` (`debugging-and-error-recovery`)
