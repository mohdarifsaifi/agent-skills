---
name: multi-repo-microservices
description: Enforces clear boundaries, contracts, and coordination patterns when an application grows beyond a single repository into multiple services. Use when splitting a monolith, adding a new service, or when deploying one service requires deploying another.
---

# Multi-Repo Microservices

## Overview

Define boundaries before you split. The most common failure mode in microservices adoption is not technical — it is organisational. Teams decompose a monolith by technical layer (a "frontend service," a "database service") or by team ownership without regard for domain boundaries, and produce a distributed monolith: the worst of both worlds. All the operational complexity of microservices, none of the independence benefits.

**The core discipline:** A service boundary is only valid if the service can be deployed, developed, scaled, and failed independently. If any of those four are not true, the boundary is in the wrong place. Fix the boundary before writing the first line of service code.

A well-structured monolith is easier to operate than a poorly-bounded microservices system. The decision to split is irreversible in practice — migration paths back to a monolith are painful. Choose carefully.

## When to Use

Activate this skill when:

- Planning to extract a service from a monolith
- Adding a new service to an existing multi-service system
- Defining how two services will communicate
- Setting up a new repository in a multi-repo system
- Deployment of one service requires deploying another (this is a warning sign — read this skill before proceeding)

**When NOT to split:**
- A monolith that is well-structured and working — do not split prematurely
- Teams smaller than 8 engineers — the coordination overhead of microservices exceeds the productivity of a team this size
- A system whose domain boundaries are not yet understood — understand the domain in a monolith first, then extract

**The splitting trigger test:**

> "Is the monolith's problem that it cannot scale, or that teams cannot work independently? Or both?"
>
> - Scaling problem only → consider vertical scaling or read-replicas first
> - Team independence problem only → enforce module boundaries in the monolith first
> - Both → microservices may be justified; apply this skill in full
> - Neither → do not split

See the decision tree at the end of this skill before committing to a microservices architecture.

## The Golden Rule

A service is correctly bounded when it satisfies all four independence tests:

```
INDEPENDENCE TEST — [Service Name]

□ Deployed independently   Can this service be deployed without deploying
                           any other service?

□ Developed independently  Does one team own this service end-to-end,
                           with no shared code paths that require
                           cross-team coordination to change?

□ Scaled independently     Can this service be scaled (more instances,
                           more resources) based on its own load, without
                           scaling services it depends on?

□ Failed independently     If this service goes down, do other services
                           degrade gracefully rather than failing completely?
```

If any test fails, the boundary is wrong. Do not proceed with implementation until all four pass. The cost of fixing a wrong boundary after services are in production is an order of magnitude higher than fixing it before writing a line of code.

## Service Boundary Patterns

### Bounded Context (Domain-Driven Design)

Each service owns one business domain completely. A bounded context is the natural unit of service decomposition — it corresponds to a cohesive set of business concepts that change together and are owned by the same team.

Services communicate at their boundaries via APIs or events. They never reach across boundaries into each other's internal state.

**Example boundaries for an enterprise visitor management platform:**

| Service | Domain | Owns |
|---------|--------|------|
| **Identity Service** | Users, authentication, permissions | User accounts, roles, sessions, access tokens |
| **Visit Service** | Visits, scheduling, activities | Visit records, days, itineraries, status transitions |
| **Directory Service** | Restaurants, partners, hotels | Venue catalogue, partner profiles, availability |
| **Notification Service** | Emails, alerts, reminders | Message templates, delivery state, retry logic |
| **Export Service** | PDF and Excel generation | Document templates, rendering, file storage |
| **Analytics Service** | Usage metrics, reporting | Event aggregations, dashboards, data retention |

Each boundary is designed so that the team owning a service can design, build, deploy, and operate it without requiring coordination with teams that own other services — except through the published API contract.

### Database Per Service

Each service has its own database. This is not a preference — it is the structural guarantee of independence.

A shared database between two services means:
- A schema change in one service can silently break the other
- The services cannot be scaled independently (they share connection pools)
- The services cannot migrate to different database technologies independently
- The ownership of data is ambiguous — both services have write access

**The rule:** if Service A needs data that Service B owns, Service A makes an API call to Service B. Service A never queries Service B's database directly.

Acceptable trade-offs:
- Data duplication across service boundaries (handled via event-driven sync)
- Eventual consistency (most business operations tolerate seconds of lag)

Not acceptable:
- Direct cross-service database queries
- A shared database schema with separate connection strings (this is still a shared database)

## Communication Patterns

### Synchronous — Request/Response

Service A calls Service B and waits for the response before proceeding.

**Use when:**
- The result of the call is required to continue (e.g., validate a user's permissions before creating a visit)
- The operation has a natural request/response shape
- Latency of the combined call is acceptable to the end user

**Technology:** REST over HTTPS is the default. GraphQL is appropriate when consumers need flexible querying and the schema complexity is justified.

**Risks and mitigations:**

| Risk | Mitigation |
|------|-----------|
| Service B is down → Service A fails | Circuit breaker — fail fast, return fallback response |
| Service B is slow → Service A is slow | Timeout — set aggressive timeouts; do not wait indefinitely |
| Transient failure → permanent failure | Retry with exponential backoff and jitter (max 3 attempts) |
| Cascading failure → full system down | Bulkhead — isolate failure; one degraded dependency must not take down the caller |

Every synchronous call must have: a timeout, a retry policy, and a circuit breaker. A service that calls downstream services without these is a reliability liability for the whole system.

### Asynchronous — Event-Driven

Service A publishes an event to a message queue. Service B (and any other interested service) consumes the event independently and at its own pace. Service A does not wait and does not know which services responded.

**Use when:**
- Service A does not need the result to continue (e.g., after a visit is confirmed, send notifications, record analytics, create calendar entries)
- Multiple services need to react to the same business event
- The operation is naturally fire-and-forget
- You want to fully decouple services from one another

**Example: "Visit Confirmed" event flow**

```
Visit Service
  → publishes: { event: "visit.confirmed", visit_id: "vis-123", ... }

Notification Service  → sends confirmation email to guest
Analytics Service     → records visit confirmation metric
Calendar Service      → creates calendar block for host
Billing Service       → generates invoice

(All independently. Visit Service knows nothing about them.)
```

**Event design rules:**
- Events describe what happened, not what to do: `visit.confirmed` not `send_confirmation_email`
- Events are immutable once published — never edit a published event
- Events carry enough data for consumers to act without calling back (include entity IDs and key fields)
- Consumer services are idempotent — processing the same event twice must not cause duplicate side effects

**Tooling options:** RabbitMQ, AWS SQS/SNS, Google Pub/Sub, Apache Kafka (high-throughput event streaming). The right tool depends on volume, ordering guarantees, and replay requirements.

## API Contracts

A service's API is a promise to its consumers. Breaking that promise without notice is a production incident for the consuming team.

**Every service API must have:**
- An OpenAPI (Swagger) specification, committed to the repository and kept current
- Versioned URL paths: `/api/v1/visits`, `/api/v2/visits`
- Explicit deprecation notices before any breaking change
- Contract tests that verify both sides of the interface

**Versioning rules:**
- v1 is forever — never delete a versioned endpoint; only deprecate it
- Breaking changes (removing a field, changing a field type, changing behaviour) always require a new version
- Non-breaking additions (new optional fields, new endpoints) can be added to the existing version
- Deprecation notice must be given a minimum of 6 months before an endpoint is removed
- Follow `skills/deprecation-and-migration/SKILL.md` (`deprecation-and-migration`) for the full deprecation workflow

### Contract Testing

Contract tests verify that the provider (the service that exposes the API) and the consumer (the service that calls the API) agree on the interface — not just at a point in time, but continuously, in CI.

**Provider test:** the provider's CI pipeline runs the contract tests for every consumer. A failing consumer contract test is a breaking change that blocks deployment.

**Consumer test:** each consumer verifies that it can correctly parse the responses it will receive from the provider.

Both must pass before either service can be deployed.

```
Contract test flow:

Consumer team writes:    "I expect GET /visits/{id} to return { id, status, guest_name }"
Provider CI verifies:    GET /visits/{id} actually returns that shape
Provider builds green:   contract is satisfied → consumer can deploy safely
Provider changes shape:  contract test fails → breaking change caught before production
```

**Tooling options:** Pact (language-agnostic, widely used), Spring Cloud Contract (JVM ecosystems).

## Shared Code Strategy

Sharing code between services must be intentional and controlled. Uncontrolled sharing creates hidden coupling — services that appear independent but break together when shared code changes.

**Share via a versioned internal package:**

| What to share | Why |
|--------------|-----|
| TypeScript interfaces / types for shared entities | Prevents contract drift without runtime coupling |
| API client SDKs (auto-generated from OpenAPI specs) | Consumers don't hand-write HTTP calls |
| Logging utilities | Consistent structured log format across all services |
| Authentication middleware | Consistent token validation logic |
| Common validation schemas | Shared business rules expressed once |

**Never share:**

| What not to share | Why |
|------------------|-----|
| Business logic | Business logic belongs in exactly one service; sharing it means two services share a domain |
| Database schemas or ORM models | Each service owns its data exclusively |
| UI components across backend services | Not applicable to backend services; for frontend microfrontends this is a separate architectural decision |

**Shared library versioning:**
- Semantic versioning: `major.minor.patch`
- Breaking changes (changing a type, removing an export) → bump major version
- Each service pins to a specific version and upgrades on its own schedule
- No service is forced to upgrade; incompatible changes are opt-in

## Repository Structure

### Monorepo — One Repository, Many Services

```
company/platform/
├── services/
│   ├── identity-service/
│   │   ├── src/
│   │   ├── package.json
│   │   └── Dockerfile
│   ├── visit-service/
│   └── notification-service/
├── packages/
│   ├── shared-types/       ← versioned shared interfaces
│   └── api-clients/        ← generated API client SDKs
└── infrastructure/
    ├── terraform/
    └── k8s/
```

**Pros:** shared code is trivially accessible; atomic commits across services; one CI/CD system; easier code search and refactoring.

**Cons:** repository grows large over time; CI/CD pipelines must be configured to build only affected services; tooling complexity increases with scale.

**Use when:** teams share significant code, cross-service changes are frequent, or the organisation is not yet large enough for polyrepo tooling overhead to be worth it.

**Tooling:** Nx or Turborepo for affected-service detection and build caching; without this, a monorepo CI pipeline that rebuilds everything on every commit becomes untenable as the repo grows.

### Polyrepo — One Repository Per Service

```
github.com/company/identity-service/
github.com/company/visit-service/
github.com/company/notification-service/
github.com/company/shared-types/          ← published as internal npm package
github.com/company/api-clients/           ← generated SDK, published internally
```

**Pros:** true independence — services can use different languages, frameworks, and dependency versions; repositories are small and focused; teams have clear ownership.

**Cons:** cross-service changes require coordinated PRs across multiple repositories; shared code must be published and versioned as a proper package; discoverability requires good documentation.

**Use when:** teams are large enough that shared infrastructure overhead is justified, services are truly independent, or different services need different technology stacks.

## Service Discovery

How services find each other varies by environment.

| Environment | Method |
|------------|--------|
| **Local development** | Hardcoded `localhost` ports; Docker Compose service names (`http://visit-service:3000`) |
| **Production (simple)** | Environment variables injected at deploy time: `VISIT_SERVICE_URL=https://visit.internal` |
| **Production (Kubernetes)** | Kubernetes service DNS: `http://visit-service.default.svc.cluster.local` |
| **Production (service mesh)** | Consul, Istio, or AWS App Mesh handle routing, mTLS, and load balancing |

Start with environment variables. Graduate to a service mesh when you have enough services that manual URL management becomes error-prone — not before.

## Inter-Service Authentication

Services must authenticate each other. An internal network is not a trust boundary. A compromised service inside the network can call any other service unless authentication is enforced.

| Method | Security level | Complexity | Use when |
|--------|---------------|-----------|---------|
| **Service API keys** | Medium | Low | Simple internal services, getting started |
| **Service-to-service JWTs** | Medium-high | Medium | Stateless auth with claim-based identity |
| **mTLS (mutual TLS)** | High | High | Regulated environments, zero-trust requirements |

Every service must verify the identity of callers. Requests without valid credentials are rejected with `401`. See `skills/security-and-hardening/SKILL.md` (`security-and-hardening`) for implementation detail.

## Observability in a Multi-Service System

Observability in a multi-service system is categorically harder than in a monolith. A single user request may traverse five services, and a failure in any one of them produces a symptom in a different one. Without distributed tracing, diagnosing cross-service latency and errors requires manually correlating log entries across five separate systems.

**Correlation ID propagation — non-negotiable:**

```
User request arrives at API Gateway
  → generate correlation_id: "req-abc123"
  → pass as HTTP header: X-Correlation-ID: req-abc123

Visit Service receives request
  → reads X-Correlation-ID from header
  → includes in all log entries: { "correlation_id": "req-abc123" }
  → passes header downstream when calling Notification Service

Notification Service receives request
  → reads X-Correlation-ID
  → includes in all log entries: { "correlation_id": "req-abc123" }
```

Every service in the call chain must: read the correlation ID from the incoming request, include it in all log entries, and pass it in all outgoing requests. A service that drops the correlation ID breaks the trace for all downstream services.

**Distributed tracing:** when log correlation is insufficient (complex call graphs, performance investigations across many services), add distributed tracing (OpenTelemetry + Jaeger, Zipkin, or Datadog APM). Each service emits spans; the tracing backend assembles them into a complete request trace.

See `skills/observability/SKILL.md` (`observability`) for the full instrumentation standard. Every service in a multi-service system must meet that standard individually.

**System-wide health dashboard:** each service exposes `/health`. An API gateway or dedicated health aggregator polls all `/health` endpoints and presents a system-wide health map. An incident in one service is immediately visible in context of the whole system.

## Deployment Coordination

True service independence means services deploy on their own schedule without coordinating with other services. When coordination is required, it is a signal that the service boundary or the API contract needs work.

There are two legitimate cases where deployment sequence matters:

**Database migrations that change shared contracts:**

```
Step 1: Deploy migration — new column added (nullable, old code ignores it)
Step 2: Deploy new service version — writes to new column
Step 3: (Later) Deploy cleanup — remove old column after all consumers updated
```

Migrations must always be backward-compatible with the currently-deployed service version. Never deploy a migration that breaks the running code.

**API version transitions:**

```
Step 1: Deploy v2 API alongside v1 (both live simultaneously)
Step 2: Update each consumer to use v2 (independent PRs, own schedule)
Step 3: Mark v1 deprecated (add deprecation headers, notify owners)
Step 4: Remove v1 after deprecation period (minimum 6 months)
```

This sequence means no consumer is ever broken by a provider deployment. The provider deploys first; consumers follow at their own pace.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "Microservices will let us scale faster" | Microservices let you scale services independently — but only after you have correctly identified the domain boundaries and built the operational infrastructure. Before that point, they slow you down. |
| "Each team should own a service" | Teams should be organised around business domains, and services should reflect those domains. Reorganising services to match org structure produces services with wrong boundaries. Conway's Law is a warning, not a design principle. |
| "We'll share the database and add APIs later" | There is no "later." A shared database produces coupling that is nearly impossible to untangle once services are in production and data is mixed. Design the boundary correctly from the start. |
| "Services are small so they're easy to reason about" | Nano-services with one or two responsibilities produce a system with hundreds of services, hundreds of deployment pipelines, and hundreds of times the operational overhead. Smaller is not always better. |
| "We can fix the boundaries later" | Service boundary changes after data has accumulated require coordinated migrations, dual-write periods, and data backfills. The cost is very high. Fix boundaries before there is production data in the wrong service. |

## Red Flags

- Deploying Service A requires deploying Service B — the services are not independent
- Two services share a database schema or query each other's tables directly
- A "service" that is smaller than 3 domain concepts (likely a nano-service)
- A "service" that contains most of the system's business logic (not a service — a monolith with an API)
- No OpenAPI specification exists for a service's external API
- Contract tests do not exist — breaking changes are only discovered in production
- No correlation ID propagation — request tracing requires manual log correlation
- A shared library that contains business logic (business logic must live in exactly one service)
- Synchronous call chains longer than 3 hops (Service A → B → C → D for a single user request)
- No circuit breakers on synchronous downstream calls

## Verification

Before deploying a new service or extracting one from a monolith:

```
□ Service passes all four independence tests (deploy, develop, scale, fail)
□ No shared database — service has its own data store
□ OpenAPI specification written and committed to the repository
□ API versioning strategy defined and applied
□ Communication pattern chosen (sync vs async) for each service interaction
□ Contract tests written and running in CI for all consumers
□ Inter-service authentication implemented and tested
□ Correlation ID propagated through all inbound and outbound requests
□ Observability standard met: structured logs, four golden signals, /health endpoint
□ Circuit breakers configured on all synchronous downstream calls
□ Deployment order documented for any migration that requires sequencing
□ Rollback procedure documented and tested
```

---

## Should I Use Microservices?

Most teams adopt microservices too early. Work through this decision tree before committing.

```
START
  │
  ▼
Is the monolith actually causing problems?
  │
  ├── No → Keep the monolith.
  │         A problem-free monolith does not need to be split.
  │
  └── Yes → What kind of problem?
              │
              ├── Deployment speed / team autonomy
              │     │
              │     └── Can this be solved with better module boundaries
              │         inside the monolith?
              │           │
              │           ├── Yes → Enforce module boundaries first.
              │           │         Modular monolith before microservices.
              │           │
              │           └── No → How many engineers?
              │                     │
              │                     ├── Fewer than 8 → Stay monolith.
              │                     │   Microservice overhead will hurt more
              │                     │   than the problem you're solving.
              │                     │
              │                     └── 8 or more → Proceed. Identify
              │                                     bounded contexts first.
              │
              ├── Scaling — one part of the system needs more resources
              │     │
              │     └── Can the overloaded component be scaled
              │         vertically or with read replicas?
              │           │
              │           ├── Yes → Do that first.
              │           │         Cheaper, lower risk, faster.
              │           │
              │           └── No → Extract that specific component only.
              │                    Do not decompose the rest of the monolith.
              │
              └── Different technology requirement
                    │
                    └── Does one part of the system need a technology
                        the rest cannot accommodate? (e.g., ML model
                        serving in Python, rest of system in Node)
                          │
                          ├── Yes → Extract that specific service.
                          │
                          └── No → "We want to use new tech" is not
                                    a valid architectural driver.
                                    Keep the monolith.
```

**The honest default:** if you are uncertain, keep the monolith. Extract services when the specific pain of the monolith is clearly identified and clearly exceeds the operational cost of the extracted service. Do not extract as a precautionary measure.

---

## See Also

- For ADRs on service boundary and technology decisions, see `skills/architecture-decisions/SKILL.md` (`architecture-decisions`) — service boundaries are irreversible architectural decisions that require documented rationale
- For API design standards for service interfaces, see `skills/api-and-interface-design/SKILL.md` (`api-and-interface-design`)
- For distributed tracing and per-service instrumentation, see `skills/observability/SKILL.md` (`observability`)
- For independent deployment pipelines per service, see `skills/ci-cd-pipeline/SKILL.md` (`ci-cd-pipeline`)
- For data ownership boundaries that simplify compliance, see `skills/compliance-and-regulatory/SKILL.md` (`compliance-and-regulatory`)
- For API versioning and contract deprecation procedures, see `skills/deprecation-and-migration/SKILL.md` (`deprecation-and-migration`)
- For inter-service authentication patterns, see `skills/security-and-hardening/SKILL.md` (`security-and-hardening`)
- For challenging service boundaries before committing to them, see `skills/doubt-driven-development/SKILL.md` (`doubt-driven-development`)
