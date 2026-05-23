---
name: ci-cd-pipeline
description: Builds automated, tested, repeatable deployment pipelines. Use when setting up a new project, before first production deployment, when manual deployments cause errors, or when deployment frequency or rollback speed needs to improve.
---

# CI/CD Pipeline

## Overview

Automate every step between a merged pull request and running production code. A pipeline removes humans from the riskiest deployment steps — not to exclude engineers, but because humans make different mistakes each time and pipelines make the same correct steps every time. A deployment that cannot be done by a pipeline is a deployment that cannot be done safely at scale.

**The core discipline:** Set up the pipeline before the first production deployment, not after the first production incident. The cost of configuring automation once is far less than the cumulative cost of manual deployment errors, inconsistent environments, and slow rollbacks.

CI and CD are separate disciplines that compose into a single workflow. Know which you are building.

## When to Use

Activate this skill when:

- Setting up any new project or service
- Preparing for a first production deployment
- Manual deployments are causing errors or inconsistency
- Deployment frequency needs to increase
- Rollback after a bad deployment takes more than 5 minutes
- "It works on my machine" is a recurring phrase

**When NOT to apply full rigor:**
- Throwaway prototypes that will never reach production
- One-person projects with a single environment and no SLA

**The pipeline readiness test:**

> "If the engineer who wrote the deployment script is unavailable, can someone else deploy safely without calling them?"
>
> - Yes → pipeline is working
> - No → the pipeline is missing, incomplete, or undocumented

A deployment process that lives in one person's head is a single point of failure.

## Core Concepts

### Continuous Integration (CI)

Every code push triggers an automated build and test run. The goal is fast feedback — an engineer knows within minutes whether their change broke something, not at the end of the day when context has switched.

CI enforces three things:
1. **Tests run automatically** — no manual "remember to run tests" step
2. **Broken builds are visible** — the team sees failures immediately
3. **Branch protection** — a branch with failing tests cannot be merged

CI does not deploy anything. It validates that the code is correct and the artifact is buildable.

### Continuous Delivery (CD — Delivery)

Every passing build produces an artifact that is ready to deploy to production at any time. Deployment is a business decision, not a technical one. When a product manager says "release that feature," the answer is never "we need two days to prepare a deployment."

Continuous Delivery means:
- Staging is always up to date with main
- The production deployment is a one-click or one-command operation
- A human approves the promotion from staging to production

### Continuous Deployment (CD — Deployment)

Every passing build is automatically deployed to production without human approval. No gate, no button, no on-call engineer woken up. The system trusts the test suite completely.

Continuous Deployment requires:
- Very high test confidence (near-comprehensive coverage, contract tests, smoke tests)
- Robust health checks and automatic rollback
- Feature flags to separate deployment from feature release

Most teams should start with Continuous Delivery and graduate to Continuous Deployment as test confidence grows. Skipping to Continuous Deployment with a weak test suite produces frequent production incidents.

## Environment Strategy

Every production system needs at least three environments. Each serves a specific purpose — do not conflate them.

### Development

Each engineer's local machine. No shared state. Never connected to production systems or real user data. Seeded with fake or anonymised data only. Engineers must be able to spin up the full application locally without needing production credentials.

If local development requires production credentials, fix the application architecture — not the developer workflow.

### Staging

A replica of production. Same infrastructure, same configuration, same data shape — but populated with anonymised data, never real PII. Staging is automatically deployed on every merge to the main branch. It is where business stakeholders perform user acceptance testing (UAT) and where the final pre-production smoke test runs.

Staging that doesn't mirror production is useless. A bug that doesn't reproduce on staging will appear in production.

### Production

Real users. Real data. Protected.

Deployments to production require either explicit approval (Continuous Delivery model) or must pass automated post-deploy health checks that trigger automatic rollback (Continuous Deployment model). All production changes are logged. Rollback must complete within 5 minutes.

## Pipeline Stages

The pipeline runs these seven stages in sequence. Every stage must pass before the next begins. Never skip a stage to deploy faster — that is how incidents happen.

### Stage 1 — Code Quality

Runs on every push, including PRs. Fast — should complete in under 2 minutes.

- **Linting:** enforce code style automatically; style debates belong in configuration, not reviews
- **Type checking:** catch type errors before tests run (TypeScript, mypy, etc.)
- **Dependency vulnerability scan:** flag known CVEs in third-party packages (`npm audit`, `pip-audit`, Snyk)
- **Secret scanning:** detect API keys, tokens, or credentials accidentally committed to the repository

A commit that introduces a known-critical vulnerability or a hard-coded secret does not proceed past Stage 1.

### Stage 2 — Testing

Runs on every push. Must complete before any artifact is built.

- **Unit tests:** fast, isolated, no external dependencies
- **Integration tests:** test interactions with the database, cache, and internal services
- **API contract tests:** verify that the API shape matches what consumers expect
- **Coverage gate:** enforce a minimum coverage threshold (80% is a reasonable starting floor); coverage must not decrease between commits

Two rules: tests must pass (zero failures), and coverage must not drop. A PR that adds code but drops coverage below the threshold is not ready to merge.

### Stage 3 — Build

Runs after tests pass on the main branch.

- Compile or bundle the application
- Build the Docker image (if containerised)
- Tag the image with the git commit SHA — never with `latest` in non-local environments
- Push the artifact to the container registry or artifact store

**Critical rule:** the artifact produced here is what gets deployed to staging and then to production. It is never rebuilt. Rebuilding from source for production means production runs different code than what was tested. Tag once, promote the same tag.

```
Artifact naming convention:
  my-service:a3f2c91      ← git commit SHA (immutable)
  my-service:main-latest  ← floating pointer to latest main build (convenience only)
```

### Stage 4 — Deploy to Staging

Runs automatically after a successful build on main. No human approval required.

- Pull the tagged artifact from Stage 3
- Deploy to the staging environment
- Run smoke tests against staging (see Stage 7 pattern)
- Notify the team of the deployment (Slack, email, or dashboard)

If the staging deployment or smoke tests fail, the pipeline stops. Production is unaffected. The team investigates before proceeding.

### Stage 5 — Production Approval Gate

Applies to Continuous Delivery only. Skip for Continuous Deployment.

A human reviews before production is touched. The approval interface must show:

```
Pending production deployment:
  Branch:   main
  Commit:   a3f2c91 — "Add user export endpoint" (Alice, 2h ago)
  Tests:    ✓ Passed (142 tests, 84% coverage)
  Staging:  ✓ Deployed and healthy (35 min ago)
  Changes:  +247 / -31 lines across 8 files

  [ Approve and Deploy ]  [ Reject ]
```

The approver must be able to see what changed and that it passed staging. "I'll just approve and watch it" is not an acceptable substitute for a functioning staging environment.

### Stage 6 — Deploy to Production

Deploys the same artifact that was deployed to staging — not a rebuild, not `latest`, but the exact tagged image from Stage 3.

- Use blue-green or rolling deployment for zero downtime (see Deployment Patterns)
- Run health checks immediately after deployment
- If health checks fail: trigger automatic rollback without waiting for a human

```
Deployment verification sequence:
  1. Deploy new version (blue-green: to idle environment)
  2. Wait for instances to report healthy
  3. Run health check endpoint: GET /health → 200 OK
  4. Run smoke tests
  5. If all pass: shift traffic
  6. If any fail: rollback immediately, alert on-call
```

### Stage 7 — Post-Deployment Verification

Runs in production immediately after traffic is shifted. Catches regressions that only appear under real load.

- **Smoke tests:** a small set of critical-path tests that verify the system works end-to-end (login, core workflow, key API endpoints)
- **Error rate monitoring:** compare error rate for 15 minutes post-deploy against the baseline; alert if it spikes above threshold (e.g., 2× baseline)
- **Latency monitoring:** verify p99 latency has not regressed beyond an acceptable threshold
- **Automatic rollback trigger:** if error rate exceeds threshold within the monitoring window, roll back automatically and page on-call

Smoke tests in production must be non-destructive. They read and verify state; they do not mutate production data.

## Branch Strategy

### Main Branch

- Always deployable — every commit on main must be in a state that could safely reach production
- Protected: PRs require at least one reviewer and all CI checks passing before merge
- Direct push forbidden; enforce via branch protection rules in the repository settings

### Feature Branches

- One branch per feature or fix
- Named descriptively: `feat/user-export`, `fix/login-timeout`, `chore/update-dependencies`
- Short-lived: merged within days, not weeks
- Long-lived feature branches accumulate merge conflicts and drift from main; they are a deployment risk

**If a feature is too large to merge in days:** use feature flags to merge incomplete code safely. Deploy the code to production behind a flag that is off. Enable the flag when the feature is complete.

### Release Branches (Enterprise Only)

For teams with scheduled release windows or regulated change management:

- Cut a release branch from main: `release/2024-Q2`
- Only bug fixes are cherry-picked onto the release branch — no new features
- Stabilisation period runs before the scheduled release
- Release branch is merged back to main after deployment

Most teams do not need release branches. They add process without adding safety if the main branch is kept deployable at all times.

## Deployment Patterns

### Blue-Green Deployment

Two identical production environments exist simultaneously. At any given time, one is live (serving traffic) and one is idle. To deploy: bring the idle environment up to date with the new version, run health checks, then atomically switch all traffic from the live environment to the newly updated one.

```
Before deploy:   Blue (live, v1.2) ← 100% traffic
                 Green (idle)

After switch:    Blue (idle, v1.2)
                 Green (live, v1.3) ← 100% traffic

Rollback:        Blue (live, v1.2) ← 100% traffic  (instant)
                 Green (idle, v1.3)
```

**Best for:** Zero-downtime requirement; instant rollback needed. Requires double the infrastructure.

### Rolling Deployment

Update instances one at a time (or in small batches). New version gradually replaces the old version across the fleet. At any point during the deployment, some instances run the new version and some run the old.

**Best for:** Large fleets where duplicating infrastructure is expensive. Rollback is slower than blue-green. Requires the new and old versions to handle requests interchangeably during the transition window.

### Canary Deployment

Deploy the new version to a small percentage of traffic first. Monitor for errors. If the error rate is acceptable, progressively increase the percentage. Roll back if errors exceed the threshold at any stage.

```
Stage 1:   1% of traffic → new version | Monitor for 5 min
Stage 2:   5% of traffic → new version | Monitor for 10 min
Stage 3:  25% of traffic → new version | Monitor for 15 min
Stage 4: 100% of traffic → new version
```

**Best for:** High-risk changes where early error detection is worth the complexity. Requires the monitoring infrastructure to attribute errors to specific versions.

## Rollback Strategy

Every deployment must have a documented, tested rollback procedure. "We'll figure it out if something goes wrong" is not a rollback procedure.

**Target:** rollback completes within 5 minutes of the decision to roll back.

| Rollback method | How | When to use |
|----------------|-----|------------|
| **Blue-green swap** | Shift traffic back to the idle environment | Blue-green deployments; instant |
| **Image redeploy** | Deploy the previous tagged image | Rolling deployments; takes 1–3 min |
| **Feature flag kill switch** | Disable the flag, revert behaviour without redeploying | Features behind flags |
| **Database migration rollback** | Apply the down migration | Schema changes — test this; not all ORMs support it |

**Rollback triggers — automatic:**
- Health check endpoint returns non-200 after deploy
- Error rate exceeds 2× baseline within 15-minute monitoring window
- Latency p99 exceeds defined threshold

**Rollback triggers — manual:**
- On-call engineer observes unexpected behaviour
- Business stakeholder reports critical user-facing issue
- Security incident related to the deployed change

Test rollbacks regularly. A rollback procedure that has never been run will fail when you need it most.

## Feature Flags

Feature flags decouple deployment from release. Code is deployed to production in a disabled state; the feature is enabled separately, without a redeployment.

**Why this matters for CI/CD:** It allows very frequent deployments (multiple per day) of incomplete features. Unfinished code ships behind a flag that is off. When the feature is complete, it is enabled — no deployment required.

```
// Example: flag-guarded feature
if (featureFlags.isEnabled('new-checkout-flow', userId)) {
  return renderNewCheckout();
}
return renderLegacyCheckout();
```

**Flag types:**
- **Release flags:** temporary; removed once the feature is fully enabled and stable
- **Ops flags:** permanent kill switches for expensive or risky operations (e.g., disable background job under load)
- **Experiment flags:** A/B test routing; governed by experimentation platform

**Flag hygiene:** Remove release flags within one sprint of full rollout. Accumulated stale flags become unmaintained conditional branches that confuse future engineers.

**Tooling options:** LaunchDarkly (full-featured, expensive), Unleash (open-source), Flagsmith (open-source), or a simple database table for basic use cases.

## Security in CI/CD

The pipeline is itself a security boundary. Treat it accordingly.

- **Secrets never in pipeline config files.** Use the platform's secret manager (GitHub Secrets, AWS Secrets Manager, HashiCorp Vault). A secret committed to a YAML file in the repository has been compromised, whether or not anyone read it.
- **Principle of least privilege for pipeline credentials.** The CI runner's service account should have exactly the permissions needed for its jobs — nothing more. A credential that can deploy to production should not also have the ability to delete production databases.
- **Docker images scanned for vulnerabilities** before they are pushed to the registry. Block images with critical CVEs from reaching staging or production.
- **SAST (Static Application Security Testing)** runs as part of Stage 1. Tools: Semgrep, CodeQL, Bandit (Python), Brakeman (Ruby).
- **Dependency scanning** on every build. Do not wait for a weekly scheduled scan to discover a critical CVE in a package that shipped to production yesterday.
- **Signed commits and artifacts** for regulated environments. The pipeline verifies that artifacts were produced by the CI system, not substituted.

## Tooling

The right tool depends on your infrastructure. These are the common options — evaluate against your team's existing stack before introducing new tools.

| Category | Options |
|----------|---------|
| **CI/CD platforms** | GitHub Actions, GitLab CI/CD, Jenkins, CircleCI, Buildkite |
| **Container registry** | GitHub Container Registry (GHCR), AWS ECR, Google Artifact Registry, Docker Hub |
| **Secret management** | AWS Secrets Manager, HashiCorp Vault, GitHub Secrets (simple cases), Azure Key Vault |
| **Feature flags** | LaunchDarkly, Unleash, Flagsmith, Posthog (built-in flags) |
| **Monitoring / alerting** | Datadog, New Relic, Grafana + Prometheus, CloudWatch |
| **Vulnerability scanning** | Snyk, Trivy, Grype, GitHub Dependabot |

Do not introduce a new tool if an existing one in your stack already covers the need. Every tool is a dependency with a maintenance cost.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "We'll set up CI/CD once we're bigger" | Manual deployments become more dangerous as the system grows, not less. The time to automate is before the first incident, not after. |
| "Our tests aren't good enough for CI to be useful" | That is the reason to write better tests, not a reason to skip CI. CI makes the gaps in test coverage visible. |
| "Staging is too expensive to maintain" | A single staging incident in production — downtime, data loss, botched migration — costs more than months of staging infrastructure. |
| "We build a fresh artifact for production to be safe" | Building from source twice means testing one artifact and running another. Use the same tagged artifact from build to production. |
| "The deployment script is in my head" | That is a single point of failure. Document it in the pipeline. When you are unavailable, the team must be able to deploy. |
| "Rollback is simple, we'll just redeploy" | "Simple" rollbacks that have never been tested fail at 2am during an incident. Test the rollback before you need it. |

## Red Flags

- First production deployment is manual — no pipeline exists
- Staging environment is not an accurate mirror of production
- Different build artifacts are used for staging and production (rebuilt from source)
- Secrets are stored in CI environment variables in plain text (not a secret manager)
- No automated rollback — rollback requires an engineer to manually redeploy
- Feature branches longer than one week
- Pipeline stages skipped "just this once" under release pressure
- No health check endpoint exists; post-deploy verification is visual inspection
- Tests are disabled in the pipeline to make deployment faster
- A single engineer owns the deployment process and it is not documented

## Verification

Before treating a pipeline as production-ready:

```
□ Pipeline runs on every push to every branch (CI)
□ Main branch is protected: PR + passing tests required before merge
□ Direct push to main is disabled
□ Staging deploys automatically on every merge to main
□ Build artifact is tagged with git commit SHA
□ Same artifact is deployed to staging and then to production (no rebuild)
□ Health check endpoint exists and is called after every deployment
□ Automatic rollback triggers if health check fails
□ Rollback procedure is documented, tested, and completes within 5 minutes
□ No secrets in pipeline config files or environment variable plaintext
□ Dependency and vulnerability scan runs on every build
□ Secret scanning runs on every commit
□ Post-deploy monitoring window defined (15 min minimum) with alert thresholds
□ Smoke tests exist and run in both staging and production after deploy
```

---

## Pipeline Setup Checklist

Step-by-step for configuring CI/CD on a new project. Complete in order — each step depends on the previous.

```
PHASE 1 — Foundation (day 1)

□ 1. Choose CI/CD platform and create pipeline configuration file
      (.github/workflows/, .gitlab-ci.yml, Jenkinsfile)
□ 2. Configure Stage 1: linting, type checking, secret scanning
□ 3. Configure Stage 2: run existing tests; add coverage reporting
□ 4. Set minimum coverage threshold; fail build if threshold not met
□ 5. Enable branch protection on main: require PR + passing CI

PHASE 2 — Build and Artifacts (day 1–2)

□ 6. Configure Stage 3: build application or Docker image
□ 7. Tag artifact with git commit SHA
□ 8. Push artifact to container registry or artifact store
□ 9. Verify the same artifact will be used for all environments

PHASE 3 — Staging (day 2–3)

□ 10. Provision staging environment that mirrors production configuration
□ 11. Seed staging with anonymised data (never production PII)
□ 12. Configure automatic deploy to staging on merge to main
□ 13. Write smoke tests for the critical user path
□ 14. Run smoke tests automatically after staging deployment
□ 15. Configure deploy notifications (Slack, email, or dashboard)

PHASE 4 — Production Pipeline (day 3–5)

□ 16. Configure production deployment job using the staging-tested artifact
□ 17. Add manual approval gate (Continuous Delivery) OR configure
      automatic promotion criteria (Continuous Deployment)
□ 18. Implement blue-green or rolling deployment strategy
□ 19. Add health check call immediately after production deployment
□ 20. Configure automatic rollback on health check failure
□ 21. Configure post-deploy monitoring window with alert thresholds

PHASE 5 — Security and Secrets (parallel with phases 1–4)

□ 22. Move all secrets from environment variable plaintext to secret manager
□ 23. Add dependency vulnerability scanning to Stage 1
□ 24. Add SAST scan to Stage 1
□ 25. Add Docker image vulnerability scan to Stage 3
□ 26. Verify pipeline service account has least-privilege permissions

PHASE 6 — Validation (before first production use)

□ 27. Run a full pipeline end-to-end from a test commit
□ 28. Verify staging deployment occurred automatically
□ 29. Execute a manual rollback in staging and verify it completes in < 5 min
□ 30. Document the deployment and rollback procedures in the project README
□ 31. Confirm a second engineer can deploy without assistance from the author
```

---

## See Also

- For tests that make CI meaningful, see `skills/test-driven-development/SKILL.md` (`test-driven-development`) — CI is only as valuable as the tests it runs
- For security scans in the pipeline, see `skills/security-and-hardening/SKILL.md` (`security-and-hardening`)
- For compliance requirements that gate deployments, see `skills/compliance-and-regulatory/SKILL.md` (`compliance-and-regulatory`)
- For the final launch checklist that uses the pipeline, see `skills/shipping-and-launch/SKILL.md` (`shipping-and-launch`)
- For architectural decisions about deployment strategy, see `skills/architecture-decisions/SKILL.md` (`architecture-decisions`)
