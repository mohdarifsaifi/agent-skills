---
name: architecture-decisions
description: Documents significant architectural decisions as ADRs before implementation. Use when a decision affects system structure, selects technology, defines contracts, impacts security or compliance, is hard to reverse, or crosses team boundaries.
---

# Architecture Decisions

## Overview

Write an Architecture Decision Record before implementing any significant architectural change. The ADR is the institutional memory of your system — it captures context, options considered, the decision made, and its consequences. Code outlives the people who wrote it. An ADR ensures the reasoning survives team changes, reorgs, and memory loss.

**The core discipline:** Decision first, implementation second. An ADR written after the fact is archaeology, not architecture. Its entire value lies in forcing explicit reasoning before you're committed to a path.

## When to Use

Trigger an ADR for any decision that:

- Affects system structure or service boundaries
- Involves technology or framework selection
- Defines API contracts or data models
- Impacts security or compliance posture
- Would be difficult or expensive to reverse
- Affects more than one team or service

**When NOT to write an ADR:**
- Implementation details (variable names, internal functions)
- Styling or UI choices
- Minor refactors within a single component
- Decisions that are trivially reversible

**The decision threshold test:**

> "If this decision turns out to be wrong in 6 months, how painful is the reversal?"
>
> - Minor pain → no ADR needed
> - Significant pain → write an ADR

When in doubt, write the ADR. A short ADR that turns out to be unnecessary costs minutes. A missing ADR for a costly mistake costs months.

## The ADR Process

### Step 1: Pause Before Deciding

When you recognize a decision-worthy moment, stop before picking a direction. Identify:

```
DECISION CHECKPOINT:
- What decision am I about to make?
- Who else is affected by this decision?
- What options exist? (Have I listed at least three?)
- What are the forcing constraints? (performance, cost, compliance, deadline)
- Who needs to be in the room for this?
→ Write an ADR before proceeding.
```

This pause is the hardest part. The pressure to "just get started" is real. Resist it.

### Step 2: Number and File the ADR

Store ADRs in `docs/adr/` (or `decisions/adr/` if the project has a dedicated decisions directory). Use a zero-padded sequential number:

```
docs/adr/
  0001-use-postgresql-as-primary-datastore.md
  0002-adopt-event-sourcing-for-order-service.md
  0003-deprecate-xml-api-in-favor-of-json.md
```

The number is permanent. When an ADR is superseded, mark it deprecated and reference the new ADR — never renumber.

### Step 3: Write the ADR

Use the template below. Every section is required. If you cannot fill out Options Considered with at least two real alternatives, you haven't thought hard enough yet.

### Step 4: Review Before Locking

An ADR in **Proposed** status is a draft. Route it for review by:

- Engineers affected by the decision
- Team or tech lead who owns the affected system
- Security or compliance if the decision touches either domain

Only move to **Accepted** once reviewers have signed off. Don't implement until the ADR is Accepted.

### Step 5: Link the ADR to the Work

When implementing a decision, link from the PR description to the relevant ADR:

```
Implements ADR-0014: see docs/adr/0014-switch-to-grpc-internal-transport.md
```

Add a brief comment in code at major decision points:

```python
# Transport layer uses gRPC — see ADR-0014
```

This creates a navigable trail from code to decision and back.

### Step 6: Supersede, Don't Delete

When a decision changes, write a new ADR explaining why the original decision no longer holds. Update the old ADR's status:

```
**Status:** Superseded by ADR-0031
```

Never delete or silently edit an Accepted ADR. The history of *why you changed your mind* is as valuable as the original decision.

## ADR Template

```markdown
# ADR-[NNN]: [Short imperative title — "Use X for Y", "Adopt X", "Replace X with Y"]

**Date:** YYYY-MM-DD
**Status:** [Proposed | Accepted | Deprecated | Superseded by ADR-NNN]
**Deciders:** [Names or roles — who was in the room]
**Tags:** [security | data-model | api-contract | infrastructure | compliance | ...]

## Context

[Describe the situation that necessitates this decision. Include system state,
constraints, and forces at play. Assume the reader has no prior context.
Be specific: "our PostgreSQL instance is hitting 80% CPU on peak days" is better
than "we have performance problems." Future readers won't have your mental model.]

## Decision Drivers

- [Constraint, principle, or goal that shaped the decision]
- [e.g., "Must support GDPR data residency requirements"]
- [e.g., "Team has no Go expertise — new language adds 6 months of ramp"]
- [e.g., "System must handle 50k req/s at p99 < 200ms"]

## Options Considered

### Option A: [Name]

[One paragraph description.]

**Pros:**
- [...]

**Cons:**
- [...]

### Option B: [Name]

[One paragraph description.]

**Pros:**
- [...]

**Cons:**
- [...]

### Option C: Do nothing / defer

[Why deferring is or isn't a viable option at this point.]

## Decision

**Chosen option: Option [X] — [Name]**

[One to three paragraphs explaining why this option was selected over the
alternatives. Reference the decision drivers. Make the reasoning explicit —
don't just say "it was the best fit." A reader six months from now should
understand exactly what made the other options lose.]

## Consequences

**Positive:**
- [What improves as a result of this decision]

**Negative:**
- [What gets worse, or what new constraints this introduces]
- [Technical debt accepted, capabilities sacrificed, or work deferred]

**Neutral / watch items:**
- [Things to monitor — metrics to track, risks to revisit]

## Compliance and Security Impact

[Any regulatory, security, or operational risks introduced by this decision.
If none: state "No compliance or security impact identified." Do not leave blank.]

## Review Trigger

[When should this decision be revisited? Examples:
- "After six months of production load data"
- "When the team exceeds 20 engineers"
- "Before extending this service to the EU region"
- "If latency p99 exceeds 500ms in production"]
```

## ADR Status Lifecycle

```
Proposed ──→ Accepted ──→ Deprecated
                │
                └──→ Superseded by ADR-NNN
```

| Status | Meaning |
|--------|---------|
| **Proposed** | Draft under review — do not implement |
| **Accepted** | Approved — implementation may proceed |
| **Deprecated** | Decision no longer applies but wasn't replaced |
| **Superseded** | Replaced by a newer ADR — link to the replacement |

## ADR Index

Maintain a `docs/adr/README.md` that lists all ADRs, their status, and a one-line summary. This is the first file a new team member reads.

```markdown
# Architecture Decision Records

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0001](0001-use-postgresql-as-primary-datastore.md) | Use PostgreSQL as primary datastore | Accepted | 2024-01-15 |
| [0002](0002-adopt-event-sourcing-for-order-service.md) | Adopt event sourcing for order service | Accepted | 2024-02-03 |
| [0003](0003-deprecate-xml-api.md) | Deprecate XML API in favor of JSON | Superseded by [0009](0009-...) | 2024-03-10 |
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We already know what we're doing, the ADR is just overhead" | If the decision is obvious, the ADR takes 20 minutes. If it isn't, you need the ADR. |
| "We'll document it after we've proven the approach works" | That's a post-mortem, not a decision record. The forcing function is writing it before committing. |
| "The decision might change, so why write it down?" | Changing decisions are exactly why you document them — so the next decision has the full history. |
| "This is a small decision, it doesn't need an ADR" | Apply the threshold test. If reversal is significant pain, write it. Small isn't the measure — reversibility is. |
| "Everyone on the team knows why we did this" | The team changes. The knowledge doesn't transfer automatically. |

## Red Flags

- Implementing a technology or framework choice before any written rationale exists
- An ADR with only one option considered (that's a justification, not a decision record)
- ADRs written retrospectively to explain decisions already in production
- No one can explain why a core architectural choice was made ("it was before my time")
- A decision that crossed team boundaries with no written agreement
- ADRs that are never superseded despite the system evolving significantly
- Treating ADR review as a formality — approving without reading the Consequences section

## Verification

Before marking an ADR Accepted:

- [ ] Context section gives enough background for someone who wasn't in the room
- [ ] At least two real alternatives are documented with honest pros and cons
- [ ] Decision rationale explicitly references the decision drivers
- [ ] Consequences section names both positive and negative outcomes
- [ ] Compliance and security impact is addressed (even if "none identified")
- [ ] A review trigger is defined
- [ ] Affected teams and the tech lead have signed off
- [ ] The ADR index (`docs/adr/README.md`) is updated
- [ ] The ADR is committed to version control before implementation begins

## See Also

- For decisions made during planning, see `skills/planning-and-task-breakdown/SKILL.md` (`planning-and-task-breakdown`) — architecture decisions belong in the ADR log, not buried in plan documents
- For decisions about API shape, see `skills/api-and-interface-design/SKILL.md` (`api-and-interface-design`)
- For decisions with security implications, see `skills/security-and-hardening/SKILL.md` (`security-and-hardening`)
- For long-term documentation strategy, see `skills/documentation-and-adrs/SKILL.md` (`documentation-and-adrs`)
- For adversarial review of ADRs before accepting, see `skills/doubt-driven-development/SKILL.md` (`doubt-driven-development`) — actively try to disprove the decision before committing to it
- For technology choices that must cite official documentation, see `skills/source-driven-development/SKILL.md` (`source-driven-development`) — every technology choice in an ADR must be grounded in sources, not memory or assumption
