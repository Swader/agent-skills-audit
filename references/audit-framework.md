# Audit Framework

Use this file as the operating checklist and output schema for the `audit-code` skill.

## Severity Rubric

- Critical: Immediate compromise, major data loss, financial loss, legal exposure, or service-wide outage likely.
- High: Exploitable or user-impacting defect with significant business risk but not immediate systemic collapse.
- Medium: Material weakness that can compound under scale/load or specific conditions.
- Low: Limited impact, hygiene issues, or improvements with small downside if deferred.

## Finding Schema

Use this structure for every finding:

- ID: Stable identifier, for example `SEC-001`, `PERF-003`.
- Role: `Security`, `Performance`, `UX`, `DX`, or `Edge`.
- Title: One-line defect statement.
- Severity: `Critical | High | Medium | Low`.
- Confidence: `High | Medium | Low`.
- Impact: Business/user/system impact in plain language.
- Evidence: File path + line reference + behavior summary.
- Trigger Conditions: Inputs, load profiles, user flows, or race conditions required.
- Reproduction: Minimal steps (or reason not reproducible in current context).
- Proposed Fix: Specific code or architecture change.
- Verification: Targeted test/check to validate the fix.
- Dependencies: Cross-team or sequencing constraints, if any.

## Role Checklists

### Security Expert

Check for:
- Authn/authz bypasses, privilege escalation, and missing tenant isolation.
- Injection vectors (SQL/command/template), unsafe deserialization, and tainted sinks.
- Secrets handling, key management, token lifetime/revocation, and session fixation.
- Idempotency/replay gaps, webhook signing/verification errors, race-prone state transitions.
- DDoS abuse surfaces: unbounded endpoints, expensive queries, amplification paths, missing rate limits.

### Performance Expert

Check for:
- N+1 queries, full scans, missing indexes, lock contention, and transaction scope bloat.
- Hot-path CPU/memory pressure, heavy sync work in request loops, and avoidable serialization cost.
- Inefficient build/runtime workflows: tasks that should move to async queues, batch jobs, or cron.
- Frontend payload bloat, hydration/render hotspots, and cache invalidation failures.
- Throughput/latency tail behavior under contention and degraded dependency modes.

### UX Expert

Check for:
- User journey friction: unnecessary steps, dead-ends, poor defaults, weak state feedback.
- Error/empty/loading states and perceived performance.
- Accessibility basics: keyboard flow, labels, focus handling, contrast, ARIA correctness.
- Human and bot operator flows where relevant (APIs, machine-consumable outputs, predictable contracts).
- Trust risks from coercive or manipulative patterns; flag compliance/reputation exposure.

### DX Expert

Check for:
- API clarity: stable contracts, explicit errors, pagination/filter semantics, and versioning hygiene.
- Code readability/extensibility: module boundaries, coupling, dead abstractions, and naming quality.
- Test strategy gaps: missing integration/contract/load tests for critical paths.
- Onboarding quality: concise docs, runbooks, architecture notes, and executable examples.
- LLM/operator friendliness: discoverable conventions and deterministic workflows.

### Edge Case Master

Check for:
- Rare-state transitions and multi-step flow interactions that break invariants.
- Time boundaries, timezone drift, retries, duplicate events, and out-of-order processing.
- Cross-system races (jobs, webhooks, external providers, same-host side effects).
- Non-obvious abuse chains combining medium findings into critical outcomes.
- Spec-vs-implementation mismatches hidden in user stories rather than obvious code smells.

## Two-Pass Execution Rule

1. Complete pass 1 for Security, Performance, UX, DX, then Edge.
2. Run tie-breaker review to reconcile conflicts.
3. Re-run Security/Performance/UX/DX using edge findings as new attack/load/flow assumptions.
4. Finish with Edge final pass to validate residual risk after proposed fixes.
5. Produce one merged final report from the tie-breaker lead.

## Final Report Template

Use this exact section order:

1. Findings (sorted by severity, blast radius, exploitability)
2. Open Questions / Assumptions
3. Remediation Plan (Now / Next / Later)
4. Verification Plan
5. Executive Summary
